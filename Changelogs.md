# Changelogs

All notable changes to PersonaStore will be documented here.
This project follows **Semantic Versioning** (`MAJOR.MINOR.PATCH`).

---

## v2.0.0! 🥳
Fully rewritten, from the ground up, following the structure of my other modules.
No new syntax, everything will work just as before.

# v1.4.0

## Added
*Serialization Engine & BufferArray utility*  
*Automatic Rate Limiting for Datastore Requests*  

### Serialization Engine
- New global serializer registry, pre-populated with built-in support for `Vector3`, `Vector2`, `CFrame`, `Color3`, `UDim`, and `UDim2`.
- `PersonaStore:RegisterSerializer(typeName, serializer)` to register custom (de)serializers for your own types (e.g. a `Currency` or `Item` struct), where `serializer` is a table of `{Serialize = function(value) -> storageSafeData, Deserialize = function(data) -> value}`.
- New `SerializationManifest` config option on `Founder.new()` / `PersonaStore:CreateDataStore()` — a `{fieldName = typeName, ...}` table applied automatically to every session loaded from that store.
- `DataSession:SetSerialize(manifest)` — replaces the session's manifest wholesale. Sessions inherit a deep copy of the store's `SerializationManifest` by default, so calling this only affects that one session.
- `DataSession:MarkFieldSerialized(fieldName, typeName)` — adds or overrides a single field's manifest entry without replacing the rest of the manifest.
- `DataSession:Serialize(value, typeName?)` / `DataSession:Deserialize(value)` — manual (de)serialization helpers for one-off conversions outside the manifest. `Serialize()` auto-detects known Roblox types via `typeof()` when `typeName` is omitted, and recurses into plain tables so nested typed values (e.g. an array of `CFrame`s) are converted too.
- Manifest-marked fields are automatically converted to storage-safe form immediately before `Save()`, `SavePatch()`, `SaveCompressed()`, and `ExportData()` write/hash their data, and converted back immediately after `LoadSession()`, `LoadReadOnlySnapshot()`, `GetVersionAsync()`, and `ImportData()` read theirs.
- `PersonaStore.SerializeValue` / `PersonaStore.DeserializeValue` exposed on the module table for use without a live session, e.g. `PersonaStore.SerializeValue(myVector, "Vector3")`.

### BufferArray
- New `BufferArray` utility class: a thin, typed wrapper over Luau `buffer` for compact numeric arrays. Supports `u8`/`i8`/`u16`/`i16`/`u32`/`i32`/`u64`/`i64`/`f32`/`f64` element types.
- `BufferArray.new(elementType, elementCount)` to create a new fixed-size typed array.
- `:Get(index)` / `:Set(index, value)` for single-element access (0-based indexing).
- `:GetRange(startIndex, length)` / `:SetRange(startIndex, values)` for bulk reads/writes.
- `:Fill(value)` to fill every element with the same value.
- `:GetBuffer()` for direct raw `buffer` access, `:GetMetadata()` for element type/size/count/total-size info.
- `:EncodeToBase64()` and the static `BufferArray.DecodeFromBase64(elementType, encoded)` for round-tripping a `BufferArray` through a DataStore-safe string field.
- Exposed as `PersonaStore.BufferArray`.

## Changed
- N/A

## Fixed
- **`BufferArray:Get()` / `:Set()` indexed the buffer as if it were a table.** The original implementation attempted `self._buffer[self._readMethod](...)`, but a Luau `buffer` isn't indexable or callable that way — the reader/writer functions (`buffer.readu8`, `buffer.writeu8`, etc.) live on the global `buffer` library, not on the buffer value itself. Fixed to call through the `buffer` library directly, e.g. `buffer.readu8(self._buffer, offset)`.

---

### Migration Notes for v1.2.0 → v1.3.0
**100% Backward Compatible** — no breaking changes. `SerializationManifest` defaults to `{}` and `BufferArray` is purely additive; every existing store/session behaves exactly as before unless you opt in:

```lua
-- v1.2.0 code continues to work exactly as before
session:SavePatch()

-- Opt in to automatic Vector3/CFrame/etc. (de)serialization for a store
local PlayerStore = PersonaStore:CreateDataStore("PlayerData_v2", {
    Schema = PlayerTemplate,
    SerializationManifest = {
        LastPosition = "Vector3",
        SpawnCFrame = "CFrame",
    },
})

-- ...or per-session
session:MarkFieldSerialized("LastPosition", "Vector3")

-- New BufferArray utility is additive
local scores = PersonaStore.BufferArray.new("u32", 1000)
scores:Set(0, 4200)
```

---

# v1.2.0

## Added
*OrderedDataStore, MemoryStore, Version APIs, and configurable integrity hashing*

---

## Changed
- `withRetry()` no longer sleeps with exponential backoff after its *final* failed attempt (it used to wait needlessly before returning failure).
- Profile schema (`profileRaw`) now includes a `FieldHashes` table alongside the existing `DataHash`.
- `Founder:BatchUpdate()` no longer calls `SavePatch()` immediately before `Destroy()` — `Destroy()` already performs a full `Save()`, so every batched key was previously being written to the DataStore twice.

---

## Fixed
- **Require guard threw the wrong error / retried needlessly.** The server-only guard was calling `withRetry(task.spawn(function() ... end))`, which passes a *thread* into `withRetry`. `withRetry` does `pcall(thread)` internally, and a thread isn't callable, so every one of its 5 attempts failed immediately (burning through exponential backoff) before finally surfacing an unrelated pcall error instead of the intended "server only" message. Fixed to call `withRetry` with an actual function and `maxAttempts = 1`.
- **Decompression could use the wrong algorithm.** `CompressionHandler.decompress()` / `getDecompressedSize()` always decompressed using the *current global* `PersonaStore.CompressionAlgorithm` rather than whichever algorithm a given profile was actually compressed with. Calling `SetCompressionSettings()` to change the global algorithm made every previously-compressed profile fail to decompress correctly. Fixed by recording the algorithm's serialized name in `CompressionMetadata.Algorithm` at compress time and resolving it back to the correct `Enum.CompressionAlgorithm` at read time (`LoadSession`, `LoadReadOnlySnapshot`, and the new `GetVersionAsync` all go through this now).
- **`VerifyDataIntegrity()` almost always returned `false`.** It hashed the live observable proxy (`self.Data`) directly via `HttpService:JSONEncode`, but every `Save()` path hashes `deepCopy(self.Data)` — a plain table. `JSONEncode` doesn't respect the proxy's `__iter` metamethod, so the two hashes were computed over different representations and essentially never matched. Fixed to `deepCopy` before hashing, consistent with the save paths.
- **`BatchUpdate()` wrote every key twice.** See "Changed" above.
- **Every save that computed an integrity hash failed with `CantStoreValue`.** `CompressionHandler.hashData()` returned the raw digest from `EncodingService:ComputeStringHash()` as-is, and stored it directly in `DataHash`/`FieldHashes`. Hash digests are effectively random bytes, not guaranteed valid UTF-8, and Roblox DataStores reject any string that isn't valid UTF-8 — so `Save()`/`SavePatch()` would fail on `UpdateAsync` every single time `PersonaStore.EnableDataIntegrityChecks` was on (the default). Fixed by base64-encoding the digest before it's stored or compared, the same approach already used for compressed buffer data.

---

### Configurable Integrity Hashing
- New `PersonaStore.IntegrityMode` setting with three modes:
  - `"Full"` *(default, back-compat)* — rehashes the entire profile on every `SavePatch()`, same as v1.1.x.
  - `"HashOnlyOnFullSave"` — `SavePatch()` skips hashing entirely; `DataHash` only refreshes the next time `Save()` runs.
  - `"PerField"` — `SavePatch()` only rehashes the fields that were actually dirtied, tracked in a new `FieldHashes` table. A profile with a 50,000-item `Inventory` no longer pays to rehash it just because `Coins` changed.
- `PersonaStore:SetIntegrityMode(mode)` to switch modes at runtime.
- `DataSession:VerifyDataIntegrity()` is now integrity-mode-aware, checking `FieldHashes` in `"PerField"` mode instead of the whole-profile hash.
- `Founder:GetKeyMetadata()` now also returns `FieldHashes`.

---

### OrderedDataStore Support
- New `OrderedFounder` class wrapping `DataStoreService:GetOrderedDataStore()`.
- `PersonaStore:CreateOrderedDataStore(storeName)` factory, registered/cached like `CreateDataStore()`.
- `OrderedFounder:Set(key, value)`, `:Get(key)`, `:Increment(key, delta)`, `:Remove(key)`.
- `OrderedFounder:GetSortedPage(ascending, pageSize, minValue, maxValue)` for paginated leaderboard reads, returning both the current page and the native `DataStorePages` object for further pagination.

---

### MemoryStoreService Support
- New `MemoryQueueWrapper` over `MemoryStoreService:GetQueue()`.
  - `PersonaStore:CreateMemoryQueue(name)` factory.
  - `:AddItem(value, expirationSeconds, priority)`, `:ReadItems(count, allOrNothing, waitTimeoutSeconds)` (returns `items, receiptId`), `:RemoveItems(receiptId)`.
- New `MemorySortedMapWrapper` over `MemoryStoreService:GetSortedMap()`.
  - `PersonaStore:CreateMemorySortedMap(name)` factory.
  - `:Set(key, value, expirationSeconds, sortKey)`, `:Get(key)`, `:Remove(key)`, `:GetRange(direction, count, exclusiveLowerBound, exclusiveUpperBound)`, `:Update(key, transformFn, expirationSeconds)`.
- All MemoryStore calls go through the same `withRetry` backoff wrapper as DataStore calls.

---

### DataStore Version APIs
- `Founder:ListVersionsAsync(key, sortDirection, minDate, maxDate, pageSize)` — returns the native `DataStoreVersionPages` object.
- `Founder:GetVersionAsync(key, version)` — returns a deep copy of a historical version, automatically decompressed if that version was stored compressed.
- `Founder:RemoveVersionAsync(key, version)` — permanently deletes a historical version.

---

### DataStore Version API Notes

* `ListVersionsAsync()` relies on Roblox's DataStore version history index, which may be eventually consistent after writes.
* Newly created versions may not appear immediately after a successful `Save()` operation, especially in Studio.
* PersonaStore's internal revision tracking remains authoritative for save ordering and mutation tracking.
* Consumers using version enumeration for administrative tools or recovery workflows should retry `ListVersionsAsync()` when recently-created versions are expected.

> Example Test
> Make a Script in ServerScriptService, place PersonaStore under it, then copy/paste this code.
```lua
-- Manual/integration test harness for PersonaStore.
-- Run in Studio with "Enable Studio Access to API Services" turned on, against a
-- throwaway/staging place. Every store/key is suffixed with a run ID so repeated
-- runs don't collide with leftover data.
--
-- This is NOT a unit test suite (DataStoreService can't be meaningfully mocked
-- without rewriting the module against an injected interface) -- it's a scripted
-- sequence of real calls with assertions, meant to be run manually in Studio and
-- read from the Output window.

local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- path.to.module
local PersonaStore = require(script.PersonaStore)

local runId = tostring(os.time())

local results = {Passed = 0,
	Failed = 0
}

local function check(description: string, condition: boolean, extra: string?)
	if condition then
		results.Passed += 1
		print("[Pass] " .. description)
	else
		results.Failed += 1
		warn("[Fail] " .. description .. (extra and (" -> " .. extra) or ""))
	end
end

local function runSection(name: string, fn: () -> ())
	print("\n=== " .. name .. " ===")
	local ok, err = pcall(fn)
	if not ok then
		results.Failed += 1
		warn("[Fail] Section '" .. name .. "' threw an error -> " .. tostring(err))
	end
end

PersonaStore:Init()

--------------------------------------------------------------------------------
runSection("Basic Save / Load roundtrip", function()
	local Store = PersonaStore:CreateDataStore("Test_Basic_" .. runId, {
		Schema = { Coins = 0, Inventory = {} },
	})

	local key = "player_1"
	local session = Store:LoadSession(key)
	check("LoadSession returns a session", session ~= nil)

	session.Data.Coins = 500
	local saveOk = session:Save()
	check("Save() succeeds", saveOk == true)

	session:Release()

	local snapshot = Store:LoadReadOnlySnapshot(key)
	check("Read-only snapshot reflects saved value", snapshot ~= nil and snapshot.Data.Coins == 500,
		snapshot and ("Coins = " .. tostring(snapshot.Data.Coins)) or "snapshot was nil")
end)

--------------------------------------------------------------------------------
runSection("SavePatch only writes dirty fields", function()
	local Store = PersonaStore:CreateDataStore("Test_Patch_" .. runId, {
		Schema = { Coins = 0, Inventory = { Sword = false } },
	})

	local key = "player_2"
	local session = Store:LoadSession(key)

	session.Data.Coins = 100
	session.Data.Inventory = { Sword = true }
	session:Save()

	-- Now only touch Coins; Inventory should be left alone by SavePatch
	session.Data.Coins = 150
	local patchOk = session:SavePatch()
	check("SavePatch() succeeds", patchOk == true)

	session:Release()

	local snapshot = Store:LoadReadOnlySnapshot(key)

	print(snapshot.FieldHashes)
	print(snapshot.FieldHashes and next(snapshot.FieldHashes))
	
	check("Patched field updated", snapshot.Data.Coins == 150)
	check("Untouched field preserved", snapshot.Data.Inventory.Sword == true)
end)

--------------------------------------------------------------------------------
runSection("Compression + algorithm-mismatch bug fix", function()
	local Store = PersonaStore:CreateDataStore("Test_Compression_" .. runId, {
		Schema = { Coins = 0, Inventory = {} },
	})

	local key = "player_3"
	
	local session = Store:LoadSession(key)
	
	session.Data.Coins = 777
	session.Data.Inventory = {
		Potion = 3
	}

	session:SaveCompressed()
	session:Release()

	local session2 = Store:LoadSession(key)

	check(
		"Compressed profile loads correctly",
		session2
			and session2.Data.Coins == 777
			and session2.Data.Inventory.Potion == 3
	)

	session2:Release()
end)

--------------------------------------------------------------------------------
runSection("VerifyDataIntegrity bug fix (Full mode)", function()
	PersonaStore:SetIntegrityMode("Full")

	local Store = PersonaStore:CreateDataStore("Test_Integrity_Full_" .. runId, {
		Schema = { Coins = 0 },
	})

	local key = "player_4"
	local session = Store:LoadSession(key)
	session.Data.Coins = 42
	session:Save()

	-- Pre-fix, this reliably returned false even with untampered data.
	check("VerifyDataIntegrity() returns true for untampered data", session:VerifyDataIntegrity() == true)

	session:Release()
end)

--------------------------------------------------------------------------------
runSection("PerField integrity mode", function()
	PersonaStore:SetIntegrityMode("PerField")

	local Store = PersonaStore:CreateDataStore("Test_Integrity_PerField_" .. runId, {
		Schema = { Coins = 0, Inventory = {} },
	})

	local key = "player_5"
	local session = Store:LoadSession(key)
	session.Data.Coins = 10
	session.Data.Inventory = { ItemA = true }
	session:Save() -- establishes baseline DataHash + FieldHashes

	-- Only touch Coins via SavePatch; Inventory's field hash should be untouched.
	session.Data.Coins = 20
	session:SavePatch()

	check("VerifyDataIntegrity() (PerField) returns true after a partial patch",
		session:VerifyDataIntegrity() == true)

	session:Release()

	local meta = Store:GetKeyMetadata(key)
	check("FieldHashes present in PerField mode", meta ~= nil and meta.Integrity.FieldHashes ~= nil)

	PersonaStore:SetIntegrityMode("Full")
end)

--------------------------------------------------------------------------------
runSection("BatchUpdate double-save bug fix", function()
	local Store = PersonaStore:CreateDataStore("Test_Batch_" .. runId, {
		Schema = { Coins = 0 },
	})

	local keys = { "batch_1", "batch_2" }

	local results1 = Store:BatchUpdate(keys, function(data)
		data.Coins = 5
	end)
	check("BatchUpdate reports success for all keys",
		results1["batch_1"] == true and results1["batch_2"] == true)

	local meta = Store:GetKeyMetadata("batch_1")
	-- A brand-new key starts at Version 0; BatchUpdate should bump it to exactly 1
	-- via a single Destroy() -> Save(). Pre-fix, this would be 2 (SavePatch + Save).
	check(
		"Revision incremented exactly once",
		meta.Revision == 1,
		"Revision = " .. tostring(meta.Revision)
	)
end)

--------------------------------------------------------------------------------
runSection("OrderedDataStore wrapper", function()
	local Leaderboard = PersonaStore:CreateOrderedDataStore("Test_Ordered_" .. runId)

	local setOk = Leaderboard:Set("player_1", 100)
	check("Set() succeeds", setOk == true)

	local value = Leaderboard:Get("player_1")
	check("Get() returns the set value", value == 100)

	local newValue = Leaderboard:Increment("player_1", 25)
	check("Increment() returns updated value", newValue == 125)

	Leaderboard:Set("player_2", 50)
	local page, pages = Leaderboard:GetSortedPage(false, 10)
	check("GetSortedPage() returns entries", page ~= nil and #page >= 2, pages and "pages returned" or "pages was nil")

	Leaderboard:Remove("player_1")
	check("Remove() clears the value", Leaderboard:Get("player_1") == nil)
end)

--------------------------------------------------------------------------------
runSection("MemoryStore queue wrapper", function()
	local Queue = PersonaStore:CreateMemoryQueue("Test_Queue_" .. runId)

	local addOk = Queue:AddItem({ Hello = "World" }, 60)
	check("AddItem() succeeds", addOk == true)

	local items, receiptId = Queue:ReadItems(1, false, 5)
	check("ReadItems() returns the item", items ~= nil and #items == 1 and receiptId ~= nil)

	if receiptId then
		local removeOk = Queue:RemoveItems(receiptId)
		check("RemoveItems() succeeds", removeOk == true)
	end
end)

--------------------------------------------------------------------------------
runSection("MemoryStore sorted map wrapper", function()
	local ActiveMatches = PersonaStore:CreateMemorySortedMap("Test_SortedMap_" .. runId)

	ActiveMatches:Set("match_1", { Players = 2 }, 60)
	local match = ActiveMatches:Get("match_1")
	check("Set()/Get() roundtrip", match ~= nil and match.Players == 2)

	ActiveMatches:Update("match_1", function(old)
		old.Players += 1
		return old
	end)
	check("Update() applies transform", ActiveMatches:Get("match_1").Players == 3)

	ActiveMatches:Remove("match_1")
	check("Remove() clears the entry", ActiveMatches:Get("match_1") == nil)
end)

--------------------------------------------------------------------------------
runSection("Version APIs", function()
	local Store = PersonaStore:CreateDataStore("Test_Versions_" .. runId, {
		Schema = { Coins = 0 },
	})

	local key = "player_versions"

	local s1 = Store:LoadSession(key)
	s1.Data.Coins = 1
	s1:Save()

	s1:Release()

	local s2 = Store:LoadSession(key)
	s2.Data.Coins = 2
	s2:Save()
	
	local pages

	for i = 1, 12 do
		pages = Store:ListVersionsAsync(key, Enum.SortDirection.Descending)

		local count = #pages:GetCurrentPage()

		if count >= 2 then
			break
		end

		task.wait(5)
	end

	check(
		"At least 2 versions exist",
		pages ~= nil and #pages:GetCurrentPage() >= 2,
		"found " .. (#pages:GetCurrentPage())
	)

	if pages then
		local currentPage = pages:GetCurrentPage()

		print("Entries:", #currentPage)

		for i, v in ipairs(currentPage) do
			print(i)
			print("Version:", v.Version)
			print("CreatedTime:", v.CreatedTime)
			print("IsDeleted:", v.IsDeleted)
		end

		if #currentPage >= 2 then
			local olderVersion = currentPage[2].Version -- second-most-recent
			local olderData = Store:GetVersionAsync(key, olderVersion)
			check("GetVersionAsync() returns the older Coins value",
				olderData ~= nil and olderData.Data.Coins == 1,
				olderData and ("Coins=" .. tostring(olderData.Data.Coins)) or "olderData was nil")

			local removeOk = Store:RemoveVersionAsync(key, olderVersion)
			check("RemoveVersionAsync() succeeds", removeOk == true)
		end
	end
end)

--------------------------------------------------------------------------------
--// NOTE: This test will ALWAYS get 23/24 passes in a studio session
--// This is because of delay when running 'Store:ListVersionsAsync(key)'
-- // as Roblox's version history endpoint is not guaranteed to be
--// immediately consistent after writes.
print(("\n=== Results: %d Passed, %d Failed ==="):format(results.Passed, results.Failed))
	-- === Results: 23 passed, 1 failed ===  -  Server - DataManager:319	
```

---

### Migration Notes for v1.1.0 → v1.2.0
**100% Backward Compatible** — no breaking changes. `PersonaStore.IntegrityMode` defaults to `"Full"`, the same hashing behavior as v1.1.x, so nothing changes unless you opt in:

```lua
-- v1.1.0 code continues to work exactly as before
session:SavePatch()

-- Opt in to cheaper patch-hashing when you have large, mostly-static fields
PersonaStore:SetIntegrityMode("PerField")

-- New OrderedDataStore / MemoryStore / Version APIs are additive
local Leaderboard = PersonaStore:CreateOrderedDataStore("Leaderboard_v1")
local Queue = PersonaStore:CreateMemoryQueue("PurchaseQueue")
local versions = PlayerStore:ListVersionsAsync(tostring(userId))
```

If you were previously relying on `VerifyDataIntegrity()` returning `false` in normal operation (i.e. code paths that treated "always fails" as expected), double check those paths — it now correctly returns `true` when data hasn't been tampered with.

---

# v1.1.0

## Added
*EncodingService Integration & Advanced Features*

### Core Compression Features
- Native Roblox `EncodingService` integration for efficient data compression.
- `DataSession:SaveCompressed()` method for bandwidth-optimized saves.
- Support for multiple compression algorithms: `Deflate` and `ZSTD`.
- Configurable compression levels (1-22 depending on algorithm).
- Compression metadata tracking (algorithm, original size, compressed size).
- Automatic compression cache to avoid recompressing unchanged data.
- `PersonaStore:SetCompressionSettings(algorithm, level)` for global configuration.

### Data Integrity & Security
- Cryptographic hash verification using `EncodingService:ComputeStringHash()`.
- Support for multiple hash algorithms: `SHA256` (default), `SHA1`, `MD5`.
- `DataSession:VerifyDataIntegrity()` method to detect data corruption.
- Automatic hash computation on every `Save()` and `SaveCompressed()`.
- Configurable integrity checking via `PersonaStore.EnableDataIntegrityChecks`.

### Read-Only Access
- `Founder:LoadReadOnlySession(key)` for lock-free profile queries.
- Perfect for leaderboards, admin panels, and data aggregation.
- Zero lock conflict overhead.
- Full profile data access without ownership constraints.

### Atomic Operations
- `DataSession:IncrementCounter(fieldName, amount)` for thread-safe increments.
- Default increment of 1 if amount not specified.
- Automatic transaction handling and saving.
- Ideal for leaderboards, currency systems, and statistics.

### Batch Operations
- `Founder:BatchUpdate(keys, transformFn)` for mass profile updates.
- Atomic updates with automatic lock management per profile.
- Returns success/failure status per key.
- Useful for seasonal resets, migrations, and event-based updates.

### Data Export & Import
- `DataSession:ExportData(includeMetadata)` for profile backup as JSON.
- `DataSession:ImportData(jsonString, overwrite)` to restore from backup.
- Optional metadata inclusion for audit trails.
- Merge mode (append) or overwrite mode for imports.

### Profile Metadata Queries
- `Founder:GetKeyMetadata(key)` to query profile info without loading.
- Returns version, last update time, session token, JobId, hash, and compression info.
- Efficient diagnostics and monitoring without lock acquisition.

### Statistics & Monitoring
- `PersonaStore:GetStatistics()` for engine-wide metrics.
- Tracks total saves, loads, compressions, decompressions.
- DataStore request attempt/failure counts.
- Total bytes saved through compression.
- `Founder:GetCompressionStats()` for store-specific compression data.
- Real-time performance monitoring for infrastructure teams.

### Enhanced Session Metadata
- Expanded `GetPerformanceMetadata()` with compression statistics.
- Tracks last compressed/uncompressed sizes per session.
- More detailed session lifecycle information.

### Base64 Encoding Support
- Built-in Base64 encoding/decoding for compressed data.
- Seamless integration with JSON-safe DataStore format.
- Automatic handling via `CompressionHandler`.

## Changed
- Improved retry logic with better exponential backoff calculation.
- Enhanced error messages for clearer debugging.
- `DataSession` now invalidates compression cache on mutations.
- Profile structure now includes `DataHash` and `CompressionMetadata` fields.
- `PersonaStore` now tracks comprehensive statistics.

## Fixed
- Retry mechanism now properly tracks DataStore request failures.

## Removed
- N/A

## Performance

- Automatic serialization only processes fields listed in the SerializationManifest.
- Unlisted fields bypass the serializer completely.
- BufferArray stores numeric collections in compact binary form, reducing DataStore payload sizes.
- Serialization occurs immediately before hashing/compression, avoiding unnecessary work during gameplay.

### Migration Notes for v1.0.0 → v1.1.0
**100% Backward Compatible** - No breaking changes. Existing code continues to work without modification.

New features are opt-in and can be adopted gradually:
```lua
-- v1.0.0 code continues to work
session:SavePatch()

-- New v1.1.0 features available when needed
session:SaveCompressed()  -- New compression
session:VerifyDataIntegrity()  -- New integrity check
```

---

## Versioning Policy

- **MAJOR**: Breaking API changes or complete rewrites
- **MINOR**: New features that are backward compatible
- **PATCH**: Bug fixes and performance improvements

---
