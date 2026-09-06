# PersonaStore Documentation

PersonaStore is a robust, production-ready DataStore persistence framework with native EncodingService integration, compression, data integrity verification, and advanced session management. It also wraps OrderedDataStores, MemoryStoreService, and DataStore versioning behind the same retry/backoff layer as the core profile engine.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Core API Reference](#core-api-reference)
3. [EncodingService Integration](#encodingservice-integration)
4. [Compression & Hashing](#compression--hashing)
5. [Integrity Modes](#integrity-modes)
6. [Read-Only Access](#read-only-access)
7. [Batch Operations](#batch-operations)
8. [OrderedDataStore Support](#ordereddatastore-support)
9. [MemoryStoreService Support](#memorystoreservice-support)
10. [DataStore Version APIs](#datastore-version-apis)
11. [Serialization Engine](#serialization-engine)
12. [BufferArray Utility](#bufferarray-utility)
13. [Statistics & Monitoring](#statistics--monitoring)
14. [Advanced Features](#advanced-features)

---

## Getting Started

Before loading any player data, initialize the engine once:

```lua
local PersonaStore = require(ServerStorage.PersonaStore)

PersonaStore:Init()
```

Create isolated DataStores with optional compression configuration:

```lua
local PlayerStore = PersonaStore:CreateDataStore("PlayerData_v1", {
    Schema = PlayerTemplate,
    AutoSaveInterval = 30
})
```

Configure compression settings globally:

```lua
-- Use ZSTD compression (better ratio, slower) with level 9
PersonaStore:SetCompressionSettings(Enum.CompressionAlgorithm.Zstd, 9)
```

Configure how much a `SavePatch()` hash pass costs (see [Integrity Modes](#integrity-modes) below):

```lua
-- Only rehash fields that actually changed, instead of the whole profile
PersonaStore:SetIntegrityMode("PerField")
```

Load and interact with sessions:

```lua
local session = PlayerStore:LoadSession(tostring(player.UserId))
if session then
    session.Data.Coins += 100
    session:SavePatch()
    session:Destroy()
end
```

---

## Core API Reference

### PersonaStore

The engine singleton that manages all DataStores, OrderedDataStores, MemoryStores, compression, and cluster communication.

#### PersonaStore:Init()

Starts the shared MessagingService subscription and registers the engine-wide shutdown handler.

```lua
PersonaStore:Init()
```

**Must be called once** before any `LoadSession()` calls.

---

#### PersonaStore:CreateDataStore(storeName, configOptions)

Creates or returns an existing `Founder` (isolated DataStore wrapper).

```lua
local PlayerStore = PersonaStore:CreateDataStore("PlayerData", {
    Schema = PlayerTemplate,
    AutoSaveInterval = 30
})
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `storeName` | `string` | Yes | Name of the Roblox DataStore |
| `configOptions` | `table` | No | Configuration table |

**Configuration Options:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `Schema` | `table` | `{}` | Template for new profiles |
| `AutoSaveInterval` | `number` | `30` | Seconds between autosaves |
| `SerializationManifest` | `table` | `{}` | `{fieldName = typeName, ...}` applied to every session loaded from this store — see [Serialization Engine](#serialization-engine) |

---

#### PersonaStore:SetCompressionSettings(algorithm, level)

Configures compression algorithm and level for all future compression operations.

```lua
PersonaStore:SetCompressionSettings(Enum.CompressionAlgorithm.Ztsd, 9)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `algorithm` | `Enum.CompressionAlgorithm` | `Zstd (only method as of now)` |
| `level` | `number` | 1-22 depending on algorithm (1=faster, 22=better compression) |

**Recommendations:**
- **Deflate, Level 6**: Good balance of speed and compression (default)
- **ZSTD, Level 9**: Best for data-heavy profiles (slower but ~20-30% better compression)
- **ZSTD, Level 22**: Maximum compression for archival/exports

> **Note:** The algorithm used is recorded per-profile in `CompressionMetadata.Algorithm`. Changing this setting mid-project is safe — existing compressed profiles are always decompressed with whatever algorithm they were originally saved with, not whatever the current global default is.

---

#### PersonaStore:SetIntegrityMode(mode)

**NEW in v1.2.0** — Controls how much work `SavePatch()` does to keep integrity hashes current. See [Integrity Modes](#integrity-modes) for full details.

```lua
PersonaStore:SetIntegrityMode("PerField")
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `mode` | `string` | One of `"Full"`, `"HashOnlyOnFullSave"`, `"PerField"` |

Invalid modes are ignored (with a `warn()`), leaving the current mode unchanged.

---

#### PersonaStore:CreateOrderedDataStore(storeName)

**NEW in v1.2.0** — Creates or returns an existing `OrderedFounder`, a thin wrapper over `DataStoreService:GetOrderedDataStore()`. See [OrderedDataStore Support](#ordereddatastore-support).

```lua
local Leaderboard = PersonaStore:CreateOrderedDataStore("WeeklyLeaderboard_v1")
```

---

#### PersonaStore:CreateMemoryQueue(name) / PersonaStore:CreateMemorySortedMap(name)

**NEW in v1.2.0** — Creates or returns cached wrappers over `MemoryStoreService` queues and sorted maps. See [MemoryStoreService Support](#memorystoreservice-support).

```lua
local PurchaseQueue = PersonaStore:CreateMemoryQueue("PendingPurchases")
local ActiveMatches = PersonaStore:CreateMemorySortedMap("ActiveMatches")
```

---

#### PersonaStore:RegisterSerializer(typeName, serializer)

**NEW in v1.3.0** — Registers a custom (de)serializer for a type name, making it usable in a `Founder`'s `SerializationManifest`, a session's `SetSerialize()` / `MarkFieldSerialized()`, or passed explicitly to `DataSession:Serialize(value, typeName)`. See [Serialization Engine](#serialization-engine).

```lua
PersonaStore:RegisterSerializer("Currency", {
    Serialize = function(c) return {Amount = c.Amount, Kind = c.Kind} end,
    Deserialize = function(d) return Currency.new(d.Amount, d.Kind) end,
})
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `typeName` | `string` | Name used to reference this type elsewhere (e.g. in a manifest) |
| `serializer` | `table` | `{Serialize = function(value) -> storageSafeData, Deserialize = function(data) -> value}` |

**Returns:** `boolean` — `false` (with a `warn()`) if `serializer` doesn't have both functions

`Vector3`, `Vector2`, `CFrame`, `Color3`, `UDim`, and `UDim2` are already registered out of the box; you only need this for your own custom types.

---

#### PersonaStore:GetStatistics()

Returns engine-wide statistics since server start.

```lua
local stats = PersonaStore:GetStatistics()
print("Total saves:", stats.TotalSaves)
print("Bytes saved by compression:", stats.BytesSavedByCompression)
print("DataStore requests failed:", stats.DatastoreRequestsFailed)
```

**Returned Table:**

| Field | Type | Description |
|-------|------|-------------|
| `TotalSaves` | `number` | Count of successful `Save()` / `SavePatch()` calls |
| `TotalLoads` | `number` | Count of sessions loaded |
| `TotalCompressions` | `number` | Count of compression operations |
| `TotalDecompressions` | `number` | Count of decompression operations |
| `DatastoreRequestsAttempted` | `number` | Total DataStore API calls attempted |
| `DatastoreRequestsFailed` | `number` | Failed DataStore API calls |
| `BytesSavedByCompression` | `number` | Total bytes reduced through compression |

---

### Founder

Represents a single isolated DataStore with session management.

#### Founder:LoadSession(key)

Acquires ownership of a profile and returns a `DataSession`.

```lua
local session = Store:LoadSession(tostring(player.UserId))
if not session then
    player:Kick("Data is still loading. Please rejoin.")
    return
end
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
|  key | `string / number` | Yes | Profile key (usually player UserID) |

**Returns:** `DataSession?` — `nil` if ownership could not be acquired

---

#### Founder:LoadSessionAsync(key, maxWaitSeconds)

Repeatedly attempts to acquire a session for up to `maxWaitSeconds`.

```lua
local session = Store:LoadSessionAsync(tostring(player.UserId), 8)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
|  key | `string / number` | Yes | Profile key |
| `maxWaitSeconds` | `number` | No | Timeout in seconds (default: 10) |

**Best for:** Player rejoin after crashes, where waiting 5-10 seconds is preferable to kicking.

---

#### Founder:LoadReadOnlySession(key)

Loads profile data without acquiring a lock (read-only, no mutations).

```lua
local readOnlyData = Store:LoadReadOnlySession(tostring(userId))
if readOnlyData then
    print("Coins:", readOnlyData.Data.Coins)
    print("Level:", readOnlyData.Data.Level)
end
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
|  key | `string / number` | Yes | Profile key |

**Returns:** `table?` — The profile data, or `nil` if not found. Compressed profiles are transparently decompressed before being returned, and any `SerializationManifest`-marked fields are converted back to real Vector3/CFrame/etc. objects.

**Use cases:**
- Leaderboard queries without locking the player
- Admin panels viewing player data
- Cross-server data aggregation

---

#### Founder:PublishGlobalUpdate(key, payload)

Sends an update to a profile regardless of whether the player is online or which server owns the session.

```lua
PlayerStore:PublishGlobalUpdate(tostring(userId), {
    Type = "Currency",
    Amount = 500,
    Reason = "DailyReward"
})
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
|  key | `string / number` | Yes | Profile key |
| `payload` | `table` | Yes | Arbitrary data describing the update |

**Returns:** `boolean` — Whether the update was successfully queued

**Best for:**
- Granting items to offline players
- Delivering web store purchases
- Admin actions (bans, warnings, etc.)

---

#### Founder:BatchUpdate(keys, transformFn)

Atomically updates multiple profiles with a transformation function.

```lua
local results = PlayerStore:BatchUpdate({userId1, userId2, userId3}, function(data)
    data.SeasonPoints = 0  -- Reset season points
    data.SeasonRank = 1
end)

for key, success in pairs(results) do
    if success then
        print("Successfully reset", key)
    end
end
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `keys` | `{string}` | Yes | Array of profile keys |
| `transformFn` | `(data: table) -> ()` | Yes | Function to transform each profile |

**Returns:** `{[string]: boolean}` — Success status per key

**Important:** `BatchUpdate` will wait up to 5 seconds per key. Avoid with very large arrays. Each key is written to the DataStore exactly once (a full `Save()` via `Destroy()`), regardless of the outcome of `transformFn`.

---

#### Founder:GetKeyMetadata(key)

Retrieves metadata about a profile without loading it.

```lua
local meta = Store:GetKeyMetadata(tostring(userId))
if meta then
    print("Version:", meta.Version)
    print("Last updated:", os.date("%Y-%m-%d %H:%M:%S", meta.LastUpdate))
    print("Compression:", meta.CompressionMetadata.IsCompressed)
end
```

**Returns:** `table?` — Metadata, or `nil` if not found

**Returned Table:**

| Field | Type | Description |
|-------|------|-------------|
| `Version` | `number` | Profile version (incremented on each save) |
| `LastUpdate` | `number` | Unix timestamp of last save |
| `SessionToken` | `string?` | Current session token if locked |
| `JobId` | `string?` | Server JobId currently owning the lock |
| `DataHash` | `string?` | Sha256 hash of the whole profile (if enabled; may be stale in `"PerField"` mode until the next full `Save()`) |
| `FieldHashes` | `table?` | Per-field Sha256 hashes (if enabled) |
| `CompressionMetadata` | `table?` | Compression info |

---

#### Founder:GetCompressionStats()

Returns compression statistics for this store.

```lua
local stats = Store:GetCompressionStats()
print("Bytes saved:", stats.BytesSavedByCompression)
```

---

#### Founder:GetActiveSessionCount()

Returns the number of currently loaded sessions on this server.

```lua
local count = Store:GetActiveSessionCount()
print("Active sessions:", count)
```

---

#### Founder:ListVersionsAsync(key, sortDirection, minDate, maxDate, pageSize)

**NEW in v1.2.0** — See [DataStore Version APIs](#datastore-version-apis).

---

#### Founder:GetVersionAsync(key, version)

**NEW in v1.2.0** — See [DataStore Version APIs](#datastore-version-apis). As of v1.3.0, returned `Data` also has any `SerializationManifest`-marked fields converted back to real Vector3/CFrame/etc. objects, the same as `LoadSession()` / `LoadReadOnlySnapshot()`.

---

#### Founder:RemoveVersionAsync(key, version)

**NEW in v1.2.0** — See [DataStore Version APIs](#datastore-version-apis).

---

### DataSession

Represents an actively loaded, locked profile. This is the only object for reading/writing profile data.

#### DataSession.Data

The observable data table for the loaded profile.

Every write at any nesting depth marks the top-level field as dirty and fires `ListenToFieldChange` listeners.

```lua
session.Data.Coins += 500
session.Data.Inventory.Sword = true
session.Data.Stats.Level = 10
```

---

#### DataSession:Save()

Writes the entire profile back to the DataStore.

```lua
local success = session:Save()
```

**Returns:** `boolean` — Whether the save succeeded

**Notes:**
- Costs more bandwidth than `SavePatch()`
- Confirms lock token before writing
- Recomputes both the whole-profile `DataHash` and the per-field `FieldHashes` table, regardless of the current `IntegrityMode` — a full save always re-establishes a clean baseline
- Any fields marked in the session's `SerializationManifest` are converted to storage-safe form before being written and hashed
- Use for important checkpoints

---

#### DataSession:SavePatch()

Writes only the top-level fields that changed since the last save.

```lua
session:SavePatch()
```

**Returns:** `boolean` — Whether the save succeeded (returns `true` if nothing changed)

**Notes:**
- This is what the autosave heartbeat calls
- Dramatically reduces bandwidth on large profiles
- Prefer over `Save()` for routine autosaving
- The cost of the integrity hash pass depends on `PersonaStore.IntegrityMode` — see [Integrity Modes](#integrity-modes)
- Manifest-marked fields among the dirty fields are serialized before being written and hashed, same as `Save()`

---

#### DataSession:SaveCompressed()

Saves the entire profile with EncodingService compression.

```lua
local success = session:SaveCompressed()
```

**Returns:** `boolean` — Whether the save succeeded

**Best for:**
- Large profiles with deep inventories
- End-of-session checkpoints
- Reducing storage costs

**Compression ratio:** Typically 40-70% depending on data structure and algorithm

**Note:** Manifest-marked fields are serialized before JSON-encoding, since `HttpService:JSONEncode` can't handle Vector3/CFrame/etc. directly.

---

#### DataSession:BeginTransaction()

Takes a deep-copy snapshot of the current `Data` for later rollback.

```lua
session:BeginTransaction()

session.Data.Coins -= 100
session.Data.Inventory.Sword = true

if not validatePurchase() then
    session:RollbackTransaction()
else
    session:CommitTransaction()
    session:SavePatch()
end
```

---

#### DataSession:CommitTransaction()

Discards the snapshot, keeping data as-is.

```lua
session:CommitTransaction()
session:SavePatch()
```

---

#### DataSession:RollbackTransaction()

Restores `Data` to the pre-transaction state.

```lua
local success = session:RollbackTransaction()
```

**Returns:** `boolean` — `false` if no transaction was active

---

#### DataSession:IncrementCounter(fieldName, amount)

Atomically increments a counter in the data.

```lua
session:IncrementCounter("Coins", 100)  -- Coins += 100
session:IncrementCounter("Level")       -- Level += 1
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fieldName` | `string` | Yes | Field to increment |
| `amount` | `number` | No | Increment amount (default: 1) |

**Returns:** `boolean` — Whether the save succeeded

**Best for:**
- Leaderboard score increments
- Currency transactions
- Statistics counters

---

#### DataSession:ExportData(includeMetadata)

Exports profile data as JSON string for backup/migration.

```lua
local json = session:ExportData(true)
-- Save to file or send to backend...
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `includeMetadata` | `boolean` | No | Include session metadata in export |

**Returns:** `string` — JSON-encoded export

**Exported structure:**

```json
{
  "Data": { /* profile data */ },
  "ExportTime": 1234567890,
  "SessionToken": "...",  // if includeMetadata = true
  "Metadata": { /* session metadata */ }
}
```

**Note:** Manifest-marked fields are serialized into the export, so `Data` is plain, JSON-safe data. `ImportData()` reverses this automatically.

---

#### DataSession:ImportData(jsonString, overwrite)

Imports profile data from a JSON export.

```lua
local success = session:ImportData(jsonString, false)  -- Merge mode
-- or
local success = session:ImportData(jsonString, true)   -- Overwrite mode
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `jsonString` | `string` | Yes | JSON export string |
| `overwrite` | `boolean` | No | `true` = replace all data, `false` = merge (default) |

**Returns:** `boolean` — Whether import succeeded

---

#### DataSession:VerifyDataIntegrity()

Verifies profile data integrity using stored hash(es).

```lua
if not session:VerifyDataIntegrity() then
    warn("Profile data was modified externally!")
    session:Reload()
end
```

**Returns:** `boolean` — Whether the stored hash(es) match the current in-memory data

**Requires:** `PersonaStore.EnableDataIntegrityChecks = true` (enabled by default)

**Behavior depends on `IntegrityMode`:**
- `"Full"` / `"HashOnlyOnFullSave"` — compares against the whole-profile `DataHash`.
- `"PerField"` — compares each field in memory against its corresponding entry in `FieldHashes`.

---

#### DataSession:IsCacheStale()

Compares in-memory version against cloud version without acquiring a lock.

```lua
if session:IsCacheStale() then
    warn("Profile was modified by another server!")
end
```

**Returns:** `boolean` — `true` if cloud version is newer

---

#### DataSession:StartAutoSave(intervalSeconds)

Starts autosave loop with given interval.

```lua
session:StartAutoSave(15)  -- Save every 15 seconds
```

**Called automatically by `LoadSession()`** using the store's `AutoSaveInterval` config.

---

#### DataSession:StopAutoSave()

Cancels the autosave loop.

```lua
session:StopAutoSave()
```

---

#### DataSession:ListenToFieldChange(callback)

Fires whenever any field in `Data` changes at any nesting depth.

```lua
session:ListenToFieldChange(function(key, value, rootKey)
    if rootKey == "Coins" then
        print("Coins changed to:", session.Data.Coins)
        player.leaderstats.Coins.Value = session.Data.Coins
    end
end)
```

| Argument | Meaning |
|----------|---------|
|  | The literal field name that was assigned |
| `value` | The new value |
| `rootKey` | The top-level field under `Data` that was modified |

---

#### DataSession:ListenToGlobalUpdate(callback)

Fires whenever a live global update arrives from another server.

```lua
session:ListenToGlobalUpdate(function(payload)
    if payload.Type == "Currency" then
        StarterGui:SetCore("SendNotification", {
            Title = "Gift Received",
            Text = "+" .. payload.Amount .. " coins"
        })
    end
end)
```

---

#### DataSession:ConsumeGlobalUpdates()

Returns all queued global updates and clears the queue.

```lua
for _, update in session:ConsumeGlobalUpdates() do
    if update.Type == "Currency" then
        session.Data.Coins += update.Amount
    elseif update.Type == "Item" then
        session.Data.Inventory[update.ItemId] = true
    end
end
```

**Returns:** `{table}` — Array of update payloads

---

#### DataSession:GetPerformanceMetadata()

Returns runtime metadata about the session.

```lua
local meta = session:GetPerformanceMetadata()
print("Last saved:", os.date("%c", meta.LastSaved))
print("Save cycles:", meta.WriteCycles)
print("Compression stats:", meta.CompressionStats)
```

**Returned Table:**

| Field | Type | Description |
|-------|------|-------------|
| `Created` | `number` | Unix timestamp when session was created |
| `LastSaved` | `number` | Unix timestamp of last successful save |
| `WriteCycles` | `number` | Count of `Save()` / `SavePatch()` calls |
| `LastModified` | `number?` | Unix timestamp of last data mutation |
| `CompressionStats` | `table` | Latest compression stats |

---

#### DataSession:SetSerialize(manifest)

**NEW in v1.3.0** — Replaces this session's serialization manifest wholesale. See [Serialization Engine](#serialization-engine).

```lua
session:SetSerialize({
    Position = "Vector3",
    SpawnCFrame = "CFrame",
})
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `manifest` | `table` | Yes | `{fieldName = typeName, ...}` |

**Returns:** `boolean` — `false` (with a `warn()`) if `manifest` isn't a table

**Note:** This *replaces* the whole manifest, including whatever the session inherited from the store's `SerializationManifest` config. Use `MarkFieldSerialized()` instead if you just want to add one field.

---

#### DataSession:MarkFieldSerialized(fieldName, typeName)

**NEW in v1.3.0** — Adds or overrides a single field in this session's serialization manifest without touching the rest.

```lua
session:MarkFieldSerialized("LastPosition", "Vector3")
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fieldName` | `string` | Yes | Top-level field name in `Data` |
| `typeName` | `string` | Yes | Registered type name (e.g. `"Vector3"`, or a custom type from `RegisterSerializer`) |

**Returns:** `boolean` — `false` if either argument is missing

---

#### DataSession:Serialize(value, typeName?) / DataSession:Deserialize(value)

**NEW in v1.3.0** — Manual (de)serialization helpers, independent of the manifest. Useful for one-off conversions, e.g. before sending data over `PublishGlobalUpdate()`.

```lua
local safePosition = session:Serialize(player.Character.HumanoidRootPart.Position, "Vector3")
local realPosition = session:Deserialize(safePosition)
```

`Serialize()` auto-detects known Roblox types via `typeof()` when `typeName` is omitted, and recurses into plain tables so nested typed values are converted too.

---

#### DataSession:Release()

Releases ownership without saving.

```lua
session:Release()
```

**Prefer `Destroy()` for normal cleanup.**

---

#### DataSession:Destroy()

Saves the full profile, then releases the session.

```lua
session:Destroy()
```

**Use for:**
- `Players.PlayerRemoving`
- Planned session closure
- Any graceful disconnect

---

## EncodingService Integration

PersonaStore uses Roblox's native `EncodingService` for efficient serialization.

### Compression Handler

The `CompressionHandler` is an internal utility managing all encoding operations:

**Compression Algorithms:**

| Algorithm | Speed | Compression Ratio | Best For |
|-----------|-------|-------------------|----------|
| `Deflate` | Fast | ~40% | General purpose, balanced |
| `ZSTD` | Slower | ~60-70% | Large profiles, storage optimization |

**Levels:**

- **1-3:** Fastest compression, lower ratio (use for real-time saves)
- **6:** Default, good balance
- **9+:** Better compression, slower (use for end-of-session saves)
- **20-22:** Maximum compression (use for archives/exports)

Each compressed profile records the exact algorithm it was compressed with in `CompressionMetadata.Algorithm`, so decompression is always correct even if you later call `SetCompressionSettings()` with a different algorithm.

### Hash Verification

When `PersonaStore.EnableDataIntegrityChecks = true` (default):

- `Save()` computes both a whole-profile `DataHash` and a per-field `FieldHashes` table
- `SavePatch()` refreshes hashes according to the current `IntegrityMode`
- The hash(es) are stored with the profile
- `session:VerifyDataIntegrity()` checks whether the stored hash(es) match the in-memory data

**Hash Algorithms Supported:**

- `Sha256` (default, recommended)
- `SHA1` (faster, less secure)
- `MD5` (fastest, not recommended for security-critical data)

### Base64 Encoding

When compressed data is stored in DataStore (JSON-compatible format):

```lua
-- Internally:
local compressed = CompressionHandler.compress(profileData)
local base64 = CompressionHandler.encodeToBase64(compressed)
-- Stored as JSON-safe string in DataStore
```

---

## Integrity Modes

**NEW in v1.2.0.** `SavePatch()` used to rehash the *entire* profile on every call, even if only one small field changed. That's expensive once a profile has large, mostly-static fields (a 50,000-item `Inventory`, for example) sitting next to something that changes constantly (`Coins`). `PersonaStore.IntegrityMode` lets you trade off hashing cost against how fine-grained your integrity checks are.

| Mode | `SavePatch()` cost | What gets checked | Notes |
|------|--------------------|--------------------|-------|
| `"Full"` *(default)* | Rehashes the whole profile every patch | Whole-profile `DataHash` | Same behavior as v1.1.x; safest, but scales with total profile size, not with what changed |
| `"HashOnlyOnFullSave"` | No hashing at all | Whole-profile `DataHash`, but only current as of the last full `Save()` | Cheapest option; use if you only need integrity guarantees at checkpoints, not on every autosave |
| `"PerField"` | Rehashes only the fields in this patch | Per-field `FieldHashes` | Best of both worlds for profiles with large static fields next to frequently-changing ones |

```lua
-- Only rehash the fields that actually changed on each SavePatch()
PersonaStore:SetIntegrityMode("PerField")
```

`DataSession:VerifyDataIntegrity()` automatically checks the right structure (`DataHash` or `FieldHashes`) for whichever mode is active. `Save()` (the full save) always refreshes both `DataHash` and `FieldHashes` regardless of mode, so switching modes mid-project — or falling back from `"PerField"` to `"Full"` — is safe as long as a full `Save()` has run at least once since the switch.

**Important caveat:** in `"PerField"` and `"HashOnlyOnFullSave"` modes, the whole-profile `DataHash` returned by `Founder:GetKeyMetadata()` can go stale between full saves. Don't treat it as a live whole-profile checksum in those modes — use `VerifyDataIntegrity()`, which knows which structure to trust.

---

## Read-Only Access

Use `LoadReadOnlySession()` to query profiles without locks:

```lua
-- Leaderboard query
local topPlayers = {}
for _, userId in ipairs(topUserIds) do
    local data = PlayerStore:LoadReadOnlySession(tostring(userId))
    if data then
        table.insert(topPlayers, {
            UserId = userId,
            Coins = data.Data.Coins,
            Level = data.Data.Level
        })
    end
end
```

**Advantages:**
- No lock acquisition overhead
- No risk of lock conflicts
- Instant reads
- Perfect for queries and reporting

For true leaderboards with efficient range queries (top N, rank lookups), consider [OrderedDataStore Support](#ordereddatastore-support) instead — it's purpose-built for sorted numeric reads and doesn't require loading/decompressing full profile documents.

---

## Batch Operations

Use `BatchUpdate()` for mass profile modifications:

```lua
-- Grant season pass bonus to multiple players
local playerIds = {1234, 5678, 9012}

local results = PlayerStore:BatchUpdate(playerIds, function(data)
    data.SeasonPoints += 500
    data.Cosmetics.SeasonPass = true
end)

print("Successfully updated:", #(results or {}))
```

Each profile is locked individually, modified, and saved exactly once (a full `Save()`, via `Destroy()`). If a lock fails, that key's result is `false`.

---

## OrderedDataStore Support

**NEW in v1.2.0.** `OrderedFounder` wraps `DataStoreService:GetOrderedDataStore()` for cases where you just need a sorted, numeric leaderboard — no session locks, schemas, or compression, since OrderedDataStores only ever store non-negative integers.

```lua
local Leaderboard = PersonaStore:CreateOrderedDataStore("WeeklyLeaderboard_v1")

-- Set an explicit score
Leaderboard:Set(tostring(player.UserId), 4200)

-- Read it back
local score = Leaderboard:Get(tostring(player.UserId))

-- Atomically bump it
Leaderboard:Increment(tostring(player.UserId), 50)

-- Remove an entry entirely
Leaderboard:Remove(tostring(player.UserId))

-- Read a sorted page (top 50, descending)
local topPage, pages = Leaderboard:GetSortedPage(false, 50)
for _, entry in ipairs(topPage) do
    print(entry.key, entry.value)
end

-- Keep paging with the native DataStorePages object
if not pages.IsFinished then
    pages:AdvanceToNextPageAsync()
end
```

| Method | Description |
|--------|-------------|
| `Set(key, value)` | Sets an explicit non-negative integer value |
| `Get(key)` | Reads the current value, or `nil` |
| `Increment(key, delta?)` | Atomically adds `delta` (default `1`), returns the new value |
| `Remove(key)` | Deletes the key |
| `GetSortedPage(ascending?, pageSize?, minValue?, maxValue?)` | Returns `(currentPage, pages)` — a plain array of `{key, value}` entries plus the native `DataStorePages` object for further pagination |

---

## MemoryStoreService Support

**NEW in v1.2.0.** For short-lived, high-throughput, ephemeral data that doesn't need DataStore durability — matchmaking queues, purchase-processing jobs, active-match tracking — PersonaStore wraps `MemoryStoreService` queues and sorted maps with the same retry/backoff behavior as everything else.

### MemoryQueueWrapper

```lua
local PurchaseQueue = PersonaStore:CreateMemoryQueue("PendingPurchases")

-- Add a job, expiring after 1 hour if never read
PurchaseQueue:AddItem({ UserId = player.UserId, ProductId = 123 }, 3600)

-- Read up to 5 items, waiting up to 5 seconds for something to arrive
local items, receiptId = PurchaseQueue:ReadItems(5, false, 5)
if items then
    for _, item in ipairs(items) do
        -- process item...
    end
    -- Only remove once processing succeeded
    PurchaseQueue:RemoveItems(receiptId)
end
```

| Method | Description |
|--------|-------------|
| `AddItem(value, expirationSeconds?, priority?)` | Adds an item to the queue (default expiration: 1 hour) |
| `ReadItems(count?, allOrNothing?, waitTimeoutSeconds?)` | Returns `(items, receiptId)`, or `(nil, nil)` on failure |
| `RemoveItems(receiptId)` | Removes a previously-read batch using its receipt ID |

### MemorySortedMapWrapper

```lua
local ActiveMatches = PersonaStore:CreateMemorySortedMap("ActiveMatches")

ActiveMatches:Set(matchId, { Players = {...}, StartedAt = os.time() }, 1800)

local match = ActiveMatches:Get(matchId)

ActiveMatches:Remove(matchId)

local recentMatches = ActiveMatches:GetRange(Enum.SortDirection.Descending, 20)

ActiveMatches:Update(matchId, function(oldValue)
    oldValue.Players = oldValue.Players or {}
    table.insert(oldValue.Players, newPlayerId)
    return oldValue
end)
```

| Method | Description |
|--------|-------------|
| `Set(key, value, expirationSeconds?, sortKey?)` | Sets a key's value (default expiration: 1 hour) |
| `Get(key)` | Returns the value, or `nil` |
| `Remove(key)` | Deletes the key |
| `GetRange(direction?, count?, exclusiveLowerBound?, exclusiveUpperBound?)` | Returns a sorted array of entries |
| `Update(key, transformFn, expirationSeconds?)` | Atomically transforms a key's value |

---

## DataStore Version APIs

**NEW in v1.2.0.** Thin, retry-wrapped access to Roblox's native DataStore version history, exposed on `Founder`.

```lua
-- List recent historical versions of a key
local pages = PlayerStore:ListVersionsAsync(tostring(userId), Enum.SortDirection.Descending)
if pages then
    for _, versionInfo in ipairs(pages:GetCurrentPage()) do
        print(versionInfo.Version, os.date("%c", versionInfo.CreatedTime / 1000))
    end
end

-- Fetch a specific historical version (auto-decompressed if it was stored compressed)
local oldSnapshot = PlayerStore:GetVersionAsync(tostring(userId), someVersionId)
if oldSnapshot then
    print("Coins at that version:", oldSnapshot.Data.Coins)
end

-- Permanently delete a specific historical version
PlayerStore:RemoveVersionAsync(tostring(userId), someVersionId)
```

| Method | Description |
|--------|-------------|
| `ListVersionsAsync(key, sortDirection?, minDate?, maxDate?, pageSize?)` | Returns the native `DataStoreVersionPages` object, or `nil` on failure |
| `GetVersionAsync(key, version)` | Returns a deep copy of that historical version's raw record (with `Data` decompressed if needed), or `nil` |
| `RemoveVersionAsync(key, version)` | Permanently removes a historical version. **This cannot be undone.** Returns `boolean` |

**Best for:**
- Investigating support tickets ("what did this player's inventory look like yesterday?")
- Manual rollback tooling for corrupted or disputed profiles
- Compliance/audit requirements around historical data retention

---

## Serialization Engine

**NEW in v1.3.0.** Roblox types like `Vector3`, `CFrame`, and `Color3` can't be JSON-encoded, so historically you had to manually convert them to plain tables before every save and back after every load. The serialization engine automates that: mark a field as a given type once, and every save/load path converts it for you.

### Built-in types

`Vector3`, `Vector2`, `CFrame`, `Color3`, `UDim`, and `UDim2` are registered out of the box.

### Marking fields — store-wide

Set `SerializationManifest` when creating a store so every session loaded from it gets the same conversions automatically:

```lua
local PlayerStore = PersonaStore:CreateDataStore("PlayerData_v2", {
    Schema = {
        Coins = 0,
        LastPosition = Vector3.new(0, 0, 0),
        SpawnCFrame = CFrame.new(),
    },
    SerializationManifest = {
        LastPosition = "Vector3",
        SpawnCFrame = "CFrame",
    },
})

local session = PlayerStore:LoadSession(tostring(player.UserId))

-- Read/write real Vector3/CFrame objects like normal -- conversion happens
-- automatically at Save() / SavePatch() / SaveCompressed() time, and is
-- reversed automatically at LoadSession() time.
session.Data.LastPosition = player.Character.HumanoidRootPart.Position
session:SavePatch()
```

### Marking fields — per session

If only some sessions from a store need a conversion, or you'd rather not touch the store config, use the session-level methods instead:

```lua
-- Add one field without disturbing the rest of the manifest
session:MarkFieldSerialized("SpawnCFrame", "CFrame")

-- Or replace the whole manifest for this session
session:SetSerialize({
    LastPosition = "Vector3",
    SpawnCFrame = "CFrame",
})
```

### Custom types

Register your own type once (usually at server start, before any `LoadSession()` calls that reference it), then use it in a manifest just like a built-in:

```lua
PersonaStore:RegisterSerializer("Currency", {
    Serialize = function(c)
        return { Amount = c.Amount, Kind = c.Kind }
    end,
    Deserialize = function(d)
        return Currency.new(d.Amount, d.Kind)
    end,
})

local PlayerStore = PersonaStore:CreateDataStore("PlayerData_v2", {
    Schema = { Wallet = Currency.new(0, "Gold") },
    SerializationManifest = { Wallet = "Currency" },
})
```

### Manual (de)serialization

For one-off conversions outside of a manifest — for example, before sending a `Vector3` through `PublishGlobalUpdate()`, which only accepts JSON-safe data:

```lua
local safePosition = session:Serialize(player.Character.HumanoidRootPart.Position, "Vector3")
PlayerStore:PublishGlobalUpdate(tostring(targetUserId), {
    Type = "Teleport",
    Position = safePosition,
})

-- On the receiving end:
session:ListenToGlobalUpdate(function(payload)
    if payload.Type == "Teleport" then
        local realPosition = session:Deserialize(payload.Position)
    end
end)
```

`Serialize(value, typeName?)` auto-detects known Roblox types via `typeof()` when `typeName` is omitted, and recurses into plain tables, so nested typed values (e.g. an array of `CFrame`s) are converted correctly too.

### Where conversion happens

| Direction | When |
|-----------|------|
| Real object → storage-safe | Immediately before `Save()`, `SavePatch()`, and `SaveCompressed()` write/hash their data; and inside `ExportData()` |
| Storage-safe → real object | Immediately after `LoadSession()`, `LoadReadOnlySnapshot()`/`LoadReadOnlySession()`, and `GetVersionAsync()` read their data; and inside `ImportData()` |

Because conversion happens right at the read/write boundary, `session.Data` always holds real Vector3/CFrame/etc. objects while you're working with it — you never see the `{__serialized = true, ...}` wrapper form unless you call `Serialize()`/`Deserialize()` manually.

---

## BufferArray Utility

**NEW in v1.3.0.** A thin, typed wrapper over Luau's `buffer` primitive, useful for storing compact numeric data — leaderboard snapshots, grids, particle-ish data — far more cheaply than an equivalent Lua table, then persisting it as a single base64 string field.

### Creating an array

```lua
-- 1000 unsigned 32-bit integers
local scores = PersonaStore.BufferArray.new("u32", 1000)
```

Supported element types: `u8`, `i8`, `u16`, `i16`, `u32`, `i32`, `u64`, `i64`, `f32`, `f64`.

### Reading and writing

```lua
scores:Set(0, 4200)
scores:Set(1, 3150)

print(scores:Get(0))  -- 4200

scores:SetRange(0, {4200, 3150, 2800})
local slice = scores:GetRange(0, 3)  -- {4200, 3150, 2800}

scores:Fill(0)  -- zero out every element
```

| Method | Description |
|--------|-------------|
| `Get(index)` | Reads the element at `index` (0-based) |
| `Set(index, value)` | Writes `value` at `index` (0-based) |
| `GetRange(startIndex, length)` | Returns a plain array of `length` elements starting at `startIndex` |
| `SetRange(startIndex, values)` | Writes a plain array of values starting at `startIndex` |
| `Fill(value)` | Sets every element to `value` |
| `GetBuffer()` | Returns the underlying raw `buffer` |
| `GetMetadata()` | Returns `{ElementType, ElementSize, ElementCount, TotalSize}` |

### Persisting to a DataStore field

`BufferArray` doesn't save itself — encode it to a string and store that string in your profile's `Data` like any other field:

```lua
-- Save
session.Data.ScoreSnapshot = scores:EncodeToBase64()
session:SavePatch()

-- Load
local restored = PersonaStore.BufferArray.DecodeFromBase64("u32", session.Data.ScoreSnapshot)
print(restored:Get(0))
```

**Best for:**
- Large fixed-size numeric datasets (leaderboard snapshots, heatmaps, tick history)
- Cases where the per-element overhead of a Lua table (and its JSON encoding) is too expensive
- Data you're willing to treat as a flat typed array rather than a keyed table

---

## Statistics & Monitoring

Monitor server health and performance:

```lua
-- Create a monitoring loop
while true do
    task.wait(300)  -- Every 5 minutes

    local stats = PersonaStore:GetStatistics()
    local playerStore = PersonaStore.RegisteredStores["PlayerData"]

    print("=== Engine Statistics ===")
    print("Active sessions:", playerStore:GetActiveSessionCount())
    print("Total loads:", stats.TotalLoads)
    print("Total saves:", stats.TotalSaves)
    print("Failed DataStore requests:", stats.DatastoreRequestsFailed)
    print("Compression saved bytes:", stats.BytesSavedByCompression / 1024 .. " KB")
    print("Compression ratio:", math.round(
        100 * (1 - stats.BytesSavedByCompression / (stats.TotalCompressions * 1000))
    ) .. "%")
end
```

---

## Advanced Features

### Compression Caching

PersonaStore caches compressed data to avoid recompressing unchanged profiles:

```lua
-- First SaveCompressed() compresses the data
session:SaveCompressed()

-- Subsequent SaveCompressed() reuses cached result
session:SaveCompressed()
```

Cache is automatically invalidated when `Data` is mutated.

### Selective Compression

Use `SaveCompressed()` only when needed:

```lua
-- Autosave with regular SavePatch()
session:StartAutoSave(30)

-- Manual full compression save on important events
if playerWonTournament then
    session:SaveCompressed()  -- Compress for better storage ratio
end
```

### Data Validation

Combine transactions and integrity checks:

```lua
session:BeginTransaction()

local hadCoins = session.Data.Coins
session.Data.Coins -= 100

if not validateTransaction() or session.Data.Coins < 0 then
    session:RollbackTransaction()
else
    session:CommitTransaction()
    session:SavePatch()
    if not session:VerifyDataIntegrity() then
        warn("Transaction verification failed!")
    end
end
```

### Cheap Patch Hashing for Large Profiles

For profiles with large, mostly-static fields (deep inventories, achievement logs) sitting next to frequently-changing ones (currency, XP), switch to per-field hashing so routine autosaves don't pay to rehash data that didn't change:

```lua
PersonaStore:SetIntegrityMode("PerField")

session:StartAutoSave(15)
-- Every autosave now only rehashes whichever top-level fields were actually dirtied
```

---

## Best Practices

1. **Always initialize:** Call `PersonaStore:Init()` once at server start
2. **Use `SavePatch()` by default:** Significantly reduces bandwidth for most use cases
3. **Enable integrity checks:** `PersonaStore.EnableDataIntegrityChecks = true` catches data corruption
4. **Pick the right `IntegrityMode`:** `"Full"` if you want the strongest guarantees on every patch; `"PerField"` if you have large static fields next to frequently-changing ones; `"HashOnlyOnFullSave"` if you only need integrity checks at checkpoints
5. **Use compression strategically:** `SaveCompressed()` for large profiles, important checkpoints
6. **Read-only for queries:** Use `LoadReadOnlySession()` for ad-hoc profile lookups; use `OrderedDataStore` for true sorted leaderboards
7. **Clean up sessions:** Always call `Destroy()` on player leave
8. **Monitor statistics:** Track engine health with `GetStatistics()`
9. **Configure compression:** Tune compression algorithm and level per your data size
10. **Reach for MemoryStore for ephemeral data:** Don't put short-lived queue/job data in a DataStore-backed profile — use `CreateMemoryQueue`/`CreateMemorySortedMap` instead
11. **Register custom serializers once, up front:** Call `PersonaStore:RegisterSerializer()` at startup, before any `CreateDataStore()`/`LoadSession()` calls that reference the type in a manifest

---

## Example: Complete Game Loop

```lua
local PersonaStore = require(ServerStorage.PersonaStore)

-- Configure engine
PersonaStore:SetCompressionSettings(Enum.CompressionAlgorithm.Zstd, 6)
PersonaStore:SetIntegrityMode("PerField")
PersonaStore:Init()

-- Create stores
local PlayerStore = PersonaStore:CreateDataStore("PlayerData_v2", {
    Schema = {
        Coins = 0,
        Level = 1,
        Inventory = {},
        Stats = { Kills = 0, Deaths = 0 },
        LastPosition = Vector3.new(0, 0, 0)
    },
    AutoSaveInterval = 30,
    SerializationManifest = {
        LastPosition = "Vector3",
    },
})

local Leaderboard = PersonaStore:CreateOrderedDataStore("WeeklyLeaderboard_v1")

-- On player join
Players.PlayerAdded:Connect(function(player)
    local session = PlayerStore:LoadSessionAsync(tostring(player.UserId), 10)

    if not session then
        player:Kick("Failed to load data. Please rejoin.")
        return
    end

    -- Listen for changes
    session:ListenToFieldChange(function(key, value, rootKey)
        if rootKey == "Coins" then
            player.leaderstats.Coins.Value = session.Data.Coins
            Leaderboard:Set(tostring(player.UserId), session.Data.Coins)
        end
    end)

    -- Process queued global updates
    for _, update in session:ConsumeGlobalUpdates() do
        if update.Type == "Currency" then
            session.Data.Coins += update.Amount
        end
    end

    player:SetAttribute("Session", session)
end)

-- On player leave
Players.PlayerRemoving:Connect(function(player)
    local session = player:GetAttribute("Session")
    if session then
        session.Data.LastPosition = player.Character.HumanoidRootPart.Position
        session:Destroy()  -- Save and release
    end
end)

-- Periodic monitoring
while true do
    task.wait(300)

    local stats = PersonaStore:GetStatistics()
    print("Active sessions:", PlayerStore:GetActiveSessionCount())
    print("Total saves:", stats.TotalSaves)
    print("Bytes saved by compression:", stats.BytesSavedByCompression / (1024 * 1024) .. " MB")
end
```

---

## Configuration Reference

### PersonaStore Settings

```lua
PersonaStore.CompressionAlgorithm = Enum.CompressionAlgorithm.Zstd
PersonaStore.CompressionLevel = 6
PersonaStore.EnableDataIntegrityChecks = true
PersonaStore.HashAlgorithm = Enum.HashAlgorithm.Sha256
PersonaStore.IntegrityMode = "Full" -- "Full" | "HashOnlyOnFullSave" | "PerField"
PersonaStore.GlobalLockTimeout = 120  -- seconds
PersonaStore.GlobalAutoSaveInterval = 30  -- seconds
```

All settings can be modified before `Init()` or via `:SetCompressionSettings()` / `:SetIntegrityMode()`.
