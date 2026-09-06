<img width="75" height="75" alt="image" src="https://github.com/user-attachments/assets/ab117cd3-8c55-419e-97e0-2d9d1b25c59b" />  

# PersonaStore

> A comprehensive Roblox data persistence framework providing session locking, transactional saves, observable data, compression, integrity verification, DataStore versioning, OrderedDataStore utilities, and MemoryStore abstractions through a fully object-oriented API and Luau type checking.

---

## Features

-  Session locking with automatic ownership validation
-  Automatic periodic patch saves
-  Selective data patching (only changed fields are written)
-  Deep observable proxy tree for nested table tracking
-  Cross-server global update system
-  MessagingService-powered cluster synchronization
-  Transaction support with rollback protection
-  Automatic schema reconciliation
-  Retry layer with exponential backoff + jitter
-  Runtime metadata & diagnostics
-  Automatic BindToClose cleanup
-  Data Serialization
-  Buffer Utility
-  Multiple isolated DataStores through Founder objects

---

# Installation

Place `PersonaStore.lua` inside **ServerScriptService** or another server-only location.

```lua
local PersonaStore = require(path.To.PersonaStore)
```

Initialize the engine once during server startup.

```lua
local PersonaStore = require(ServerStorage.PersonaStore)

PersonaStore:Init()
```

---

# Quick Start

Create a schema.

```lua
local PlayerTemplate = {
    Cash = 0,
    Gems = 0,

    Inventory = {},

    Stats = {
        Level = 1,
        XP = 0
    }
}
```

Create a DataStore.

```lua
local PlayerStore = PersonaStore:CreateDataStore("PlayerData", {
    Schema = PlayerTemplate
})
```

Load a session.

```lua
local session = PlayerStore:LoadSession(player.UserId)

if not session then
    player:Kick("Failed to load data.")
    return
end
```

Access data normally.

```lua
session.Data.Cash += 100

session.Data.Stats.Level += 1
```

Destroy the session when finished.

```lua
session:Destroy()
```

---

# Architecture

PersonaStore is composed of three primary objects.

| Object | Purpose |
|---------|----------|
| **PersonaStore** | Global engine responsible for initialization, routing, shutdown handling, and DataStore creation. |
| **Founder** | Represents a single isolated DataStore and manages session ownership. |
| **DataSession** | Active profile wrapper used for reading, writing, saving, and listening to data changes. |

---

# Example

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local PersonaStore = require(ReplicatedStorage.Packages.PersonaStore)
PersonaStore:Init()

local Template = {
	Money = 0,
	Kills = 0,
}

local MoneyStore = PersonaStore:CreateDataStore("MoneyStats_v1", {
	Schema = Template
})

local sessions = {}

Players.PlayerAdded:Connect(function(player)
	local session = MoneyStore:LoadSessionAsync(tostring(player.UserId), 10)
	if not session then
		player:Kick("Failed to load session your session. Please rejoin.")
		return
	end
	
    local leaderstats = Instance.new("Folder", player)
	leaderstats.Name = "leaderstats"
	
    local money = Instance.new("IntValue", leaderstats)
    money.Name = "Money"
	money.Value = session.Data.Money
	
    local kills = Instance.new("IntValue", leaderstats)
    kills.Name = "Kills"
	kills.Value = session.Data.Kills
	
	sessions[player.UserId] = session
	
	session:ListenToFieldChange(function(_, new, root)
		if root == "Kills" then
			kills.Value = new
		end
		
		if root == "Money" then
			money.Value = new
		end
	end)
end)

Players.PlayerRemoving:Connect(function(player)
	local session = sessions[player.UserId]
	if session then
		session:Destroy()
	end
	
	sessions[player.UserId] = nil
end)
```

---

# Why PersonaStore?

Roblox's DataStoreService is intentionally low-level.

PersonaStore builds on top of it by providing:

- Session ownership
- Automatic saving
- Observable data
- Transactions
- Patch saving
- Global updates
- Retry handling
- Cross-server synchronization
- Schema reconciliation

so developers can spend more time building games rather than persistence infrastructure.

---

### PersonaStore is built with time, effort, love, and passion.

---

**Author:** @Draxxor   
**Version:** 2.0.0   
**License:** MIT   

Made for developers by a developer. ❤
