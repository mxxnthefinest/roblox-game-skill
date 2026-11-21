# Elimination Survival (Round-Based Trials) Genre Template

> **Load this template when:** Building a round-based elimination & survival game modeled after *Death Penalty* (random wheel/TV trials + execution animations + final showdown), *Ink Game* (curated Squid Game elimination ladder: Red Light Green Light, Dalgona, Glass Stepping Stones, Mingle, 1v1 / Sky Finale), or *Simon Says*.
>
> **Requires:** Always apply `references/modular-service-architecture.md` (`--!strict`, `Init()`/`Start()` ModuleScripts, `EventBus`, `RemoteGateway`, `Trove` cleanup, 3-layer data separation) and `templates/game-scaffold.md`.

---

## 1. Global 7-State Finite State Machine & Tempo Rules

```text
Intermission (15-25s, AliveQueue >= MinPlayers)
   │
   ▼
MatchmakingAndTeleport (3-5s: lock roster, MatchState="Alive", clone map, PivotTo + freeze)
   │
   ▼
RuleBriefing (4-8s: broadcast RuleConfig + IssuedAt/DeadlineAt via GetServerTimeNow())
   │
   ▼
ActiveRound (15-90s: server rule engine, position/QTE validation, instant violations)
   │
   ▼
EvaluationAndElimination (3-6s: idempotent execution FX + ragdoll -> MatchState="Spectating")
   ├──► AliveCount > FinaleThreshold && RoundsLeft > 0 ──► RuleBriefing (Next Trial)
   ├──► AliveCount <= FinaleThreshold && AliveCount >= 2 ──► FinaleShowdown (30-180s, revive OFF)
   └──► AliveCount <= 1 ──► RewardsAndLobbyReturn (6-10s: DataStore rewards, RoundTrove:Clean()) ──► Intermission
```

### Tempo & Retention Guardrails

- **Max Dead Time Between Active Rounds:** Keep `EvaluationAndElimination` + `RoundTrove:Clean()` + `RuleBriefing` <= **10–14 seconds** total.
- **Post-Elimination Instant Transition:** Transition `Eliminated` -> `Spectating` within **0.5–1.5s** after execution animation; show live `AliveCount` (`Kalan: 3/24`) and offer lobby mini-obby / `SpectatorSabotage` interaction so eliminated players stay until the finale.
- **Target Match Duration:** 4–7 min (fast random trials, 6–10 mini-rounds) or 10–16 min (curated 5–7 stage ladder).

---

## 2. Strict Shared Types & Tension Math

### 2.1 `ReplicatedStorage/Shared/Types/EliminationTypes.luau`

```luau
--!strict
-- ReplicatedStorage/Shared/Types/EliminationTypes.luau

export type PlayerMatchState = "InLobby" | "Alive" | "Eliminated" | "Spectating"

export type GlobalGameState =
	"Intermission"
	| "MatchmakingAndTeleport"
	| "RuleBriefing"
	| "ActiveRound"
	| "EvaluationAndElimination"
	| "FinaleShowdown"
	| "RewardsAndLobbyReturn"

export type RuleEngineType =
	"SimonCommand"
	| "MotionFreeze"
	| "GlassBridge"
	| "CarveQTE"
	| "RoomCapacity"
	| "Showdown"

export type PenaltyType =
	"Trapdoor"
	| "RagdollFling"
	| "ExecutionShot"
	| "Explosion"
	| "LaserDisintegrate"
	| "CustomCosmetic"

export type RuleConfig = {
	MinigameId: string,
	MapId: string,
	EngineType: RuleEngineType,
	MinPlayers: number,
	MaxPlayers: number,
	BriefingSeconds: number,
	RoundSeconds: number,
	AllowSabotage: boolean,
	IsTrapCommand: boolean?,
	TargetAction: string?,
	MoveThreshold: number?,
}

export type PlayerSessionRecord = {
	UserId: number,
	State: PlayerMatchState,
	RoundJoined: number,
	Strikes: number,
	LastValidCFrame: CFrame?,
	LastCheckTimestamp: number,
	SabotageImmunityUntil: number,
}

export type ExecutionContext = {
	OriginPosition: Vector3?,
	KillerPlayer: Player?,
	Reason: string,
}

export type ViolationPayload = {
	player: Player,
	reason: string,
	severity: "Instant" | "EndOfRound",
	penalty: PenaltyType,
	context: ExecutionContext?,
}

return {}
```

### 2.2 `ReplicatedStorage/Shared/Config/TensionMath.luau`

```luau
--!strict
-- ReplicatedStorage/Shared/Config/TensionMath.luau

local TensionMath = {}

local BASE_GRACE_SECONDS = 0.25
local MAX_PING_COMPENSATION = 0.35 -- Caps lag-switch abuse

-- Exponential command window decay: T(r) = max(T_min, T_base * exp(-k * (r - 1)))
function TensionMath.calculateCommandWindow(roundIndex: number, baseSeconds: number?, minSeconds: number?, decayRate: number?): number
	local r = math.max(1, math.floor(roundIndex))
	local tBase = baseSeconds or 6.0
	local tMin = minSeconds or 2.0
	local k = decayRate or 0.14
	return math.max(tMin, tBase * math.exp(-k * (r - 1)))
end

-- Trap & deception probability scaling: P_trap(r) = min(P_max, P_0 + alpha * (r - 1))
function TensionMath.calculateTrapProbability(roundIndex: number, baseProb: number?, alpha: number?, maxProb: number?): number
	local r = math.max(1, math.floor(roundIndex))
	local p0 = baseProb or 0.15
	local step = alpha or 0.06
	local pMax = maxProb or 0.55
	return math.min(pMax, p0 + step * (r - 1))
end

-- Normalized atmosphere tension index tau in [0, 1]
function TensionMath.calculateTensionIndex(aliveCount: number, initialCount: number): number
	if initialCount <= 0 then
		return 0
	end
	return math.clamp(1 - (math.max(0, aliveCount) / initialCount), 0, 1)
end

-- Deterministic latency-fair reflex check start time using workspace:GetServerTimeNow()
function TensionMath.getEffectiveCheckStartTime(player: Player, phaseServerTime: number): number
	local oneWayLatency = math.clamp(player:GetNetworkPing(), 0, MAX_PING_COMPENSATION)
	return phaseServerTime + BASE_GRACE_SECONDS + oneWayLatency
end

return TensionMath
```

---

## 3. Hybrid Communication Contract (Attributes + Typed Remotes)

| Channel | Mechanism | Direction | Purpose & Anti-Exploit Rule |
|---|---|---|---|
| **Sticky Global State** | `workspace:SetAttribute` | S → C | `GameState`, `PhaseDeadline`, `AliveCount`, `InitialCount`, `CurrentMinigameId` (auto-syncs late joiners with zero polling). |
| **Sticky Player State** | `player:SetAttribute` | S → C | `MatchState` (`"InLobby" \| "Alive" \| "Eliminated" \| "Spectating"`), `WinStreak` (read-only mirror; server table is authoritative). |
| `RE_CommandBroadcast` | `RemoteEvent` | S → C | Rule text, `IssuedAt`, `DeadlineAt` (`workspace:GetServerTimeNow()`), `IsRedLight`. |
| `RE_ExecutionFX` | `RemoteEvent` | S → C | `PenaltyType`, target `UserId`, `Reason` (kill feed + client camera/SFX). |
| `RE_SabotageAlert` | `RemoteEvent` | S → C | `sabotageId`, `initiatorName`, `targetName?` (social attribution). |
| `RE_RoundResults` | `RemoteEvent` | S → C | Near-miss telemetry (`placement`, `totalPlayers`, `distanceRemaining`) + rewards summary. |
| `RE_ClientActionIntent` | `RemoteEvent` | C → S | Raw QTE/input intent (`"CarveTick"`, `"PressButton"`), rate-limited (`10/s`), `Alive` + distance `<= 10 studs`. |
| `RE_SabotageRequest` | `RemoteEvent` | C → S | `sabotageId`, optional `targetUserId`; validated for currency/receipt + cooldown + immunity. |

---

## 4. Core Modular Services (`--!strict`)

### 4.1 `ServerScriptService/Core/PlayerStateService.luau`

```luau
--!strict
-- ServerScriptService/Core/PlayerStateService.luau

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local EliminationTypes = require(ReplicatedStorage.Shared.Types.EliminationTypes)
local Trove = require(ReplicatedStorage.Shared.Utils.Trove)

local PlayerStateService = {}

local serviceTrove = Trove.new()
local records: { [Player]: EliminationTypes.PlayerSessionRecord } = {}

local function initPlayerRecord(player: Player)
	records[player] = {
		UserId = player.UserId,
		State = "InLobby",
		RoundJoined = 0,
		Strikes = 0,
		LastValidCFrame = nil,
		LastCheckTimestamp = workspace:GetServerTimeNow(),
		SabotageImmunityUntil = 0,
	}
	player:SetAttribute("MatchState", "InLobby")
end

function PlayerStateService.Init()
	-- No yields at module level
end

function PlayerStateService.Start()
	serviceTrove:Connect(Players.PlayerAdded, initPlayerRecord)
	serviceTrove:Connect(Players.PlayerRemoving, function(player: Player)
		records[player] = nil
	end)
	for _, player in Players:GetPlayers() do
		initPlayerRecord(player)
	end
end

function PlayerStateService.GetRecord(player: Player): EliminationTypes.PlayerSessionRecord?
	return records[player]
end

function PlayerStateService.GetState(player: Player): EliminationTypes.PlayerMatchState
	local rec = records[player]
	return if rec then rec.State else "InLobby"
end

function PlayerStateService.SetState(player: Player, newState: EliminationTypes.PlayerMatchState)
	local rec = records[player]
	if not rec then
		return
	end
	rec.State = newState
	player:SetAttribute("MatchState", newState)
end

function PlayerStateService.GetPlayersInState(targetState: EliminationTypes.PlayerMatchState): { Player }
	local list: { Player } = {}
	for player, rec in records do
		if player:IsDescendantOf(Players) and rec.State == targetState then
			table.insert(list, player)
		end
	end
	return list
end

return PlayerStateService
```

### 4.2 `ServerScriptService/Services/RuleCommandEngine.luau`

Enforces server-side movement measurement (NEVER trusting client `Humanoid.MoveDirection`), `GetServerTimeNow()` + ping-capped grace windows, `0.25s` speed/teleport rollback, and hidden server-memory Glass Bridge panels (anti-ESP):

```luau
--!strict
-- ServerScriptService/Services/RuleCommandEngine.luau

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local EliminationTypes = require(ReplicatedStorage.Shared.Types.EliminationTypes)
local TensionMath = require(ReplicatedStorage.Shared.Config.TensionMath)
local TypeGuards = require(ReplicatedStorage.Shared.Utils.TypeGuards)
local Trove = require(ReplicatedStorage.Shared.Utils.Trove)
local Signal = require(ReplicatedStorage.Shared.Utils.Signal)
local PlayerStateService = require(ServerScriptService.Core.PlayerStateService)
local RemoteGateway = require(ServerScriptService.Core.RemoteGateway)

local POSITION_EPSILON = 0.35 -- Studs allowed while frozen
local LOOK_DOT_MIN = 0.94     -- ~20 deg max rotation while frozen
local SPEED_CHECK_INTERVAL = 0.25
local PING_DIST_BUFFER = 4.0

local RuleCommandEngine = {}

RuleCommandEngine.OnViolation = Signal.new() :: Signal.Signal<EliminationTypes.ViolationPayload>

local roundTrove = Trove.new()
local currentRule: EliminationTypes.RuleConfig? = nil
local isRedLight = false
local redLightServerTime = 0
local anchorTransforms: { [Player]: { pos: Vector3, look: Vector3, checkAt: number } } = {}
local hiddenSafePanels: { [BasePart]: boolean } = {}
local qteProgress: { [Player]: number } = {}
local speedCheckAcc = 0

local function flagViolation(player: Player, reason: string, severity: "Instant" | "EndOfRound", penalty: EliminationTypes.PenaltyType)
	if PlayerStateService.GetState(player) ~= "Alive" then
		return
	end
	RuleCommandEngine.OnViolation:Fire({
		player = player,
		reason = reason,
		severity = severity,
		penalty = penalty,
		context = { Reason = reason },
	})
end

local function stepAntiTeleportAndMotion(dt: number)
	speedCheckAcc += dt
	local checkSpeedNow = speedCheckAcc >= SPEED_CHECK_INTERVAL
	if checkSpeedNow then
		speedCheckAcc -= SPEED_CHECK_INTERVAL
	end

	local now = workspace:GetServerTimeNow()
	for _, player in PlayerStateService.GetPlayersInState("Alive") do
		local rec = PlayerStateService.GetRecord(player)
		local char = player.Character
		local root = char and char:FindFirstChild("HumanoidRootPart")
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not rec or not (root and root:IsA("BasePart")) or not hum or hum.Health <= 0 then
			continue
		end

		-- 1. Anti-speed / anti-teleport check (every 0.25s)
		if checkSpeedNow then
			if rec.LastValidCFrame then
				local elapsed = math.max(0.05, now - rec.LastCheckTimestamp)
				local maxDist = (hum.WalkSpeed * elapsed) + PING_DIST_BUFFER
				if (root.Position - rec.LastValidCFrame.Position).Magnitude > maxDist then
					rec.Strikes += 1
					if rec.Strikes >= 2 and char then
						char:PivotTo(rec.LastValidCFrame)
					end
				else
					rec.Strikes = math.max(0, rec.Strikes - 1)
					rec.LastValidCFrame = root.CFrame
				end
			else
				rec.LastValidCFrame = root.CFrame
			end
			rec.LastCheckTimestamp = now
		end

		-- 2. MotionFreeze (Red Light) server telemetry check
		if isRedLight and currentRule and currentRule.EngineType == "MotionFreeze" then
			local anchor = anchorTransforms[player]
			if not anchor then
				anchor = {
					pos = root.Position,
					look = root.CFrame.LookVector,
					checkAt = TensionMath.getEffectiveCheckStartTime(player, redLightServerTime),
				}
				anchorTransforms[player] = anchor
			elseif now >= anchor.checkAt then
				-- Capture true post-grace anchor on first active check tick
				local distMoved = (root.Position - anchor.pos).Magnitude
				local lookDot = root.CFrame.LookVector:Dot(anchor.look)
				if distMoved > POSITION_EPSILON or lookDot < LOOK_DOT_MIN then
					flagViolation(player, "Moved during Red Light", "Instant", "ExecutionShot")
				end
			else
				-- Continuously refresh anchor until personal ping-compensated deadline passes
				anchor.pos = root.Position
				anchor.look = root.CFrame.LookVector
			end
		end
	end
end

-- Hidden Glass Bridge initialization: ZERO secret attributes sent to clients
function RuleCommandEngine.RegisterGlassBridgeRows(rows: { { left: BasePart, right: BasePart } })
	table.clear(hiddenSafePanels)
	for _, row in rows do
		local leftSafe = math.random() < 0.5
		hiddenSafePanels[row.left] = leftSafe
		hiddenSafePanels[row.right] = not leftSafe

		for _, panel in { row.left, row.right } do
			panel:SetNetworkOwner(nil)
			roundTrove:Connect(panel.Touched, function(hit: BasePart)
				local model = hit:FindFirstAncestorOfClass("Model")
				local player = if model then Players:GetPlayerFromCharacter(model) else nil
				if not player or PlayerStateService.GetState(player) ~= "Alive" then
					return
				end
				if hiddenSafePanels[panel] == false then
					panel.CanCollide = false
					panel.Transparency = 0.85
					flagViolation(player, "Stepped on tempered weak glass", "Instant", "Trapdoor")
				end
			end)
		end
	end
end

function RuleCommandEngine.BeginRound(rule: EliminationTypes.RuleConfig)
	roundTrove:Clean()
	table.clear(anchorTransforms)
	table.clear(qteProgress)
	currentRule = rule
	isRedLight = false

	local issuedAt = workspace:GetServerTimeNow()
	local deadlineAt = issuedAt + rule.RoundSeconds
	workspace:SetAttribute("PhaseDeadline", deadlineAt)

	RemoteGateway.FireAllClients("RE_CommandBroadcast", {
		engineType = rule.EngineType,
		issuedAt = issuedAt,
		deadlineAt = deadlineAt,
		targetAction = rule.TargetAction,
		isTrap = rule.IsTrapCommand == true,
	})

	roundTrove:Connect(RunService.Heartbeat, stepAntiTeleportAndMotion)
end

function RuleCommandEngine.SetRedLight(active: boolean)
	isRedLight = active
	table.clear(anchorTransforms)
	if active then
		redLightServerTime = workspace:GetServerTimeNow()
	end
	RemoteGateway.FireAllClients("RE_CommandBroadcast", {
		engineType = "MotionFreeze",
		isRedLight = active,
		issuedAt = workspace:GetServerTimeNow(),
	})
end

function RuleCommandEngine.EndRoundCleanup()
	roundTrove:Clean()
	table.clear(anchorTransforms)
	table.clear(hiddenSafePanels)
	table.clear(qteProgress)
	currentRule = nil
	isRedLight = false
end

function RuleCommandEngine.Init()
	RemoteGateway.EnsureEvent("RE_CommandBroadcast")
	RemoteGateway.RegisterEvent({
		name = "RE_ClientActionIntent",
		policy = { maxPerSecond = 10, cooldownSeconds = 0.08, kickAfterViolations = 8 },
		validator = function(player: Player, intentType: unknown): (boolean, string?)
			if PlayerStateService.GetState(player) ~= "Alive" then
				return false, "Player not Alive"
			end
			if not TypeGuards.isOneOf(intentType, { "CarveTick", "PressButton" }) then
				return false, "Invalid intentType"
			end
			return true, nil
		end,
		handler = function(player: Player, intentType: unknown)
			if not currentRule then
				return
			end
			if intentType == "CarveTick" and currentRule.EngineType == "CarveQTE" then
				qteProgress[player] = (qteProgress[player] or 0) + 10
			end
		end,
	})
end

function RuleCommandEngine.Start() end

return RuleCommandEngine
```

### 4.3 `ServerScriptService/Services/EliminationService.luau`

Idempotent elimination, movement lock, `BallSocketConstraint` Ragdoll physics + `ExecutionFX` broadcast, and fast `1.5s` transition to `Spectating`:

```luau
--!strict
-- ServerScriptService/Services/EliminationService.luau

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local EliminationTypes = require(ReplicatedStorage.Shared.Types.EliminationTypes)
local Signal = require(ReplicatedStorage.Shared.Utils.Signal)
local PlayerStateService = require(ServerScriptService.Core.PlayerStateService)
local RemoteGateway = require(ServerScriptService.Core.RemoteGateway)

local SPECTATE_TRANSITION_DELAY = 1.5

local EliminationService = {}

EliminationService.OnPlayerEliminated = Signal.new() :: Signal.Signal<{ player: Player, remainingAlive: number }>

local function applyRagdoll(character: Model, impulseDir: Vector3?)
	local humanoid = character:FindFirstChildOfClass("Humanoid")
	local root = character:FindFirstChild("HumanoidRootPart")
	if not humanoid then
		return
	end
	humanoid.WalkSpeed = 0
	humanoid.JumpPower = 0
	humanoid.AutoRotate = false
	humanoid:ChangeState(Enum.HumanoidStateType.Physics)

	for _, desc in character:GetDescendants() do
		if desc:IsA("Motor6D") and desc.Part0 and desc.Part1 then
			local att0 = Instance.new("Attachment")
			att0.CFrame = desc.C0
			att0.Parent = desc.Part0

			local att1 = Instance.new("Attachment")
			att1.CFrame = desc.C1
			att1.Parent = desc.Part1

			local socket = Instance.new("BallSocketConstraint")
			socket.Attachment0 = att0
			socket.Attachment1 = att1
			socket.LimitsEnabled = true
			socket.TwistLimitsEnabled = true
			socket.Parent = desc.Part0

			local noCol = Instance.new("NoCollisionConstraint")
			noCol.Part0 = desc.Part0
			noCol.Part1 = desc.Part1
			noCol.Parent = desc.Part0

			desc.Enabled = false
		end
	end

	if root and root:IsA("BasePart") and impulseDir then
		root:ApplyImpulse(impulseDir * root.AssemblyMass)
	end
end

local function stripTools(player: Player)
	local backpack = player:FindFirstChildOfClass("Backpack")
	if backpack then
		for _, tool in backpack:GetChildren() do
			if tool:IsA("Tool") then
				tool:Destroy()
			end
		end
	end
	if player.Character then
		for _, tool in player.Character:GetChildren() do
			if tool:IsA("Tool") then
				tool:Destroy()
			end
		end
	end
end

function EliminationService.EliminatePlayer(
	player: Player,
	penaltyType: EliminationTypes.PenaltyType,
	context: EliminationTypes.ExecutionContext?
)
	-- Idempotency guard: prevent double-elimination & double-FX
	if PlayerStateService.GetState(player) ~= "Alive" then
		return
	end

	PlayerStateService.SetState(player, "Eliminated")
	stripTools(player)

	local remainingAlive = #PlayerStateService.GetPlayersInState("Alive")
	workspace:SetAttribute("AliveCount", remainingAlive)

	if player.Character then
		local impulse = if penaltyType == "RagdollFling" then Vector3.new(0, 75, 45) else Vector3.new(0, 20, 0)
		applyRagdoll(player.Character, impulse)
	end

	RemoteGateway.FireAllClients("RE_ExecutionFX", {
		targetUserId = player.UserId,
		targetName = player.Name,
		penaltyType = penaltyType,
		reason = if context then context.Reason else "Eliminated",
		remainingAlive = remainingAlive,
	})

	EliminationService.OnPlayerEliminated:Fire({
		player = player,
		remainingAlive = remainingAlive,
	})

	task.delay(SPECTATE_TRANSITION_DELAY, function()
		if player:IsDescendantOf(Players) and PlayerStateService.GetState(player) == "Eliminated" then
			PlayerStateService.SetState(player, "Spectating")
		end
	end)
end

function EliminationService.Init()
	RemoteGateway.EnsureEvent("RE_ExecutionFX")
end

function EliminationService.Start() end

return EliminationService
```

### 4.4 `ServerScriptService/Services/SpectatorSabotageService.luau`

Anti-P2W spectator tampering (respects `AllowSabotage`, `GlobalSabotageCooldown`, `TargetImmunityWindow`, atomic currency check, and `RE_SabotageAlert` social attribution):

```luau
--!strict
-- ServerScriptService/Services/SpectatorSabotageService.luau

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local TypeGuards = require(ReplicatedStorage.Shared.Utils.TypeGuards)
local PlayerStateService = require(ServerScriptService.Core.PlayerStateService)
local RemoteGateway = require(ServerScriptService.Core.RemoteGateway)

type SabotageDef = {
	costCoins: number,
	impulseStrength: number?,
	requiresTarget: boolean,
}

local GLOBAL_SABOTAGE_COOLDOWN = 12
local TARGET_IMMUNITY_WINDOW = 20

local SABOTAGE_CATALOG: { [string]: SabotageDef } = {
	BananaPeel = { costCoins = 50, impulseStrength = 25, requiresTarget = true },
	FogBlind = { costCoins = 75, impulseStrength = nil, requiresTarget = false },
	PushImpulse = { costCoins = 120, impulseStrength = 45, requiresTarget = true },
}

local SpectatorSabotageService = {}

local allowSabotage = false
local lastGlobalSabotageAt = 0

function SpectatorSabotageService.SetRoundSabotageAllowed(allowed: boolean)
	allowSabotage = allowed
end

function SpectatorSabotageService.Init()
	RemoteGateway.EnsureEvent("RE_SabotageAlert")
	RemoteGateway.RegisterEvent({
		name = "RE_SabotageRequest",
		policy = { maxPerSecond = 2, cooldownSeconds = 1.0, kickAfterViolations = 6 },
		validator = function(player: Player, sabotageId: unknown, targetUserId: unknown): (boolean, string?)
			if PlayerStateService.GetState(player) ~= "Spectating" then
				return false, "Only spectators can trigger sabotage"
			end
			if not TypeGuards.isNonEmptyString(sabotageId, 32) or not SABOTAGE_CATALOG[sabotageId :: string] then
				return false, "Unknown sabotageId"
			end
			if targetUserId ~= nil and not TypeGuards.isIntegerInRange(targetUserId, 1, 9e12) then
				return false, "Invalid targetUserId"
			end
			return true, nil
		end,
		handler = function(player: Player, sabotageId: unknown, targetUserId: unknown)
			if not allowSabotage then
				return
			end
			local now = workspace:GetServerTimeNow()
			if (now - lastGlobalSabotageAt) < GLOBAL_SABOTAGE_COOLDOWN then
				return
			end

			local def = SABOTAGE_CATALOG[sabotageId :: string]
			local targetPlayer: Player? = if targetUserId then Players:GetPlayerByUserId(targetUserId :: number) else nil
			if def.requiresTarget then
				if not targetPlayer or PlayerStateService.GetState(targetPlayer) ~= "Alive" then
					return
				end
				local targetRec = PlayerStateService.GetRecord(targetPlayer)
				if not targetRec or now < targetRec.SabotageImmunityUntil then
					return
				end
				targetRec.SabotageImmunityUntil = now + TARGET_IMMUNITY_WINDOW
			end

			-- Hook: deduct def.costCoins atomically via Economy/ProfileDataService here
			lastGlobalSabotageAt = now

			if targetPlayer and targetPlayer.Character and def.impulseStrength then
				local root = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
				if root and root:IsA("BasePart") then
					root:ApplyImpulse(Vector3.new(0, 15, def.impulseStrength) * root.AssemblyMass)
				end
			end

			RemoteGateway.FireAllClients("RE_SabotageAlert", {
				sabotageId = sabotageId,
				initiatorName = player.Name,
				targetName = if targetPlayer then targetPlayer.Name else nil,
			})
		end,
	})
end

function SpectatorSabotageService.Start() end

return SpectatorSabotageService
```

### 4.5 `StarterPlayerScripts/Controllers/AtmosphereTensionController.luau`

Drives smooth 4-stage visual, audio, and FOV escalation based on `tau = clamp(1 - AliveCount / InitialCount, 0, 1)`:

```luau
--!strict
-- StarterPlayerScripts/Controllers/AtmosphereTensionController.luau

local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local SoundService = game:GetService("SoundService")
local Workspace = game:GetService("Workspace")

local AtmosphereTensionController = {}

local TWEEN_INFO = TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)

local function applyTensionForRatio()
	local initialCount = (Workspace:GetAttribute("InitialCount") :: number?) or 0
	local aliveCount = (Workspace:GetAttribute("AliveCount") :: number?) or 0
	if initialCount <= 0 then
		return
	end

	local tau = math.clamp(1 - (aliveCount / initialCount), 0, 1)
	local targetBrightness = 2.0 - (1.2 * tau)
	local targetFOV = 70 - (6 * tau)
	local targetMusicSpeed = 1.0 + (0.18 * tau)

	TweenService:Create(Lighting, TWEEN_INFO, { Brightness = targetBrightness }):Play()

	local cam = Workspace.CurrentCamera
	if cam then
		TweenService:Create(cam, TWEEN_INFO, { FieldOfView = targetFOV }):Play()
	end

	local bgMusic = SoundService:FindFirstChild("MatchMusic")
	if bgMusic and bgMusic:IsA("Sound") then
		TweenService:Create(bgMusic, TWEEN_INFO, { PlaybackSpeed = targetMusicSpeed }):Play()
	end
end

function AtmosphereTensionController.Init() end

function AtmosphereTensionController.Start()
	Workspace:GetAttributeChangedSignal("AliveCount"):Connect(applyTensionForRatio)
	Workspace:GetAttributeChangedSignal("GameState"):Connect(applyTensionForRatio)
end

return AtmosphereTensionController
```

### 4.6 `ServerScriptService/Services/MinigameRotationManager.luau`

Handles `WheelRandom` vs `CuratedLadder` selection, `ServerStorage.Maps` cloning, `SetNetworkOwner(nil)` on all traps/moving platforms, and zero-leak `RoundTrove:Clean()` map teardown:

```luau
--!strict
-- ServerScriptService/Services/MinigameRotationManager.luau

local ServerStorage = game:GetService("ServerStorage")
local Workspace = game:GetService("Workspace")
local CollectionService = game:GetService("CollectionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local EliminationTypes = require(ReplicatedStorage.Shared.Types.EliminationTypes)
local Trove = require(ReplicatedStorage.Shared.Utils.Trove)

export type RotationMode = "WheelRandom" | "CuratedLadder"

local MinigameRotationManager = {}

local roundTrove = Trove.new()
local activeMapModel: Model? = nil
local playedHistory: { string } = {}

function MinigameRotationManager.ResetHistory()
	table.clear(playedHistory)
end

function MinigameRotationManager.SelectNextRule(
	pool: { EliminationTypes.RuleConfig },
	aliveCount: number,
	mode: RotationMode
): EliminationTypes.RuleConfig?
	if aliveCount <= 2 then
		for _, rule in pool do
			if rule.EngineType == "Showdown" then
				return rule
			end
		end
	end

	local eligible: { EliminationTypes.RuleConfig } = {}
	for _, rule in pool do
		if rule.EngineType ~= "Showdown"
			and aliveCount >= rule.MinPlayers
			and aliveCount <= rule.MaxPlayers
			and not table.find(playedHistory, rule.MinigameId)
		then
			table.insert(eligible, rule)
		end
	end

	if #eligible == 0 then
		-- Reset history if pool exhausted
		table.clear(playedHistory)
		for _, rule in pool do
			if rule.EngineType ~= "Showdown" and aliveCount >= rule.MinPlayers and aliveCount <= rule.MaxPlayers then
				table.insert(eligible, rule)
			end
		end
	end

	if #eligible == 0 then
		return nil
	end

	local chosen = if mode == "CuratedLadder" then eligible[1] else eligible[math.random(1, #eligible)]
	table.insert(playedHistory, chosen.MinigameId)
	return chosen
end

function MinigameRotationManager.LoadMap(mapId: string): { BasePart }
	MinigameRotationManager.CleanupActiveMap()

	local mapsFolder = ServerStorage:FindFirstChild("Maps")
	local template = if mapsFolder then mapsFolder:FindFirstChild(mapId) else nil
	if not (template and template:IsA("Model")) then
		warn(`[MinigameRotationManager] Map not found: {mapId}`)
		return {}
	end

	local clone = template:Clone()
	clone.Name = "ActiveMap"
	clone.Parent = Workspace
	activeMapModel = clone
	roundTrove:Add(clone)

	-- Lock physics authority for all moving traps / platforms on the server
	for _, desc in clone:GetDescendants() do
		if desc:IsA("BasePart") and (CollectionService:HasTag(desc, "MapTrap") or not desc.Anchored) then
			desc:SetNetworkOwner(nil)
		end
	end

	local spawns: { BasePart } = {}
	local spawnsFolder = clone:FindFirstChild("SpawnPoints")
	if spawnsFolder then
		for _, child in spawnsFolder:GetChildren() do
			if child:IsA("BasePart") then
				table.insert(spawns, child)
			end
		end
	end
	return spawns
end

function MinigameRotationManager.GetRoundTrove(): Trove.Trove
	return roundTrove
end

function MinigameRotationManager.CleanupActiveMap()
	roundTrove:Clean()
	activeMapModel = nil
end

function MinigameRotationManager.Init() end
function MinigameRotationManager.Start() end

return MinigameRotationManager
```

### 4.7 `ServerScriptService/Core/GameLoopService.luau`

Drives the 7-state FSM (`Intermission` -> `MatchmakingAndTeleport` -> `RuleBriefing` -> `ActiveRound` -> `EvaluationAndElimination` -> `FinaleShowdown` -> `RewardsAndLobbyReturn`) and checks win conditions immediately when an `Alive` player disconnects:

```luau
--!strict
-- ServerScriptService/Core/GameLoopService.luau

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local EliminationTypes = require(ReplicatedStorage.Shared.Types.EliminationTypes)
local PlayerStateService = require(ServerScriptService.Core.PlayerStateService)
local RuleCommandEngine = require(ServerScriptService.Services.RuleCommandEngine)
local EliminationService = require(ServerScriptService.Services.EliminationService)
local SpectatorSabotageService = require(ServerScriptService.Services.SpectatorSabotageService)
local MinigameRotationManager = require(ServerScriptService.Services.MinigameRotationManager)

local MIN_PLAYERS = 2
local INTERMISSION_SECONDS = 20
local FINALE_THRESHOLD = 2
local MAX_ROUNDS = 7

local GameLoopService = {}

local currentState: EliminationTypes.GlobalGameState = "Intermission"
local roundPool: { EliminationTypes.RuleConfig } = {}
local isRunning = false

local function setGameState(state: EliminationTypes.GlobalGameState, durationSeconds: number?)
	currentState = state
	Workspace:SetAttribute("GameState", state)
	if durationSeconds then
		Workspace:SetAttribute("PhaseDeadline", workspace:GetServerTimeNow() + durationSeconds)
	end
end

local function updateAliveCount(): number
	local count = #PlayerStateService.GetPlayersInState("Alive")
	Workspace:SetAttribute("AliveCount", count)
	return count
end

function GameLoopService.SetRulePool(pool: { EliminationTypes.RuleConfig })
	roundPool = pool
end

function GameLoopService.Init()
	RuleCommandEngine.OnViolation:Connect(function(payload: EliminationTypes.ViolationPayload)
		if payload.severity == "Instant" then
			EliminationService.EliminatePlayer(payload.player, payload.penalty, payload.context)
		end
	end)

	Players.PlayerRemoving:Connect(function(player: Player)
		if PlayerStateService.GetState(player) == "Alive" then
			task.defer(updateAliveCount)
		end
	end)
end

function GameLoopService.Start()
	if isRunning then
		return
	end
	isRunning = true

	task.spawn(function()
		while isRunning do
			-- 1. Intermission
			setGameState("Intermission", INTERMISSION_SECONDS)
			local deadline = workspace:GetServerTimeNow() + INTERMISSION_SECONDS
			while workspace:GetServerTimeNow() < deadline do
				if #Players:GetPlayers() < MIN_PLAYERS then
					deadline = workspace:GetServerTimeNow() + INTERMISSION_SECONDS
					Workspace:SetAttribute("PhaseDeadline", deadline)
				end
				task.wait(0.5)
			end

			-- 2. MatchmakingAndTeleport
			setGameState("MatchmakingAndTeleport", 4)
			local participants = Players:GetPlayers()
			Workspace:SetAttribute("InitialCount", #participants)
			for _, p in participants do
				PlayerStateService.SetState(p, "Alive")
			end
			updateAliveCount()
			MinigameRotationManager.ResetHistory()
			task.wait(3)

			-- 3. Multi-round trial loop
			local roundIdx = 0
			while updateAliveCount() > FINALE_THRESHOLD and roundIdx < MAX_ROUNDS do
				roundIdx += 1
				local rule = MinigameRotationManager.SelectNextRule(roundPool, updateAliveCount(), "WheelRandom")
				if not rule then
					break
				end

				setGameState("RuleBriefing", rule.BriefingSeconds)
				SpectatorSabotageService.SetRoundSabotageAllowed(rule.AllowSabotage)
				local spawns = MinigameRotationManager.LoadMap(rule.MapId)
				local alive = PlayerStateService.GetPlayersInState("Alive")
				for i, p in alive do
					if p.Character and #spawns > 0 then
						p.Character:PivotTo(spawns[((i - 1) % #spawns) + 1].CFrame + Vector3.new(0, 4, 0))
					end
				end
				task.wait(rule.BriefingSeconds)

				setGameState("ActiveRound", rule.RoundSeconds)
				RuleCommandEngine.BeginRound(rule)
				local roundEnd = workspace:GetServerTimeNow() + rule.RoundSeconds
				while workspace:GetServerTimeNow() < roundEnd and updateAliveCount() > 1 do
					task.wait(0.2)
				end

				setGameState("EvaluationAndElimination", 4)
				RuleCommandEngine.EndRoundCleanup()
				task.wait(3)
				MinigameRotationManager.CleanupActiveMap()
			end

			-- 4. FinaleShowdown (if 2..FinaleThreshold remain)
			if updateAliveCount() >= 2 then
				setGameState("FinaleShowdown", 90)
				SpectatorSabotageService.SetRoundSabotageAllowed(false)
				local finaleEnd = workspace:GetServerTimeNow() + 90
				while workspace:GetServerTimeNow() < finaleEnd and updateAliveCount() > 1 do
					task.wait(0.25)
				end
			end

			-- 5. RewardsAndLobbyReturn
			setGameState("RewardsAndLobbyReturn", 7)
			MinigameRotationManager.CleanupActiveMap()
			for _, p in Players:GetPlayers() do
				PlayerStateService.SetState(p, "InLobby")
				task.spawn(p.LoadCharacter, p)
			end
			task.wait(6)
		end
	end)
end

return GameLoopService
```

---

## 5. Viral & Retention Checklist (Near-Miss + Social Flex)

1. **Readable Kill Feed:** Always display `TargetName` + failure reason (`"Kanıp zıpladı!"`, `"Kırmızı ışıkta kıpırdadı"`) so shared-arena mistakes create clippable moments.
2. **Near-Miss Telemetry (`RE_RoundResults`):** When eliminating top-half survivors, send placement (`"24 oyuncu arasından 2. oldun!"`) or remaining distance (`"Bitişe 3 stud kala elendin"`).
3. **Zero-Friction Re-Queue:** Keep `RewardsAndLobbyReturn` <= `10s` and `Intermission` <= `25s` so losers immediately jump into the next trial.
4. **Cosmetic & Status Flex:** Persist `Wins`, `WinStreak`, custom `ExecutionCosmetics`, and `VictoryEmotes` in `ProfileService` (`DataStore`) — never in ephemeral match state.
