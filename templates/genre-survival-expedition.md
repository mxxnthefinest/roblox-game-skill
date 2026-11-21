# Survival & Expedition (Central Anchor Co-op) Genre Template

> **Load this template when:** Building a co-op survival, base-defense, or expedition game modeled after *99 Nights in the Forest* (static expanding campfire base + night survival) or *Drill to Earth's Core* (mobile advancing drill vehicle + layered underground exploration).
>
> **Requires:** Always apply `references/modular-service-architecture.md` (`--!strict`, `Init()`/`Start()` ModuleScripts, `EventBus`, `RemoteGateway`, 3-layer data separation) and `templates/game-scaffold.md`.

---

## 1. Genre Blueprint & Core Loop

Both archetypes share the **"Central Anchor + Fuel Bottleneck + Layered Risk-Reward Exploration + Co-op Defense"** loop:

- **Static Anchor Mode (`Survival99`):** Expanding Campfire safe zone (`Level 1..6`), `180s` Day / `90s` Night rhythmic cycle, wood/scrap/pelts/corpses, Crafting Bench tiers (`1..5`), Beds + rescued kids (`DayMultiplier` `1x..9x`), every-3-night Cultist raids + Stalker threat (`The Deer`) stunned by Flashlight cone.
- **Mobile Anchor Mode (`DrillExpedition` / `InfiniteMode`):** Server-owned Drill vehicle advancing through layers (`Grass` → `Dirt` → `Stone` → `Crystal` → `Frozen` → `100 km Glowing Core`), Little/Large/Boss Walls (`BossWalls` lock teleporters & close red return wall behind players), welded turrets (`Sentry`/`Gunner`) & saboteurs (`Fuel Leech`, `Bombo`), coin economy + 3-pick Drill Upgrades.

```text
Explore & Gather (Micro: 1-30s)
   │
   ▼  [Sack full OR night/wall threshold]
Return to Central Anchor (Campfire / Drill)
   ├──► Feed Fuel (Survival)
   └──► Craft / Sell / Upgrade (Progression)
   │
   ▼
Defend Night / Breach Boss Wall (Meso: 3-10m)
   ├──► Fuel == 0 / Wipe ──► SafeZone Collapses ──► Partial Meta Rewards (Lobby)
   └──► Victory ──► Unlock Layer / Advance Day ──► 99th Day / 100km Core -> Infinite Mode
```

---

## 2. Strict Shared Data & Network Contracts

### 2.1 `ReplicatedStorage/Shared/Types/DataTypes.luau`

```luau
--!strict
-- ReplicatedStorage/Shared/Types/DataTypes.luau

export type ClassProgress = {
	Level: number,
	QuestProgress: { [string]: number },
}

export type PlayerProfile = {
	SchemaVersion: number,
	Meta: {
		FirstJoinTimestamp: number,
		LastLoginTimestamp: number,
		TotalPlaytimeSeconds: number,
	},
	Currencies: {
		Diamonds: number,
		Cores: number,
	},
	Records: {
		MaxNightsSurvived: number,
		Fastest99NightsSeconds: number?,
		MaxDepthMeters: number,
		TotalDepthMeters: number,
		TotalFuelContributed: number,
		TotalBossesDefeated: number,
	},
	UnlockedRegions: { string },
	Classes: {
		EquippedClassId: string,
		Owned: { [string]: ClassProgress },
	},
	PreRunInventory: {
		OwnedBoosts: { string },
		Cosmetics: { string },
	},
	Achievements: { string },
	Settings: {
		MusicVolume: number,
		SFXVolume: number,
		LowGraphicsMode: boolean,
	},
}

-- Ephemeral in-run state: NEVER persisted to DataStoreService.
-- Checkpointed to MemoryStoreService for mid-run rejoin recovery.
export type RunSessionState = {
	SessionId: string,
	Mode: "Survival99" | "DrillExpedition" | "InfiniteMode",
	CurrentDay: number,
	DayMultiplier: number, -- 1..9
	Phase: "Day" | "Night",
	PhaseTimeRemaining: number,
	CurrentDepthMeters: number,
	CurrentLayerId: string,
	Anchor: {
		Level: number,
		Fuel: number,
		MaxFuel: number,
		SafeZoneRadius: number,
		DrillPower: number,
		DrillDefense: number,
		IsBossWallActive: boolean,
	},
	TeamCoins: number,
	CraftingBenchLevel: number, -- 1..5
	RescuedChildrenCount: number, -- 0..4
	BedsBuilt: number,
	ActiveWeatherEvent: string?,
}

export type SackItem = {
	InstanceUUID: string,
	ItemId: string,
	FuelValue: number,
	SellValue: number,
	Weight: number,
}

export type OfferData = {
	OfferId: string,
	Title: string,
	Description: string,
	Rarity: "Common" | "Rare" | "Epic" | "Mythic",
	RequiredItemId: string?,
	CostCoins: number?,
}

return {}
```

### 2.2 `ReplicatedStorage/Shared/Config/ProgressionMath.luau`

Pure, deterministic scaling functions shared by Server (authoritative evaluation) and Client (UI cost previews):

```luau
--!strict
-- ReplicatedStorage/Shared/Config/ProgressionMath.luau

local ProgressionMath = {}

function ProgressionMath.calculateUpgradeCost(baseCost: number, growthRate: number, level: number): number
	local safeLevel = math.max(1, math.floor(level))
	return math.floor(baseCost * (growthRate ^ (safeLevel - 1)))
end

function ProgressionMath.calculateDrillDamage(baseDamage: number, scaleFactor: number, level: number): number
	local safeLevel = math.max(1, math.floor(level))
	return baseDamage + scaleFactor * (safeLevel - 1) * (1 + 0.08 * safeLevel)
end

function ProgressionMath.calculateHarvestSpeed(baseSpeed: number, bonusScale: number, level: number): number
	local safeLevel = math.max(1, math.floor(level))
	return baseSpeed + bonusScale * math.log(1 + 0.5 * safeLevel)
end

-- Sub-linear party scaling: 4 players ~= 2.27x difficulty (keeps co-op rewarding)
function ProgressionMath.calculatePartyMultiplier(partySize: number): number
	local n = math.max(1, math.floor(partySize))
	if n == 1 then
		return 1.0
	end
	return 1.0 + 0.5 * ((n - 1) ^ 0.85)
end

-- Compound polynomial-exponential Boss Wall HP vs linear Infinite sell mult
function ProgressionMath.calculateBossWallHP(baseHP: number, depthKm: number): number
	local d = math.max(0, depthKm)
	return math.floor(baseHP * ((1 + d / 10) ^ 1.65) * (1.12 ^ math.floor(d / 25)))
end

function ProgressionMath.calculateInfiniteSellMultiplier(depthKm: number): number
	local d = math.max(0, depthKm)
	if d < 100 then
		return 1.0
	end
	return 1.2 + 0.2 * math.floor(d / 100)
end

function ProgressionMath.calculateRaidCount(raidIndex: number, partySize: number): number
	local k = math.max(1, math.floor(raidIndex))
	local base = math.floor(3 + 1.5 * k)
	return math.floor(base * ProgressionMath.calculatePartyMultiplier(partySize))
end

function ProgressionMath.calculateStunDuration(baseStun: number, minStun: number, stunCountInNight: number): number
	local n = math.max(1, math.floor(stunCountInNight))
	return math.max(minStun, baseStun * (0.82 ^ (n - 1)))
end

function ProgressionMath.calculateDayMultiplier(bedsBuilt: number, rescuedChildren: number): number
	return math.clamp(1 + math.max(0, bedsBuilt) + math.max(0, rescuedChildren), 1, 9)
end

return ProgressionMath
```

---

## 3. RemoteEvent & Rate-Limit Contract

Register all remotes in `RemoteGateway` during `Init()`:

| Remote | Direction | Rate Policy | Server Validator Requirements |
|---|---|---|---|
| `RF_InitialSync` | C → S | 1/s, 2s cd | Initial join snapshot ONLY (profile summary + current `RunSessionState`). |
| `RE_ActionRequest` | C → S | 12/s | `actionType` in `{"Mine","Attack","Interact","Flashlight"}`, finite `hitPos`, `(rootPos - targetPos).Magnitude <= range + 4`, tool ownership. |
| `RE_InventoryAction` | C → S | 6/s | `op` in `{"Pickup","Drop","DropAll","Equip","DepositFuel","Weld"}`, `InstanceUUID` ownership, capacity / weld distance check. |
| `RE_CraftOrUpgrade` | C → S | 3/s | `stationType` in `{"Bench","DrillBoard","PeltTrader","Shop"}`, whitelist recipe/card ID, atomic currency/item deduction. |
| `RE_VehicleControl` | C → S | 10/s | `throttle` finite in `[-1, 1]`, `action` in `{"Drive","Stop","ResetDrill"}`, verify player sits in driver `VehicleSeat`. |
| `RE_StateSnapshot` | S → C | 5 Hz (0.2s) | Delta table of changed `RunSessionState` & player vitals fields only. |
| `RE_WorldFeedback` | S → C | Event-driven | Scoped to players within `250 studs` (`BroadcastWorldFeedbackNear`). |
| `RE_OfferPrompt` | S → C | Event-driven | Per-player personalized vendor / 3-pick Drill Upgrade offers. |

---

## 4. Core Service Implementations (`--!strict`)

### 4.1 `CycleAndVitalsService.luau`

Handles the `180s` Day / `90s` Night cycle, party-scaled + leech anchor fuel drain, Fuel=0 SafeZone collapse, player Hunger/Freeze/Health tick, and 5 Hz `RE_StateSnapshot` delta broadcasting.

```luau
--!strict
-- ServerScriptService/Services/CycleAndVitalsService.luau

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local DataTypes = require(ReplicatedStorage.Shared.Types.DataTypes)
local ProgressionMath = require(ReplicatedStorage.Shared.Config.ProgressionMath)
local Trove = require(ReplicatedStorage.Shared.Utils.Trove)
local EventBus = require(ServerScriptService.Core.EventBus)
local RemoteGateway = require(ServerScriptService.Core.RemoteGateway)

type PlayerVitals = {
	Hunger: number,
	Temperature: number,
	InSafeZone: boolean,
}

local DAY_DURATION = 180
local NIGHT_DURATION = 90
local SNAPSHOT_INTERVAL = 0.2 -- 5 Hz
local VITALS_TICK_INTERVAL = 1.0
local BASE_FUEL_DRAIN_PER_SEC = 0.45

local CycleAndVitalsService = {}

local serviceTrove = Trove.new()
local playerVitals: { [Player]: PlayerVitals } = {}
local anchorPivotPos: Vector3 = Vector3.zero
local attachedLeeches = 0
local snapshotAcc = 0
local vitalsAcc = 0

local runState: DataTypes.RunSessionState = {
	SessionId = "run_default",
	Mode = "Survival99",
	CurrentDay = 1,
	DayMultiplier = 1,
	Phase = "Day",
	PhaseTimeRemaining = DAY_DURATION,
	CurrentDepthMeters = 0,
	CurrentLayerId = "GrassLayer",
	Anchor = {
		Level = 1,
		Fuel = 100,
		MaxFuel = 100,
		SafeZoneRadius = 45,
		DrillPower = 10,
		DrillDefense = 5,
		IsBossWallActive = false,
	},
	TeamCoins = 0,
	CraftingBenchLevel = 1,
	RescuedChildrenCount = 0,
	BedsBuilt = 0,
	ActiveWeatherEvent = nil,
}

local function getWeatherMultiplier(): number
	if runState.ActiveWeatherEvent == "Blizzard" then
		return 1.5
	elseif runState.ActiveWeatherEvent == "Rain" then
		return 1.35
	end
	return 1.0
end

local function syncLightingForPhase(phase: "Day" | "Night")
	Lighting.ClockTime = if phase == "Day" then 13 else 0
end

local function stepPhase(dt: number)
	runState.PhaseTimeRemaining -= dt
	if runState.PhaseTimeRemaining > 0 then
		return
	end

	if runState.Phase == "Day" then
		runState.Phase = "Night"
		runState.PhaseTimeRemaining = NIGHT_DURATION
	else
		runState.Phase = "Day"
		runState.PhaseTimeRemaining = DAY_DURATION
		runState.CurrentDay += runState.DayMultiplier
	end

	syncLightingForPhase(runState.Phase)
	EventBus.BE_PhaseChanged:Fire({
		newPhase = runState.Phase,
		dayNumber = runState.CurrentDay,
	})
end

local function stepAnchorFuel(dt: number)
	if runState.Anchor.Fuel <= 0 then
		return
	end
	local partyCount = math.max(1, #Players:GetPlayers())
	local drainRate = BASE_FUEL_DRAIN_PER_SEC
		* getWeatherMultiplier()
		* ProgressionMath.calculatePartyMultiplier(partyCount)
	local leechDrain = attachedLeeches * 1.0

	runState.Anchor.Fuel = math.max(0, runState.Anchor.Fuel - (drainRate + leechDrain) * dt)

	if runState.Anchor.Fuel == 0 then
		runState.Anchor.SafeZoneRadius = 0
		EventBus.BE_AnchorFuelState:Fire({
			state = "Extinguished",
			level = runState.Anchor.Level,
		})
	end
end

local function stepPlayerVitals()
	for _, player in Players:GetPlayers() do
		local vitals = playerVitals[player]
		local char = player.Character
		local root = char and char:FindFirstChild("HumanoidRootPart")
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not vitals or not (root and root:IsA("BasePart")) or not hum or hum.Health <= 0 then
			continue
		end

		local dist = (root.Position - anchorPivotPos).Magnitude
		vitals.InSafeZone = runState.Anchor.Fuel > 0 and dist <= runState.Anchor.SafeZoneRadius
		vitals.Hunger = math.max(0, vitals.Hunger - 0.35)

		if vitals.InSafeZone then
			vitals.Temperature = math.min(100, vitals.Temperature + 4.0)
			if vitals.Hunger > 20 then
				hum.Health = math.min(hum.MaxHealth, hum.Health + 0.5)
			end
		elseif runState.ActiveWeatherEvent == "Blizzard" then
			vitals.Temperature = math.max(0, vitals.Temperature - 2.5)
		end

		if vitals.Hunger == 0 or vitals.Temperature == 0 then
			hum:TakeDamage(4)
		end
	end
end

function CycleAndVitalsService.AddAnchorFuel(amount: number): boolean
	if amount <= 0 or amount ~= amount then
		return false
	end
	local wasEmpty = runState.Anchor.Fuel <= 0
	runState.Anchor.Fuel = math.min(runState.Anchor.MaxFuel, runState.Anchor.Fuel + amount)
	if wasEmpty and runState.Anchor.Fuel > 0 then
		runState.Anchor.SafeZoneRadius = 40 + runState.Anchor.Level * 10
		EventBus.BE_AnchorFuelState:Fire({
			state = "Relit",
			level = runState.Anchor.Level,
		})
	end
	return true
end

function CycleAndVitalsService.GetRunState(): DataTypes.RunSessionState
	return runState
end

function CycleAndVitalsService.SetAnchorPosition(pos: Vector3)
	anchorPivotPos = pos
end

function CycleAndVitalsService.SetAttachedLeechCount(count: number)
	attachedLeeches = math.max(0, math.floor(count))
end

function CycleAndVitalsService.Init()
	syncLightingForPhase("Day")
end

function CycleAndVitalsService.Start()
	local function onPlayerAdded(player: Player)
		playerVitals[player] = {
			Hunger = 100,
			Temperature = 100,
			InSafeZone = true,
		}
	end

	serviceTrove:Connect(Players.PlayerAdded, onPlayerAdded)
	serviceTrove:Connect(Players.PlayerRemoving, function(player: Player)
		playerVitals[player] = nil
	end)
	for _, p in Players:GetPlayers() do
		onPlayerAdded(p)
	end

	serviceTrove:Connect(RunService.Heartbeat, function(dt: number)
		stepPhase(dt)
		stepAnchorFuel(dt)

		vitalsAcc += dt
		if vitalsAcc >= VITALS_TICK_INTERVAL then
			vitalsAcc -= VITALS_TICK_INTERVAL
			stepPlayerVitals()
		end

		snapshotAcc += dt
		if snapshotAcc >= SNAPSHOT_INTERVAL then
			snapshotAcc -= SNAPSHOT_INTERVAL
			for _, player in Players:GetPlayers() do
				RemoteGateway.FireClient("RE_StateSnapshot", player, {
					day = runState.CurrentDay,
					phase = runState.Phase,
					timeLeft = math.ceil(runState.PhaseTimeRemaining),
					fuel = math.floor(runState.Anchor.Fuel),
					maxFuel = runState.Anchor.MaxFuel,
					depth = math.floor(runState.CurrentDepthMeters),
					coins = runState.TeamCoins,
					vitals = playerVitals[player],
				})
			end
		end
	end)
end

return CycleAndVitalsService
```

### 4.2 `GatheringCraftingService.luau` (UUID Anti-Dupe + Range/Cooldown/Mutation Auth)

```luau
--!strict
-- ServerScriptService/Services/GatheringCraftingService.luau

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local CollectionService = game:GetService("CollectionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local DataTypes = require(ReplicatedStorage.Shared.Types.DataTypes)
local TypeGuards = require(ReplicatedStorage.Shared.Utils.TypeGuards)
local Trove = require(ReplicatedStorage.Shared.Utils.Trove)
local RemoteGateway = require(ServerScriptService.Core.RemoteGateway)

local MAX_MINE_RANGE = 16
local RANGE_TOLERANCE = 4
local DEFAULT_SACK_CAPACITY = 8

local GatheringCraftingService = {}

local serviceTrove = Trove.new()

local playerSacks: { [Player]: { DataTypes.SackItem } } = {}
local playerCapacities: { [Player]: number } = {}
local nodeHealth: { [string]: number } = {}
local activeItemLocks: { [string]: boolean } = {}

local function getNodeById(nodeId: string): Model?
	for _, inst in CollectionService:GetTagged("ResourceNode") do
		if inst:IsA("Model") and inst:GetAttribute("NodeId") == nodeId then
			return inst
		end
	end
	return nil
end

local function isToolCompatible(toolType: string, mutationType: string?): boolean
	if mutationType == "Stone" then
		return toolType == "Pickaxe" or toolType == "Explosive"
	end
	return true
end

local function handleMineRequest(player: Player, targetId: string)
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	local tool = char and char:FindFirstChildOfClass("Tool")
	if not (root and root:IsA("BasePart")) or not tool then
		return
	end

	local node = getNodeById(targetId)
	if not node then
		return
	end

	local nodePos = node:GetPivot().Position
	if (root.Position - nodePos).Magnitude > (MAX_MINE_RANGE + RANGE_TOLERANCE) then
		return
	end

	local toolType = (tool:GetAttribute("ToolType") :: string?) or "Axe"
	local mutation = node:GetAttribute("Mutation") :: string?
	if not isToolCompatible(toolType, mutation) then
		RemoteGateway.BroadcastWorldFeedbackNear(nodePos, 250, "Spark", {
			reason = "MutationImmune",
			nodeId = targetId,
		})
		return
	end

	local hp = nodeHealth[targetId] or (node:GetAttribute("MaxHP") :: number?) or 100
	local damage = (tool:GetAttribute("HarvestDamage") :: number?) or 25
	hp -= damage
	nodeHealth[targetId] = hp

	RemoteGateway.BroadcastWorldFeedbackNear(nodePos, 250, "HitMarker", {
		nodeId = targetId,
		remainingHP = math.max(0, hp),
	})

	if hp <= 0 then
		nodeHealth[targetId] = nil
		local dropId = (node:GetAttribute("DropItemId") :: string?) or "Wood"
		GatheringCraftingService.TryAddToSack(player, {
			InstanceUUID = HttpService:GenerateGUID(false),
			ItemId = dropId,
			FuelValue = (node:GetAttribute("DropFuelValue") :: number?) or 8,
			SellValue = (node:GetAttribute("DropSellValue") :: number?) or 5,
			Weight = 1,
		})
		node:Destroy()
	end
end

function GatheringCraftingService.TryAddToSack(player: Player, item: DataTypes.SackItem): boolean
	local sack = playerSacks[player]
	local cap = playerCapacities[player] or DEFAULT_SACK_CAPACITY
	if not sack or #sack >= cap or activeItemLocks[item.InstanceUUID] then
		return false
	end
	activeItemLocks[item.InstanceUUID] = true
	table.insert(sack, item)
	activeItemLocks[item.InstanceUUID] = nil
	return true
end

function GatheringCraftingService.PopItemByUUID(player: Player, uuid: string): DataTypes.SackItem?
	local sack = playerSacks[player]
	if not sack or activeItemLocks[uuid] then
		return nil
	end
	activeItemLocks[uuid] = true
	for idx, item in sack do
		if item.InstanceUUID == uuid then
			table.remove(sack, idx)
			activeItemLocks[uuid] = nil
			return item
		end
	end
	activeItemLocks[uuid] = nil
	return nil
end

function GatheringCraftingService.Init()
	RemoteGateway.RegisterEvent({
		name = "RE_ActionRequest",
		policy = { maxPerSecond = 12, cooldownSeconds = 0.08, kickAfterViolations = 8 },
		validator = function(_player: Player, actionType: unknown, targetId: unknown, hitPos: unknown): (boolean, string?)
			if not TypeGuards.isOneOf(actionType, { "Mine", "Attack", "Interact", "Flashlight" }) then
				return false, "Invalid actionType"
			end
			if not TypeGuards.isNonEmptyString(targetId, 64) then
				return false, "Invalid targetId"
			end
			if hitPos ~= nil and not TypeGuards.isFiniteVector3(hitPos) then
				return false, "Invalid hitPos"
			end
			return true, nil
		end,
		handler = function(player: Player, actionType: unknown, targetId: unknown)
			if actionType == "Mine" then
				handleMineRequest(player, targetId :: string)
			end
		end,
	})
end

function GatheringCraftingService.Start()
	serviceTrove:Connect(Players.PlayerAdded, function(p: Player)
		playerSacks[p] = {}
		playerCapacities[p] = DEFAULT_SACK_CAPACITY
	end)
	serviceTrove:Connect(Players.PlayerRemoving, function(p: Player)
		playerSacks[p] = nil
		playerCapacities[p] = nil
	end)
	for _, p in Players:GetPlayers() do
		playerSacks[p] = {}
		playerCapacities[p] = DEFAULT_SACK_CAPACITY
	end
end

return GatheringCraftingService
```

### 4.3 `BaseAndVehicleDefenseService.luau` (Network Ownership + Welding + Boss Wall Lock)

```luau
--!strict
-- ServerScriptService/Services/BaseAndVehicleDefenseService.luau

local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local TypeGuards = require(ReplicatedStorage.Shared.Utils.TypeGuards)
local RemoteGateway = require(ServerScriptService.Core.RemoteGateway)
local EventBus = require(ServerScriptService.Core.EventBus)

local BaseAndVehicleDefenseService = {}

local drillModel: Model? = nil
local driverSeat: VehicleSeat? = nil
local isBossWallEngaged = false

function BaseAndVehicleDefenseService.AttachVehicleModel(model: Model)
	drillModel = model
	-- Mandatory StreamingEnabled & Anti-Exploit Physics rules:
	model.ModelStreamingMode = Enum.ModelStreamingMode.Persistent
	if model.PrimaryPart then
		-- Server retains full physics authority; blocks client speed/teleport exploits
		model.PrimaryPart:SetNetworkOwner(nil)
	end
	driverSeat = model:FindFirstChildWhichIsA("VehicleSeat", true)
	if driverSeat then
		-- Prevent driver from hijacking network ownership on seat occupancy
		driverSeat:GetPropertyChangedSignal("Occupant"):Connect(function()
			if model.PrimaryPart then
				model.PrimaryPart:SetNetworkOwner(nil)
			end
		end)
	end
end

function BaseAndVehicleDefenseService.ValidateAndWeldObject(player: Player, propPart: BasePart, localOffset: CFrame): boolean
	if not drillModel or not drillModel.PrimaryPart then
		return false
	end
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	if not (root and root:IsA("BasePart")) then
		return false
	end
	if (root.Position - drillModel.PrimaryPart.Position).Magnitude > 40 then
		return false
	end

	propPart.CFrame = drillModel.PrimaryPart.CFrame * localOffset
	propPart.Anchored = false
	propPart.Massless = true

	local weld = Instance.new("WeldConstraint")
	weld.Part0 = drillModel.PrimaryPart
	weld.Part1 = propPart
	weld.Parent = propPart
	return true
end

function BaseAndVehicleDefenseService.SetBossWallActive(active: boolean, layerId: string)
	isBossWallEngaged = active
	EventBus.BE_WallStateChanged:Fire({
		wallType = "Boss",
		state = if active then "Engaged" else "Breached",
		layerId = layerId,
	})
end

function BaseAndVehicleDefenseService.AreTeleportersLocked(): boolean
	return isBossWallEngaged
end

function BaseAndVehicleDefenseService.Init()
	RemoteGateway.RegisterEvent({
		name = "RE_VehicleControl",
		policy = { maxPerSecond = 10, cooldownSeconds = 0.05, kickAfterViolations = 8 },
		validator = function(_player: Player, throttle: unknown, action: unknown): (boolean, string?)
			if not TypeGuards.isNumberInRange(throttle, -1, 1) then
				return false, "Invalid throttle"
			end
			if not TypeGuards.isOneOf(action, { "Drive", "Stop", "ResetDrill" }) then
				return false, "Invalid vehicle action"
			end
			return true, nil
		end,
		handler = function(player: Player, throttle: unknown, action: unknown)
			if not driverSeat or not driverSeat.Occupant then
				return
			end
			local char = player.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			if driverSeat.Occupant ~= hum then
				return -- Only current seated driver may control throttle
			end
			if action == "Stop" then
				driverSeat:SetAttribute("TargetThrottle", 0)
			elseif action == "Drive" then
				driverSeat:SetAttribute("TargetThrottle", throttle :: number)
			end
		end,
	})
end

function BaseAndVehicleDefenseService.Start()
	local existingDrill = Workspace:FindFirstChild("DrillMachine", true)
	if existingDrill and existingDrill:IsA("Model") then
		BaseAndVehicleDefenseService.AttachVehicleModel(existingDrill)
	end
end

return BaseAndVehicleDefenseService
```

---

## 5. Retention & Behavioral Rules ("Golden 5 Minutes" + "Push Your Luck")

When implementing onboarding, enemy AI, or progression loops for this genre, enforce these heuristics:

1. **Zero Text Walls — Spatial Guidance (`OnboardingBeamController.luau`):** On first spawn, point a glowing ground `Beam` from the player's `HumanoidRootPart` to the Central Anchor (Campfire / Drill Seat) for the first `<= 180` seconds until first interaction/fuel deposit.
2. **Night-1 Grace Window:** In `ThreatDirectorService`, keep unkillable Stalker threats passive/distant during Night 1 (play warning audio/silhouette only) while letting players finish their first upgrade loop within 180 seconds.
3. **Dual-Purpose Resource Tension:** Mark defeated non-Stone enemies as grabbable `Corpse` items (`DropFuelValue > 0` AND `DropSellValue > 0`) so teams constantly decide between immediate Fuel survival and Coin upgrades.
4. **Point of No Return:** On `Fuel == 0`, hide the Campfire compass/map beacon; during `BossWall` engagement, disable map teleporters (`AreTeleportersLocked()`) and spawn a solid barrier behind the vehicle.
5. **Active Maintenance Threats:** Pair static turrets with anchor saboteurs (`Fuel Leech` draining `1/s`, `Bombo` applying a `60s` `Disabled` attribute to welded defenses) so AFK bases fail.
6. **"One More Run" Meta Bridge:** Even on defeat, commit earned partial `Diamonds`/`Cores` and class `QuestProgress` immediately to `ProfileDataService`, and display next-unlock progress bars in the Lobby UI.
