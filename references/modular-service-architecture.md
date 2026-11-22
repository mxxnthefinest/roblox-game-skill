# Modular Service Architecture & GDD Technical Standards

> **Load this reference when:** Structuring a new multi-system game, refactoring spaghetti scripts into modular `ModuleScript` services, enforcing `--!strict` Luau type safety, setting up `Signal`/`RemoteGateway` event-driven communication, or separating persistent DataStore profiles from ephemeral run state.

---

## 1. Anti-Spaghetti Engineering Charter (Mandatory)

Every project generated or refactored under this skill MUST follow these structural rules:

1. **Mandatory `--!strict` Mode:**
   - Every `.luau` file (`Script`, `LocalScript`, `ModuleScript`) MUST start on line 1 with `--!strict`.
   - All public service methods, constructors, signal payloads, and state tables MUST carry explicit Luau type annotations.
   - Shared domain data contracts live in `ReplicatedStorage/Shared/Types/` so Server, Client, and Shared code check against one single source of truth.
   - Runtime validation is STILL required on every `RemoteEvent` / `RemoteFunction` boundary (static `--!strict` checks do not protect against malformed payloads sent by exploiters).

2. **Thin Bootstrappers, Fat Services (Zero God Scripts):**
   - `ServerScriptService/ServerMain.server.luau` and `StarterPlayerScripts/ClientMain.client.luau` are **thin bootstrappers** (< 100 lines). They contain ZERO gameplay math, combat rules, or data mutations.
   - All server logic lives inside single-responsibility `ModuleScript` services under `ServerScriptService/Services/`.
   - All client logic lives inside single-responsibility `ModuleScript` controllers under `StarterPlayerScripts/Controllers/`.
   - **Never** place `Script` instances inside `Workspace` or scatter `LocalScript`s across `Workspace`/`Tool` hierarchies for core logic; bind world objects via `CollectionService` tags.

3. **Two-Phase Lifecycle (`Init` → `Start`) & Zero Top-Level Yields:**
   - A `ModuleScript` body MUST NOT yield (`WaitForChild`, `task.wait`, `GetAsync`, `LoadProfileAsync` at top level).
   - Phase 1 (`Init(ctx)`): Register dependencies, create/bind internal `Signal`s, register remotes. Do NOT fire gameplay events or read another service's runtime state yet.
   - Phase 2 (`Start()`): Connect runtime listeners, start `Heartbeat` loops, bind `Players.PlayerAdded` (plus iterate `Players:GetPlayers()` for Studio fast-start).
   - **No circular `require()`:** Services communicate downward to `Core/` + `Shared/` and horizontally ONLY via `EventBus` signals or injected service interfaces passed into `Init(ctx)`.

4. **Strict Client–Server Boundary:**
   - **Client = Input + Presentation ONLY.** Optimistic local animations, SFX, particles, camera shake, UI `TweenService` interpolation.
   - **Server = Single Source of Truth.** All currency, inventory (`InstanceUUID`), crafting, damage, fuel, AI, and physics authority (`PrimaryPart:SetNetworkOwner(nil)`).

5. **3-Layer Data Separation:**
   - **Layer 1 — Persistent Player Profile (`ProfileService` / `ProfileStore`):** Cross-session currency (`Diamonds`/`Cores`), unlocked classes & quest progress, records, achievements, settings.
   - **Layer 2 — Ephemeral Run State (`RunSessionState` + `MemoryStoreService`):** Current match/expedition state (day, phase, fuel, depth, team coins, cleared dungeons). NEVER persisted to `DataStoreService`.
   - **Layer 3 — Hidden Server Definitions (`ServerStorage/Modules/`):** Loot tables, enemy stats, recipe definitions, pricing curves. Clients receive only UI-safe display metadata.

---

## 2. Canonical Modular Folder Hierarchy

```text
game/
├── ServerScriptService/
│   ├── ServerMain.server.luau               -- Thin server bootstrapper (Init -> Start)
│   ├── Core/                                -- Infrastructure modules (no gameplay rules)
│   │   ├── EventBus.luau                    -- Typed inter-service Signals (BE_*)
│   │   ├── RemoteGateway.luau               -- Single Remotes router + runtime guards + rate-limit
│   │   ├── ProfileDataService.luau          -- Persistent ProfileStore/ProfileService wrapper
│   │   └── SessionMemoryService.luau        -- Ephemeral RunSessionState + MemoryStore rejoin cache
│   └── Services/                            -- Single-responsibility Input/Process/Output services
│       ├── GatheringCraftingService.luau
│       ├── CycleAndVitalsService.luau
│       ├── BaseAndVehicleDefenseService.luau
│       ├── EconomyAndShopService.luau
│       ├── ThreatDirectorService.luau
│       └── ClassAndProgressionService.luau
├── ServerStorage/
│   ├── Modules/                             -- Secret configs (invisible to clients)
│   │   ├── LootTables.luau
│   │   ├── RecipeDefinitions.luau
│   │   ├── ClassDefinitions.luau
│   │   └── EnemyDefinitions.luau
│   └── Assets/                              -- Cloned on demand (enemies, maps, structures)
├── ReplicatedStorage/
│   ├── Remotes/                             -- Single container for ALL RE_* and RF_InitialSync
│   └── Shared/
│       ├── Types/
│       │   ├── DataTypes.luau               -- PlayerProfile, RunSessionState, OfferData
│       │   └── NetworkTypes.luau            -- ActionType, InventoryOp, WorldFeedbackType
│       ├── Config/
│       │   ├── Constants.luau               -- Public rates, ranges, limits, remote names
│       │   └── ProgressionMath.luau         -- Pure deterministic scaling equations
│       └── Utils/
│           ├── Signal.luau                  -- Lightweight typed Luau Signal
│           ├── Trove.luau                   -- Strict connection & instance cleanup
│           └── TypeGuards.luau              -- Runtime validation helpers (NaN, bounds, enums)
├── StarterPlayer/
│   └── StarterPlayerScripts/
│       ├── ClientMain.client.luau           -- Thin client bootstrapper
│       └── Controllers/                     -- Presentation & input controllers
│           ├── InputController.luau
│           ├── HUDController.luau
│           ├── FeedbackController.luau
│           └── OnboardingBeamController.luau
├── StarterGui/
│   └── MainGui/                             -- ResetOnSpawn = false
└── Workspace/
    ├── World/                               -- StreamingEnabled = true
    └── Anchors/                             -- Critical base/vehicle models (ModelStreamingMode = Persistent/Atomic)
```

---

## 3. Core Infrastructure Modules (`--!strict`)

### 3.1 Typed Signal (`ReplicatedStorage/Shared/Utils/Signal.luau`)

Use a pure Luau `Signal` for server-to-server `BE_*` events instead of `BindableEvent` when passing typed Luau tables/callbacks in the same VM (avoids Roblox `BindableEvent` table deep-copying and reference stripping, while keeping strict type safety). Use `BindableEvent` only if cross-Actor/Parallel Luau VM signaling is required.

```luau
--!strict
-- ReplicatedStorage/Shared/Utils/Signal.luau

export type Connection = {
	Connected: boolean,
	Disconnect: (self: Connection) -> (),
}

export type Signal<T> = {
	Connect: (self: Signal<T>, handler: (payload: T) -> ()) -> Connection,
	Once: (self: Signal<T>, handler: (payload: T) -> ()) -> Connection,
	Fire: (self: Signal<T>, payload: T) -> (),
	DisconnectAll: (self: Signal<T>) -> (),
}

type InternalNode<T> = {
	handler: (payload: T) -> (),
	connected: boolean,
}

type InternalSignal<T> = {
	_listeners: { InternalNode<T> },
}

local Signal = {}
Signal.__index = Signal

function Signal.new<T>(): Signal<T>
	local self: InternalSignal<T> = {
		_listeners = {},
	}
	setmetatable(self :: any, Signal)
	return (self :: any) :: Signal<T>
end

function Signal.Connect<T>(self: InternalSignal<T>, handler: (payload: T) -> ()): Connection
	local node: InternalNode<T> = {
		handler = handler,
		connected = true,
	}
	table.insert(self._listeners, node)

	local conn: Connection = {
		Connected = true,
		Disconnect = function(c: Connection)
			if not node.connected then
				return
			end
			node.connected = false
			c.Connected = false
			local idx = table.find(self._listeners, node)
			if idx then
				table.remove(self._listeners, idx)
			end
		end,
	}
	return conn
end

function Signal.Once<T>(self: InternalSignal<T>, handler: (payload: T) -> ()): Connection
	local conn: Connection
	conn = Signal.Connect(self, function(payload: T)
		conn:Disconnect()
		handler(payload)
	end)
	return conn
end

function Signal.Fire<T>(self: InternalSignal<T>, payload: T)
	-- Iterate over a snapshot so handlers can safely disconnect during Fire()
	local snapshot = table.clone(self._listeners)
	for _, node in snapshot do
		if node.connected then
			task.spawn(node.handler, payload)
		end
	end
end

function Signal.DisconnectAll<T>(self: InternalSignal<T>)
	for _, node in self._listeners do
		node.connected = false
	end
	table.clear(self._listeners)
end

return Signal
```

### 3.2 Strict Trove (`ReplicatedStorage/Shared/Utils/Trove.luau`)

```luau
--!strict
-- ReplicatedStorage/Shared/Utils/Trove.luau

export type Cleanable = RBXScriptConnection | Instance | () -> () | { Disconnect: (self: any) -> () } | { Destroy: (self: any) -> () } | thread

local Trove = {}
Trove.__index = Trove

export type Trove = typeof(setmetatable({} :: {
	_objects: { Cleanable },
	_cleaned: boolean,
}, Trove))

function Trove.new(): Trove
	local self = setmetatable({
		_objects = {} :: { Cleanable },
		_cleaned = false,
	}, Trove)
	return self
end

function Trove:Add<T>(object: T): T
	if self._cleaned then
		warn("[Trove] Adding object to an already-cleaned Trove")
	end
	table.insert(self._objects, object :: any)
	return object
end

function Trove:Connect(signal: RBXScriptSignal, callback: (...any) -> ()): RBXScriptConnection
	local conn = signal:Connect(callback)
	self:Add(conn)
	return conn
end

function Trove:Clean()
	self._cleaned = true
	for i = #self._objects, 1, -1 do
		local obj: any = self._objects[i]
		local objType = typeof(obj)
		if objType == "RBXScriptConnection" then
			if (obj :: RBXScriptConnection).Connected then
				(obj :: RBXScriptConnection):Disconnect()
			end
		elseif objType == "Instance" then
			(obj :: Instance):Destroy()
		elseif objType == "function" then
			(obj :: () -> ())()
		elseif objType == "thread" then
			pcall(task.cancel, obj :: thread)
		elseif objType == "table" then
			if typeof(obj.Disconnect) == "function" then
				obj:Disconnect()
			elseif typeof(obj.Destroy) == "function" then
				obj:Destroy()
			end
		end
	end
	table.clear(self._objects)
end

function Trove:Destroy()
	self:Clean()
end

return Trove
```

### 3.3 Runtime Type Guards (`ReplicatedStorage/Shared/Utils/TypeGuards.luau`)

```luau
--!strict
-- ReplicatedStorage/Shared/Utils/TypeGuards.luau

local TypeGuards = {}

function TypeGuards.isFiniteNumber(value: unknown): boolean
	if typeof(value) ~= "number" then
		return false
	end
	return value == value and value > -math.huge and value < math.huge
end

function TypeGuards.isNumberInRange(value: unknown, minVal: number, maxVal: number): boolean
	if not TypeGuards.isFiniteNumber(value) then
		return false
	end
	local num = value :: number
	return num >= minVal and num <= maxVal
end

function TypeGuards.isIntegerInRange(value: unknown, minVal: number, maxVal: number): boolean
	if not TypeGuards.isNumberInRange(value, minVal, maxVal) then
		return false
	end
	local num = value :: number
	return math.floor(num) == num
end

function TypeGuards.isNonEmptyString(value: unknown, maxLen: number?): boolean
	if typeof(value) ~= "string" then
		return false
	end
	local str = value :: string
	local limit = maxLen or 128
	return #str > 0 and #str <= limit
end

function TypeGuards.isOneOf<T>(value: unknown, whitelist: { T }): boolean
	for _, allowed in whitelist do
		if value == allowed then
			return true
		end
	end
	return false
end

function TypeGuards.isFiniteVector3(value: unknown): boolean
	if typeof(value) ~= "Vector3" then
		return false
	end
	local vec = value :: Vector3
	return TypeGuards.isFiniteNumber(vec.X)
		and TypeGuards.isFiniteNumber(vec.Y)
		and TypeGuards.isFiniteNumber(vec.Z)
end

return TypeGuards
```

### 3.4 Typed Server EventBus (`ServerScriptService/Core/EventBus.luau`)

All server services talk ONLY through `EventBus` signals or read-only getters—never direct internal table mutations.

```luau
--!strict
-- ServerScriptService/Core/EventBus.luau

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Signal = require(ReplicatedStorage.Shared.Utils.Signal)

export type PhaseChangedPayload = {
	newPhase: "Day" | "Night",
	dayNumber: number,
}

export type AnchorFuelStatePayload = {
	state: "Extinguished" | "Relit" | "LevelUp",
	level: number,
}

export type WallStateChangedPayload = {
	wallType: "Little" | "Large" | "Boss",
	state: "Engaged" | "Breached",
	layerId: string,
}

export type EntityEliminatedPayload = {
	entityId: string,
	killerPlayer: Player?,
	isBoss: boolean,
	mutations: { string },
	pos: Vector3,
}

export type ProgressMilestonePayload = {
	player: Player,
	milestoneId: string,
	rewardType: "Diamond" | "Core",
	amount: number,
}

local EventBus = {
	BE_PhaseChanged = Signal.new() :: Signal.Signal<PhaseChangedPayload>,
	BE_AnchorFuelState = Signal.new() :: Signal.Signal<AnchorFuelStatePayload>,
	BE_WallStateChanged = Signal.new() :: Signal.Signal<WallStateChangedPayload>,
	BE_EntityEliminated = Signal.new() :: Signal.Signal<EntityEliminatedPayload>,
	BE_ProgressMilestone = Signal.new() :: Signal.Signal<ProgressMilestonePayload>,
}

return EventBus
```

### 3.5 Centralized RemoteGateway (`ServerScriptService/Core/RemoteGateway.luau`)

Enforces:
- Single `ReplicatedStorage/Remotes` folder.
- Every `RemoteEvent` passes per-player `os.clock()` rate-limit + cooldown + runtime validator.
- `RemoteFunction` is restricted strictly to `RF_InitialSync` (client-to-server initial state fetch). Never used for gameplay actions; `InvokeClient` is forbidden.
- Throttled 5 Hz delta state replication (`RE_StateSnapshot`) and 250-stud proximity-scoped effects (`RE_WorldFeedback`).

```luau
--!strict
-- ServerScriptService/Core/RemoteGateway.luau

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

export type RatePolicy = {
	maxPerSecond: number,
	cooldownSeconds: number,
	kickAfterViolations: number?,
}

export type EventRegistration = {
	name: string,
	policy: RatePolicy,
	validator: (player: Player, ...unknown) -> (boolean, string?),
	handler: (player: Player, ...unknown) -> (),
}

type PlayerRateTrack = {
	windowStart: number,
	count: number,
	lastAllowed: number,
	violations: number,
}

local DEFAULT_WORLD_FX_RADIUS = 250

local RemoteGateway = {}

local remotesFolder: Folder? = nil
local rateTracks: { [Player]: { [string]: PlayerRateTrack } } = {}
local createdRemotes: { [string]: RemoteEvent | RemoteFunction } = {}

local function getOrCreateFolder(): Folder
	if remotesFolder then
		return remotesFolder
	end
	local existing = ReplicatedStorage:FindFirstChild("Remotes")
	if existing and existing:IsA("Folder") then
		remotesFolder = existing
		return existing
	end
	local folder = Instance.new("Folder")
	folder.Name = "Remotes"
	folder.Parent = ReplicatedStorage
	remotesFolder = folder
	return folder
end

local function checkLimit(player: Player, remoteName: string, policy: RatePolicy): boolean
	local now = os.clock()
	local perPlayer = rateTracks[player]
	if not perPlayer then
		perPlayer = {}
		rateTracks[player] = perPlayer
	end

	local track = perPlayer[remoteName]
	if not track then
		track = {
			windowStart = now,
			count = 0,
			lastAllowed = 0,
			violations = 0,
		}
		perPlayer[remoteName] = track
	end

	if policy.cooldownSeconds > 0 and (now - track.lastAllowed) < policy.cooldownSeconds then
		return false
	end

	if (now - track.windowStart) >= 1.0 then
		track.windowStart = now
		track.count = 0
	end

	track.count += 1
	if track.count > policy.maxPerSecond then
		track.violations += 1
		warn(`[RemoteGateway] Rate limit exceeded: {player.Name} on {remoteName} (#{track.violations})`)
		if policy.kickAfterViolations and track.violations >= policy.kickAfterViolations then
			task.defer(function()
				if player:IsDescendantOf(Players) then
					player:Kick("Network rate limit exceeded.")
				end
			end)
		end
		return false
	end

	track.lastAllowed = now
	return true
end

function RemoteGateway.Init()
	getOrCreateFolder()
	RemoteGateway.EnsureEvent("RE_StateSnapshot")
	RemoteGateway.EnsureEvent("RE_WorldFeedback")
	RemoteGateway.EnsureEvent("RE_OfferPrompt")
	Players.PlayerRemoving:Connect(function(player: Player)
		rateTracks[player] = nil
	end)
end

function RemoteGateway.EnsureEvent(name: string): RemoteEvent
	local cached = createdRemotes[name]
	if cached and cached:IsA("RemoteEvent") then
		return cached
	end
	local folder = getOrCreateFolder()
	local existing = folder:FindFirstChild(name)
	if existing and existing:IsA("RemoteEvent") then
		createdRemotes[name] = existing
		return existing
	end
	local re = Instance.new("RemoteEvent")
	re.Name = name
	re.Parent = folder
	createdRemotes[name] = re
	return re
end

function RemoteGateway.RegisterEvent(reg: EventRegistration)
	local re = RemoteGateway.EnsureEvent(reg.name)
	re.OnServerEvent:Connect(function(player: Player, ...: unknown)
		if not checkLimit(player, reg.name, reg.policy) then
			return
		end
		local ok, err = reg.validator(player, ...)
		if not ok then
			warn(`[RemoteGateway] Rejected {reg.name} from {player.Name}: {err or "invalid payload"}`)
			return
		end
		reg.handler(player, ...)
	end)
end

-- Strictly limited to initial join state synchronization (never gameplay actions)
function RemoteGateway.RegisterInitialSync(
	handler: (player: Player) -> { [string]: any }
)
	local folder = getOrCreateFolder()
	local rf = folder:FindFirstChild("RF_InitialSync")
	if not rf then
		local created = Instance.new("RemoteFunction")
		created.Name = "RF_InitialSync"
		created.Parent = folder
		rf = created
	end
	local syncFunc = rf :: RemoteFunction
	createdRemotes["RF_InitialSync"] = syncFunc

	syncFunc.OnServerInvoke = function(player: Player): { [string]: any }?
		if not checkLimit(player, "RF_InitialSync", {
			maxPerSecond = 1,
			cooldownSeconds = 2,
			kickAfterViolations = 5,
		}) then
			return nil
		end
		return handler(player)
	end
end

function RemoteGateway.FireClient(remoteName: string, player: Player, ...: any)
	local remote = createdRemotes[remoteName]
	if remote and remote:IsA("RemoteEvent") then
		remote:FireClient(player, ...)
	end
end

function RemoteGateway.FireAllClients(remoteName: string, ...: any)
	local remote = createdRemotes[remoteName]
	if remote and remote:IsA("RemoteEvent") then
		remote:FireAllClients(...)
	end
end

function RemoteGateway.BroadcastWorldFeedbackNear(
	origin: Vector3?,
	radius: number?,
	fxType: string,
	meta: { [string]: any }
)
	local remote = createdRemotes["RE_WorldFeedback"]
	if not (remote and remote:IsA("RemoteEvent")) then
		return
	end
	local maxDist = radius or DEFAULT_WORLD_FX_RADIUS
	for _, player in Players:GetPlayers() do
		if not origin then
			remote:FireClient(player, fxType, nil, meta)
			continue
		end
		local char = player.Character
		local root = char and char:FindFirstChild("HumanoidRootPart")
		if root and root:IsA("BasePart") and (root.Position - origin).Magnitude <= maxDist then
			remote:FireClient(player, fxType, origin, meta)
		end
	end
end

return RemoteGateway
```

### 3.6 Ephemeral Run State & MemoryStore Rejoin Recovery (`ServerScriptService/Core/SessionMemoryService.luau`)

Never pollute persistent player `DataStore` profiles with active match state (`RunSessionState`). Keep `RunSessionState` in memory during a session and checkpoint snapshots to `MemoryStoreService` with a short TTL so disconnected players can rejoin mid-run and recover completed dungeons, anchor tier, and depth.

```luau
--!strict
-- ServerScriptService/Core/SessionMemoryService.luau

local MemoryStoreService = game:GetService("MemoryStoreService")
local HttpService = game:GetService("HttpService")

local CHECKPOINT_TTL_SECONDS = 3600 -- 1 hour rejoin window
local MAP_NAME = "ActiveRunCheckpoints_v1"

export type RunCheckpoint = {
	SessionId: string,
	Mode: string,
	CurrentDay: number,
	CurrentDepthMeters: number,
	AnchorLevel: number,
	AnchorFuel: number,
	TeamCoins: number,
	CompletedStructureIds: { string },
	UpdatedAt: number,
}

local SessionMemoryService = {}

local checkpointMap: MemoryStoreHashMap? = nil
local currentRunId: string = HttpService:GenerateGUID(false)

function SessionMemoryService.Init()
	local ok, mapOrErr = pcall(function()
		return MemoryStoreService:GetHashMap(MAP_NAME)
	end)
	if ok then
		checkpointMap = mapOrErr
	else
		warn(`[SessionMemoryService] MemoryStore unavailable (Studio offline?): {mapOrErr}`)
	end
end

function SessionMemoryService.GetSessionId(): string
	return currentRunId
end

function SessionMemoryService.SaveCheckpoint(sessionKey: string, snapshot: RunCheckpoint): boolean
	if not checkpointMap then
		return false
	end
	local ok, err = pcall(function()
		(checkpointMap :: MemoryStoreHashMap):SetAsync(sessionKey, snapshot, CHECKPOINT_TTL_SECONDS)
	end)
	if not ok then
		warn(`[SessionMemoryService] Checkpoint save failed: {err}`)
		return false
	end
	return true
end

function SessionMemoryService.LoadCheckpoint(sessionKey: string): RunCheckpoint?
	if not checkpointMap then
		return nil
	end
	local ok, result = pcall(function()
		return (checkpointMap :: MemoryStoreHashMap):GetAsync(sessionKey)
	end)
	if not ok then
		warn(`[SessionMemoryService] Checkpoint load failed: {result}`)
		return nil
	end
	return result :: RunCheckpoint?
end

function SessionMemoryService.ClearCheckpoint(sessionKey: string)
	if not checkpointMap then
		return
	end
	pcall(function()
		(checkpointMap :: MemoryStoreHashMap):RemoveAsync(sessionKey)
	end)
end

return SessionMemoryService
```

---

## 4. Six-Service Input / Process / Output Architecture

Divide complex gameplay into isolated `ModuleScript` services that communicate strictly via `EventBus` (`BE_*`) and explicit public getters:

| Service (`ServerScriptService/Services/`) | Owns (Single Source of Truth) | Inputs | Key Server Validations | Emits (`EventBus`) |
|---|---|---|---|---|
| `GatheringCraftingService` | Resource node HP, per-player sack slots (`InstanceUUID`), bench/craft placement | `RE_ActionRequest` (`Mine`), `RE_InventoryAction`, `RE_CraftOrUpgrade` | Magnitude `<= Range + 4`, `os.clock()` cooldown, tool ownership, node mutation vs damage type (`Pickaxe`/`Explosive` for `Stone`), atomic recipe deduction | `InventoryUpdated` (local), consumes `BE_EntityEliminated` for corpses |
| `CycleAndVitalsService` | Day/Night phase (`180s`/`90s`), `DayMultiplier` (1..9), anchor fuel/level, player Hunger/Freeze/HP | Heartbeat `dt`, `PartySize`, weather modifiers, fuel deposits, food/potion use | Server tick drain `dt * BaseDrain * WeatherMult * PartyScale`, Leech drain, SafeZone radius check, food cooldown | `BE_PhaseChanged`, `BE_AnchorFuelState` |
| `BaseAndVehicleDefenseService` | Anchor/Drill CFrame & physics, welded attachments, turrets, wall HP, SafeZone boundary | `RE_VehicleControl` (`Throttle`/`Stop`/`ResetDrill`), `RE_InventoryAction` (`Weld`), enemy sabotage | Driver seat occupancy check, `PrimaryPart:SetNetworkOwner(nil)`, server `WeldConstraint` check, Raycast LoS turrets, Boss Wall lock | `BE_WallStateChanged` |
| `EconomyAndShopService` | Session `TeamCoins`, per-player vendor offers (`PeltTrader`), lobby class/boost purchases | `RE_CraftOrUpgrade` (`Shop`/`PeltTrader`), meta rewards from `BE_ProgressMilestone` | Server-only price & multiplier math (`Base * Layer * Class * Event`), `if balance >= cost`, per-player trade state | Sends `RE_OfferPrompt` & currency deltas |
| `ThreatDirectorService` | AI Director budget, active NPCs (`CollectionService` tags), FSM states, weather events | Day/Night (`BE_PhaseChanged`), fuel=0 (`BE_AnchorFuelState`), depth, light cone (`RE_ActionRequest` `Flashlight`) | Day-1 grace period, `DotProduct` flashlight cone + range check, diminishing stun returns, 3-point Boss wave spawning | `BE_EntityEliminated` |
| `ClassAndProgressionService` | Active class modifiers (`StatModifiers`), class quest accumulation, `BadgeService`, `DayMultiplier` | `BE_EntityEliminated`, `BE_WallStateChanged`, fuel/mining/rescue stats | Server-only stat modifier registration, persistent profile quest updates via `ProfileDataService`, milestone verification | `BE_ProgressMilestone` |

---

## 5. Thin Bootstrapper Pattern (`ServerMain.server.luau`)

```luau
--!strict
-- ServerScriptService/ServerMain.server.luau
-- Thin bootstrapper: NO gameplay logic here.

local ServerScriptService = game:GetService("ServerScriptService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Core = ServerScriptService:WaitForChild("Core", 10)
local Services = ServerScriptService:WaitForChild("Services", 10)
assert(Core and Services, "[ServerMain] Missing Core or Services folder")

local RemoteGateway = require(Core.RemoteGateway)
local ProfileDataService = require(Core.ProfileDataService)
local SessionMemoryService = require(Core.SessionMemoryService)

export type LifecycleService = {
	Init: (() -> ())?,
	Start: (() -> ())?,
}

-- 1. Boot Core infrastructure in strict dependency order
RemoteGateway.Init()
ProfileDataService.Init()
SessionMemoryService.Init()

-- 2. Require domain services (must NOT yield in module bodies)
local serviceList: { LifecycleService } = {
	require(Services.ClassAndProgressionService) :: LifecycleService,
	require(Services.CycleAndVitalsService) :: LifecycleService,
	require(Services.BaseAndVehicleDefenseService) :: LifecycleService,
	require(Services.GatheringCraftingService) :: LifecycleService,
	require(Services.EconomyAndShopService) :: LifecycleService,
	require(Services.ThreatDirectorService) :: LifecycleService,
}

-- 3. Phase 1: Init (wire remotes, event subscriptions, internal state)
for _, service in serviceList do
	if service.Init then
		service.Init()
	end
end

-- 4. Phase 2: Start (start loops, connect PlayerAdded, spawn initial world)
for _, service in serviceList do
	if service.Start then
		service.Start()
	end
end

-- 5. Enforce StreamingEnabled critical settings
workspace.StreamingEnabled = true
workspace.StreamingMinRadius = 128
workspace.StreamingTargetRadius = 256

print("[ServerMain] All Core & Domain services initialized and started.")
```

---

## 6. Client–Server Authority & Streaming Checklist

Before shipping or accepting any module into the codebase, verify ALL 7 rules:

| System | Client Role (Presentation/Input) | Server Authority & Exploit Defense |
|---|---|---|
| **Mining / Chopping / Combat** | Local swing animation, hit VFX, camera shake, raycast preview | `(PlayerPos - TargetPos).Magnitude <= Range + 4`, `os.clock()` rate limit, tool ownership, damage-type vs mutation check |
| **Inventory / Sack** | Hotbar/sack UI, mobile-friendly touch drag preview | Slot capacity checks, unique `InstanceUUID` per item instance to block duplication races |
| **Economy & Vendors** | Vendor/shop UI, per-player `RE_OfferPrompt` rendering | Server computes `BaseValue * Mults`; atomic `balance >= cost`; never accept price/amount from client |
| **Vitals & Anchor Fuel** | `TweenService` HUD bar interpolation, low-HP/fuel vignette | Fixed server `Heartbeat` tick drain, SafeZone distance check, consumable cooldowns |
| **Vehicles / Anchors & Welding** | Driver throttle/brake input (`RE_VehicleControl`), speedometer, camera shake | `PrimaryPart:SetNetworkOwner(nil)` prevents speed/teleport hacks; server validates `WeldConstraint` attachments |
| **Enemy AI & Flashlight** | Client-side walk/attack animations, local audio, spark VFX | Server hitbox/pathing via `CollectionService` tags; server `DotProduct` + range check for flashlight stun |
| **Streaming & Persistence** | `StreamingEnabled = true` streams distant chunks out | Set `ModelStreamingMode = Enum.ModelStreamingMode.Atomic` or `Persistent` on Campfire, DrillMachine, and critical hubs |

---

## 7. Round-Based Elimination FSM & Hybrid Clock/Attribute Standards

For trial/elimination games (*Death Penalty*, *Ink Game*, *Simon Says* — see `templates/genre-elimination-survival.md`), apply four additional architectural invariants:

1. **Global FSM + `RoundTrove` Deterministic Cleanup:**
   - Orchestrate rounds via `GameLoopService` (`Intermission` → `MatchmakingAndTeleport` → `RuleBriefing` → `ActiveRound` → `EvaluationAndElimination` → `FinaleShowdown` → `RewardsAndLobbyReturn`).
   - Bind every round's cloned map (`workspace.ActiveMap`), `RBXScriptConnection`s, temporary tools, and `task` threads inside a per-round `RoundTrove` so `RoundTrove:Clean()` wipes 100% of round state between rounds with zero leaks.
2. **Single Clock Source (`workspace:GetServerTimeNow()`):**
   - Never use `tick()` or `os.time()` for round countdowns or reflex freeze windows.
   - Server broadcasts `IssuedAt` / `DeadlineAt`; reflex checks apply `BASE_GRACE + math.clamp(player:GetNetworkPing(), 0, 0.35)` to protect legitimate high-ping players while capping lag-switch abuse.
3. **Hybrid State Sync (`Attribute` + One-Way `RemoteEvent`):**
   - Publish sticky states (`workspace` `GameState`, `PhaseDeadline`, `AliveCount`; `Player` `MatchState`) via `Instance:SetAttribute` so late joiners sync automatically and client UI reacts via `GetAttributeChangedSignal`.
   - Use typed one-way `RemoteEvent`s (`CommandBroadcast`, `ExecutionFX`, `SabotageAlert`, `ClientActionIntent`) for transient events.
4. **Zero Client Secrets & Server Motion Measurement:**
   - Never expose hidden panel/trap truth tables (e.g., Glass Bridge safe tiles) in `ReplicatedStorage` or `Attribute`s — hold `{ [BasePart]: boolean }` strictly in `ServerScriptService`.
   - Measure movement/freeze rules directly from `HumanoidRootPart.Position` & `LookVector` deltas on the server (`Humanoid.MoveDirection` from clients is ignored); lock moving traps/platforms with `Part:SetNetworkOwner(nil)`.

---

## 8. Co-op Physics Constraints, Velocity Watchdog & Binary `buffer` Streaming

For coupled 2-player physics & asymmetric obbies (*Chained*, *Carry the Glass Together!*, *Be My Camera*, *Throw Me* — see `templates/genre-coop-physics-obby.md`):

1. **Split-Ownership Physics Rule:**
   - **Never** call `SetNetworkOwner(nil)` on live player characters (causes unbearable input lag). Keep each client owning their `HumanoidRootPart`, cap `Constraint.MaxForce` (e.g., `2.5 * m * g`), and enforce a `RunService.PostSimulation` server velocity watchdog clamping `||v||` and rejecting `NaN`/`Inf`.
   - **Do** call `SetNetworkOwner(nil)` on shared carried assemblies (`GlassPane`, vehicles, traps) for deterministic shatter/impact checks (`DeltaV = (v_t - v_{t-dt}).Magnitude`).
2. **Atomic Checkpoint Reset Sequence:**
   - Set `session.IsResetting = true`, disable all `ActiveConstraints`, zero `AssemblyLinearVelocity` and `AssemblyAngularVelocity` on every part, execute `Model:PivotTo()`, yield 1 physics step (`RunService.PostSimulation:Wait()`), then re-enable constraints.
3. **18-Byte `buffer` + `UnreliableRemoteEvent` Camera/Transform Streaming:**
   - Pack transform as `3x f32` (12 bytes) + `3x i16` quantized Euler orientation (6 bytes) = **18 bytes** at **15–20 Hz** (`< 0.4 KB/s` per duo), interpolate client-side via `alpha = 1 - math.exp(-lambda * dt)` in `BindToRenderStep`, and call `Player:RequestStreamAroundAsync(pos)` if `StreamingEnabled = true`.
4. **Pooled Ink/Draw Segments & Targeted Purchase Fallback:**
   - Validate client-drawn segments on the server (`length <= 12` studs, `InkBudget`, max 10 Hz) using a pre-allocated `Part` Object Pool with TTL.
   - In `ProcessReceipt` for targeted sabotages, grant a reusable inventory token if the target disconnected mid-checkout so buyers never lose Robux.
