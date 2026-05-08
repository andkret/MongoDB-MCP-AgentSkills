# MongoDB Live Solar Device State Tracker
## Step-by-Step Project Guide

A live operational data store powered by Atlas Stream Processing, designed and built with Claude Code + MongoDB Agent Skills.

---

## Phase 1: Atlas Setup

### 1.1 Create Atlas Account & Cluster
- Go to [mongodb.com/atlas](https://www.mongodb.com/atlas) and sign up for free
- Create a new project (e.g. `solar-tracker`)
- Deploy a free **M0 cluster** (any cloud provider/region is fine)
- Set a database username and password — save these
- Add your IP address to the IP Access List (or `0.0.0.0/0` for dev)

### 1.2 Get Your Connection String
- Atlas UI → Connect → Drivers
- Copy your connection string:
  ```
  mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/
  ```

### 1.3 Create the Database and Collections
Atlas UI → Browse Collections → Create Database:
- Database name: `solar-data`
- Create two collections:

**Collection 1: `raw-data`** (Time-Series)
- timeField: `timestamp`
- metaField: `device_id`
- granularity: `seconds`
- **expireAfterSeconds: `300`** (5 minutes)
- Archives every raw reading, but auto-deletes after 5 minutes to keep the collection bounded

> **Video moment:** "I'm setting a 5-minute retention. Watch the document count go up as data flows in, then plateau as old documents get auto-deleted. This is the Time-Series collection's TTL feature working — no cron jobs, no cleanup scripts, MongoDB handles it natively."

**Collection 2: `device_state`** (Regular collection)
- Uncheck Time-Series
- Live operational state — one doc per device
- ⚠️ Time-Series collections do NOT support upserts

> **Video moment:** "Two collections, two purposes. MongoDB lets you do both natively."

### 1.4 Create the Stream Processing Workspace
- Atlas UI → left sidebar → **Stream Processing**
- Click **Create Workspace**
- Choose **SP2** ($0.06/hr) — cheapest, fine for dev
- Workspace creation is free, you only pay while processors run
- Name: `market-tracker`, AWS, your nearest region
- ⚠️ Terminate processors when done recording to stop billing

### 1.5 Add Atlas Cluster to Connection Registry
- Inside the workspace → **Connection Registry**
- Add your Atlas cluster as a connection — name it something memorable (e.g. `Atlas DB`)
- Verify `sample_stream_solar` is listed as a built-in source
- ⚠️ There is NO stock price sample stream — only solar

### 1.6 Create a Straight-Through Pipeline to `raw-data`
Before designing any schema, get raw data flowing into MongoDB so you can actually look at it.

In the Stream Processing workspace → **Create Stream Processor** → paste:

```json
{
  "name": "raw-ingest",
  "pipeline": [
    {
      "$source": {
        "connectionName": "sample_stream_solar"
      }
    },
    {
      "$merge": {
        "into": {
          "connectionName": "Atlas DB",
          "db": "solar-data",
          "coll": "raw-data"
        }
      }
    }
  ]
}
```

Replace `Atlas DB` with your Atlas connection name. Start the processor.

### 1.7 Explore the Data in Atlas Data Explorer
- Atlas → Browse Collections → `solar-data` → `raw-data`
- You should see new documents arriving at ~2 events/second
- Each document looks like:

```json
{
  "_id": ObjectId("..."),
  "device_id": "device_7",
  "event_type": 0,
  "group_id": 8,
  "max_watts": 450,
  "obs": {
    "watts": 66,
    "temp": 8
  },
  "timestamp": "2026-05-08T09:13:51.278+00:00"
}
```

Now you understand the actual field shape — perfect input for the schema design step coming up.

---

## Phase 2: Local Dev Setup

### 2.1 Prerequisites
- **[Node.js v22 LTS](https://nodejs.org/)** — `.pkg` installer
  - ⚠️ Node 18 will fail. Must be v20+
- **[VS Code](https://code.visualstudio.com/)**
- **[Claude Code](https://claude.ai/code)** — installed and authenticated
- Create a project folder and open it in VS Code

### 2.2 Install the MongoDB Plugin in Claude Code
Inside Claude Code:
- Open **Manage Plugins**
- Search for **MongoDB**
- Install → choose **Global** (available across all projects)

This installs the **Agent Skills** (the SKILL.md files).

To verify, type in Claude Code:
```
/mongodb-schema-design
```
Claude responds with a summary of what the skill knows — that's your "show don't tell" video moment.

### 2.3 Connect Claude Code to Atlas (MCP Server)
This is **separate from the plugin** — it gives Claude Code live access to your cluster.

In your terminal:
```bash
claude mcp add mongodb-mcp-server -- npx -y mongodb-mcp-server@latest --connectionString "your-connection-string-here"
```

Verify:
```bash
claude mcp list
```
You should see `mongodb-mcp-server: ... ✓ Connected`.

⚠️ **Security note:** The connection string with password is stored in plain text in the MCP config and visible via `claude mcp list`. Rotate the password before recording the video, then again after publishing.

Restart VS Code completely. Test in Claude Code:
> *"List the collections in the solar-data database"*

When asked permission, choose **"Yes, allow for this project (just you)"**.

> **Plugin vs MCP — what does what:**
> - Plugin = Agent Skills (MongoDB expertise baked into Claude)
> - MCP = Live connection to Atlas (so Claude can actually query)
> - You need both

---

## Phase 3: Schema Design with Agent Skills

### 3.1 Invoke the Schema Design Skill
In Claude Code:
```
/mongodb-schema-design
```

Then prompt:
> *"I want to maintain a live state document in the solar-data database — one document per device_id that gets upserted with the latest reading from a solar device stream. Look at the raw-data collection in solar-data to understand the actual field shape coming in. Then design the device_state document schema with these requirements:*
> 
> *- One document per device_id, upserted on every event*
> *- Add an efficiency_pct field calculated from watts / max_watts*
> *- Add an update_count field that increments on every upsert*
> *- Track when each device was last seen*
> 
> *Tell me which indexes I need and why."*

### 3.2 Expected Schema Output
The skill will use the MCP server to inspect your real `raw-data` collection, then produce something like:
```js
{
  _id:            "device_0",                        // = device_id
  group_id:       8,
  max_watts:      450,
  event_type:     0,
  obs: {
    watts:        67,
    temp:         5
  },
  efficiency_pct: 14.9,                              // obs.watts / max_watts * 100
  update_count:   142,                               // increments each upsert
  last_seen:      ISODate("2026-05-08T09:13:16Z")
}
```

### 3.3 Expected Index Output
```js
// 1. Automatic — covers all upserts by _id
{ _id: 1 }

// 2. "All devices in group N"
{ group_id: 1 }

// 3. "Devices not seen in last 5 minutes" (stale detection)
{ group_id: 1, last_seen: 1 }

// 4. "Top performing devices in a group"
{ group_id: 1, efficiency_pct: -1 }
```

The skill will recommend starting with just `{ group_id: 1 }` — over-indexing hurts write throughput.

### 3.4 Create the Index
> *"Create the group_id index on the device_state collection in solar-data."*

Claude executes this directly via MCP. Collection doesn't need to have documents yet — the index is just metadata.

---

## Phase 4: Build the Stream Processing Pipeline

### 4.1 Generate the Pipeline JSON
In Claude Code:
```
/atlas-stream-processing
```

Then:
> *"Write the Atlas Stream Processing pipeline to read from sample_stream_solar and upsert into the device_state collection in solar-data. Use the schema we designed — one document per device_id using _id as the key, calculate efficiency_pct as obs.watts / max_watts * 100, increment update_count on each upsert, and set last_seen from the stream timestamp."*

### 4.2 The Working Pipeline
This is the version that actually worked in Atlas (use `$toDate` not `$dateFromString`):

```json
{
  "name": "solar-device-state-upsert",
  "pipeline": [
    {
      "$source": {
        "connectionName": "sample_stream_solar"
      }
    },
    {
      "$set": {
        "_id": "$device_id",
        "efficiency_pct": {
          "$cond": {
            "if": { "$gt": ["$max_watts", 0] },
            "then": { "$round": [{ "$multiply": [{ "$divide": ["$obs.watts", "$max_watts"] }, 100] }, 1] },
            "else": null
          }
        },
        "last_seen": { "$toDate": "$timestamp" },
        "update_count": 1
      }
    },
    {
      "$unset": ["device_id", "timestamp"]
    },
    {
      "$merge": {
        "into": {
          "connectionName": "Atlas DB",
          "db": "solar-data",
          "coll": "device_state"
        },
        "on": "_id",
        "whenMatched": [
          {
            "$set": {
              "group_id": "$$new.group_id",
              "max_watts": "$$new.max_watts",
              "event_type": "$$new.event_type",
              "obs": "$$new.obs",
              "efficiency_pct": "$$new.efficiency_pct",
              "last_seen": "$$new.last_seen",
              "update_count": { "$add": ["$update_count", 1] }
            }
          }
        ],
        "whenNotMatched": "insert"
      }
    }
  ]
}
```

### 4.3 Create the Stream Processor in Atlas UI
- Stream Processing workspace → **Create Stream Processor**
- Paste the JSON above (Atlas UI accepts the full `name` + `pipeline` structure)
- Replace `Atlas DB` with whatever you named your Atlas cluster connection
- Start the processor

### 4.4 Verify Live Data
Atlas → Browse Collections → `solar-data` → `device_state`:
- Document count = number of unique devices (NOT growing endlessly) ✓
- `update_count` increments on each refresh ✓
- `efficiency_pct` is populated ✓
- `last_seen` is updating ✓

If documents keep being inserted (count grows): the `$merge` upsert is misconfigured — check `on: "_id"`.

---

## Phase 5: Bonus — Time-Series Moment

The `raw-data` Time-Series collection captures every raw event without deduplication. You can extend the pipeline with a second `$merge` (insert-only) to write to `raw-data` as well.

> **Video moment:** "device_state tells you what's happening right now. raw-data tells you everything that ever happened. Same database, different patterns."

---

## Phase 6: Explore Live Data with Claude Code

With the pipeline running, ask Claude Code natural language questions. The Natural Language Querying skill translates them into MongoDB aggregations and runs them via MCP:

**Suggested questions:**
- *"Which solar device has the highest efficiency_pct right now?"*
- *"Show me all devices in group 1 and their current output"*
- *"Which devices are running below 20% efficiency?"*
- *"Which device hasn't sent a reading in the last 2 minutes?"*
- *"What's the total power output across all devices right now?"*

Ask Claude to show the generated query each time — that's the Natural Language Querying skill in action.

---

## Cleanup
**Terminate the stream processor** in Atlas when done to stop billing ($0.06/hr while running).

---

## Summary

| Component | Technology |
|---|---|
| Cloud database | MongoDB Atlas (M0 free tier) |
| Live data source | Atlas Sample Solar Stream (`sample_stream_solar`) |
| Stream processing | Atlas Stream Processing (SP2, $0.06/hr) |
| Live state store | `device_state` — regular collection, one doc per device |
| Historical archive | `raw-data` — Time-Series collection |
| Agent Skills | MongoDB plugin via Claude Code marketplace |
| MCP connection | `claude mcp add` with Atlas connection string |
| Live data exploration | Claude Code + MCP (Natural Language Querying skill) |

---

## Gotchas & Lessons Learned

- **No free tier for Stream Processing** — SP2 is $0.06/hr. Terminate when not recording.
- **No stock price sample stream** — only `sample_stream_solar` is available.
- **Time-Series collections don't support upserts** — use a regular collection for live state.
- **Node.js must be v22** — MCP server fails on v18. Use the nodejs.org `.pkg` installer.
- **Plugin ≠ MCP** — Plugin gives Skills, `claude mcp add` gives Atlas access. You need both.
- **Don't install the VS Code MongoDB extension AND the Claude Code plugin** — they both register MCP servers and conflict. Use only the Claude Code plugin.
- **Connection string is stored in plain text** in `claude mcp list` output. Rotate password before/after recording.
- **Atlas UI Stream Processor accepts JSON with `name` + `pipeline`** wrapper — not just an array.
- **Use `$toDate` not `$dateFromString`** for the timestamp conversion — simpler and works reliably.
- **Connection name in `$merge` must match exactly** what you registered in Connection Registry.
- **Collection doesn't need documents to have an index** — create indexes upfront before data lands.