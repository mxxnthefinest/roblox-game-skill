# 2-Player Co-op Physics & Asymmetric Teamwork Obby Template

> **Load this template when:** Building a 2-player co-op physics or asymmetric teamwork obby modeled after *Chained*, *Carry the Glass Together!*, *Be My Camera*, *Throw Me*, *Gecko Duo*, *Draw Together!*, or *Kick Your Friend*.
>
> **Requires:** Always apply `references/modular-service-architecture.md` (`--!strict`, `Init()`/`Start()` ModuleScripts, `EventBus`, `RemoteGateway`, `Trove` cleanup, 3-layer data separation) and `templates/game-scaffold.md`.

---

## 1. Core Loop & "One More Try" Mechanics

```text
Lobby ──► DuoPairing (0.2s Pad Polling / Invite) ──► RoleAssignment
   │
   ▼
Traversal (Coupled Physics / Camera Stream / Ink Draw / Ballistic Throw)
   ├──► CheckpointReached ──► Save PartnerCheckpoint + Advance Stage
   └──► FailureState (Fall / Glass Shatter DeltaV > 22 / Sabotage)
           │
           ▼
        RagdollReset (1.2s - 2.0s comedic failure window)
           │
           ▼
        Atomic CheckpointRespawn (Disable Constraints -> Zero Velocities -> PivotTo -> PostSimulation:Wait() -> Re-enable)
```

### Design Pillars

1. **Productive Friction:** Partner A's inputs physically perturb Partner B's state (`RopeConstraint`, carried `GlassPane`, launch impulse, or shared camera).
2. **Comedic Readability (1.2–2.0s):** Failures manifest as visible slapstick physics chains before resetting cleanly.
3. **Zero-Friction Instant Respawn (< 2s):** Never kick back to lobby on stage falls; atomically reset both players and shared assemblies at the active checkpoint.

---

## 2. Strict Shared Types & 18-Byte Binary `buffer` Replication

### 2.1 `ReplicatedStorage/Shared/Types/CoopPhysicsTypes.luau`

```luau
--!strict
-- ReplicatedStorage/Shared/Types/CoopPhysicsTypes.luau

export type DuoRole = "Primary" | "Secondary" | "Camera" | "Runner" | "Drawer" | "Thrower"

export type TetherType = "Rope" | "Rod" | "Elastic" | "CarriedObject"

export type TetherConfig = {
	BaseLength: number,
	Restitution: number?,
	MaxForce: number?,
	ShatterDeltaV: number?,
}

export type DuoSession = {
	SessionId: string,
	PlayerA: Player,
	PlayerB: Player?,
	Roles: { [Player]: DuoRole },
	CurrentStage: number,
	IsResetting: boolean,
	SharedAssembly: Model?,
	ActiveConstraints: { Constraint },
	CreatedAt: number,
}

export type PlayerProfileSchema = {
	SchemaVersion: number,
	HighestStageReached: number,
	ActiveSoloStage: number,
	PartnerCheckpoints: { [string]: number }, -- [PartnerUserId_String] = Stage (max 20 entries)
	Prestiges: number,
	Currency: number,
	OwnedCosmetics: {
		Chains: { string },
		GlassSkins: { string },
		Trails: { string },
	},
	Equipped: {
		Chain: string,
		GlassSkin: string,
		Trail: string,
	},
	Stats: {
		TotalDeaths: number,
		GlassShatters: number,
		PartnersThrown: number,
		SabotagesBought: number,
		FastestRunSeconds: number?,
	},
	ProcessedReceipts: { [string]: boolean },
}

return {}
```

### 2.2 `ReplicatedStorage/Shared/Utils/CFrameBuffer.luau` (18-Byte `buffer` Serialization)

Packs a `CFrame` into **18 bytes** (`3x f32` position + `3x i16` quantized Euler angles) instead of 48-byte userdata (`< 0.4 KB/s` @ 20 Hz over `UnreliableRemoteEvent`):

```luau
--!strict
-- ReplicatedStorage/Shared/Utils/CFrameBuffer.luau

local CFrameBuffer = {}

local ANGLE_SCALE = 32767 / math.pi
local INV_ANGLE_SCALE = math.pi / 32767

function CFrameBuffer.pack(cf: CFrame): buffer
	local buf = buffer.create(18)
	local pos = cf.Position
	local rx, ry, rz = cf:ToOrientation()

	buffer.writef32(buf, 0, pos.X)
	buffer.writef32(buf, 4, pos.Y)
	buffer.writef32(buf, 8, pos.Z)

	buffer.writei16(buf, 12, math.round(math.clamp(rx, -math.pi, math.pi) * ANGLE_SCALE))
	buffer.writei16(buf, 14, math.round(math.clamp(ry, -math.pi, math.pi) * ANGLE_SCALE))
	buffer.writei16(buf, 16, math.round(math.clamp(rz, -math.pi, math.pi) * ANGLE_SCALE))
	return buf
end

function CFrameBuffer.unpack(buf: buffer): CFrame?
	if buffer.len(buf) ~= 18 then
		return nil
	end
	local x = buffer.readf32(buf, 0)
	local y = buffer.readf32(buf, 4)
	local z = buffer.readf32(buf, 8)
	-- Reject NaN/Inf payloads
	if x ~= x or y ~= y or z ~= z or math.abs(x) == math.huge or math.abs(y) == math.huge or math.abs(z) == math.huge then
		return nil
	end

	local rx = buffer.readi16(buf, 12) * INV_ANGLE_SCALE
	local ry = buffer.readi16(buf, 14) * INV_ANGLE_SCALE
	local rz = buffer.readi16(buf, 16) * INV_ANGLE_SCALE
	return CFrame.new(x, y, z) * CFrame.fromOrientation(rx, ry, rz)
end

return CFrameBuffer
```

---

## 3. Core Modular Services (`--!strict`)

### 3.1 `ServerScriptService/Services/DuoManager.luau`

Handles `Workspace:GetPartsInPart()` spatial pad pairing (`0.2s` polling), disconnect fallback (Bot/Safe Unlinking), and **Atomic Checkpoint Respawn** (prevents physics teleport flings):

```luau
--!strict
-- ServerScriptService/Services/DuoManager.luau

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local CollectionService = game:GetService("CollectionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local CoopPhysicsTypes = require(ReplicatedStorage.Shared.Types.CoopPhysicsTypes)
local Signal = require(ReplicatedStorage.Shared.Utils.Signal)
local Trove = require(ReplicatedStorage.Shared.Utils.Trove)

local PAD_POLL_INTERVAL = 0.2

local DuoManager = {}

DuoManager.DuoPairedSignal = Signal.new() :: Signal.Signal<CoopPhysicsTypes.DuoSession>
DuoManager.DuoRespawnedSignal = Signal.new() :: Signal.Signal<{ session: CoopPhysicsTypes.DuoSession, stageIndex: number }>
DuoManager.DuoDisbandedSignal = Signal.new() :: Signal.Signal<{ sessionId: string, reason: string }>

local serviceTrove = Trove.new()
local sessionsById: { [string]: CoopPhysicsTypes.DuoSession } = {}
local sessionByPlayer: { [Player]: CoopPhysicsTypes.DuoSession } = {}
local checkpointCFrames: { [number]: CFrame } = {}
local pollAcc = 0

local function zeroAssemblyVelocity(model: Model)
	for _, desc in model:GetDescendants() do
		if desc:IsA("BasePart") then
			desc.AssemblyLinearVelocity = Vector3.zero
			desc.AssemblyAngularVelocity = Vector3.zero
		end
	end
end

function DuoManager.RegisterCheckpoint(stageIndex: number, cf: CFrame)
	checkpointCFrames[stageIndex] = cf
end

function DuoManager.GetSessionForPlayer(player: Player): CoopPhysicsTypes.DuoSession?
	return sessionByPlayer[player]
end

function DuoManager.GetAllSessions(): { [string]: CoopPhysicsTypes.DuoSession }
	return sessionsById
end

function DuoManager.CreateDuoSession(
	playerA: Player,
	playerB: Player?,
	startStage: number?
): CoopPhysicsTypes.DuoSession?
	if sessionByPlayer[playerA] or (playerB and sessionByPlayer[playerB]) then
		return nil
	end

	local session: CoopPhysicsTypes.DuoSession = {
		SessionId = HttpService:GenerateGUID(false),
		PlayerA = playerA,
		PlayerB = playerB,
		Roles = {
			[playerA] = "Primary",
		},
		CurrentStage = startStage or 1,
		IsResetting = false,
		SharedAssembly = nil,
		ActiveConstraints = {},
		CreatedAt = os.clock(),
	}
	if playerB then
		session.Roles[playerB] = "Secondary"
		sessionByPlayer[playerB] = session
	end

	sessionsById[session.SessionId] = session
	sessionByPlayer[playerA] = session
	DuoManager.DuoPairedSignal:Fire(session)
	return session
end

-- Atomic synchronized checkpoint reset: prevents constraint explosions and recursive death triggers
function DuoManager.RespawnDuoAtCheckpoint(session: CoopPhysicsTypes.DuoSession)
	if session.IsResetting then
		return
	end
	session.IsResetting = true

	local baseCF = checkpointCFrames[session.CurrentStage] or CFrame.new(0, 10, 0)

	-- 1. Temporarily disable all physical constraints
	for _, constraint in session.ActiveConstraints do
		constraint.Enabled = false
	end

	-- 2. Zero velocities and PivotTo offset spawns
	local members: { Player } = { session.PlayerA }
	if session.PlayerB then
		table.insert(members, session.PlayerB)
	end

	for idx, member in members do
		local char = member.Character
		if char then
			zeroAssemblyVelocity(char)
			local offsetX = if idx == 1 then -3 else 3
			char:PivotTo(baseCF * CFrame.new(offsetX, 4, 0))
		end
	end

	if session.SharedAssembly then
		zeroAssemblyVelocity(session.SharedAssembly)
		session.SharedAssembly:PivotTo(baseCF * CFrame.new(0, 5, 0))
	end

	-- 3. Wait 1 physics step before re-enabling constraints
	RunService.PostSimulation:Wait()
	for _, constraint in session.ActiveConstraints do
		if constraint.Parent then
			constraint.Enabled = true
		end
	end

	session.IsResetting = false
	DuoManager.DuoRespawnedSignal:Fire({
		session = session,
		stageIndex = session.CurrentStage,
	})
end

function DuoManager.DisbandSession(sessionId: string, reason: string)
	local session = sessionsById[sessionId]
	if not session then
		return
	end
	for _, c in session.ActiveConstraints do
		c:Destroy()
	end
	table.clear(session.ActiveConstraints)

	sessionByPlayer[session.PlayerA] = nil
	if session.PlayerB then
		sessionByPlayer[session.PlayerB] = nil
	end
	sessionsById[sessionId] = nil

	DuoManager.DuoDisbandedSignal:Fire({
		sessionId = sessionId,
		reason = reason,
	})
end

local function pollPairingPads()
	for _, pad in CollectionService:GetTagged("DuoPairingPad") do
		if not pad:IsA("BasePart") then
			continue
		end
		local parts = workspace:GetPartsInPart(pad)
		local candidates: { Player } = {}
		for _, part in parts do
			local model = part:FindFirstAncestorOfClass("Model")
			local player = if model then Players:GetPlayerFromCharacter(model) else nil
			if player and not sessionByPlayer[player] and not table.find(candidates, player) then
				table.insert(candidates, player)
			end
			if #candidates >= 2 then
				break
			end
		end
		if #candidates == 2 then
			DuoManager.CreateDuoSession(candidates[1], candidates[2], 1)
		end
	end
end

function DuoManager.Init() end

function DuoManager.Start()
	serviceTrove:Connect(Players.PlayerRemoving, function(player: Player)
		local session = sessionByPlayer[player]
		if not session then
			return
		end
		-- Mode B Safe Unlinking (or hook Mode A Bot fallback here)
		DuoManager.DisbandSession(session.SessionId, "PartnerDisconnected")
	end)

	serviceTrove:Connect(RunService.Heartbeat, function(dt: number)
		pollAcc += dt
		if pollAcc >= PAD_POLL_INTERVAL then
			pollAcc -= PAD_POLL_INTERVAL
			pollPairingPads()
		end
	end)
end

return DuoManager
```

### 3.2 `ServerScriptService/Services/PhysicsTetherService.luau`

Enforces:
- **NEVER `SetNetworkOwner(nil)` on live characters** (prevents input lag; uses `RopeConstraint` + capped soft spring + `PostSimulation` server velocity watchdog clamping `||v|| > 120 studs/s` & `NaN`/`Inf`!).
- **DO use `SetNetworkOwner(nil)` on shared carried assemblies (`GlassPane`)** with velocity-delta shatter physics ($\Delta v = \|\vec{v}_t - \vec{v}_{t - \Delta t}\|$) and pooled fracture shards.

```luau
--!strict
-- ServerScriptService/Services/PhysicsTetherService.luau

local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerScriptService = game:GetService("ServerScriptService")

local CoopPhysicsTypes = require(ReplicatedStorage.Shared.Types.CoopPhysicsTypes)
local Signal = require(ReplicatedStorage.Shared.Utils.Signal)
local Trove = require(ReplicatedStorage.Shared.Utils.Trove)
local DuoManager = require(ServerScriptService.Services.DuoManager)

local MAX_SAFE_CHAR_SPEED = 120 -- Studs/sec ceiling outside authorized ThrowMe impulse
local DEFAULT_SHATTER_DELTA_V = 22

local PhysicsTetherService = {}

PhysicsTetherService.ObjectShatteredSignal = Signal.new() :: Signal.Signal<{
	session: CoopPhysicsTypes.DuoSession,
	impactVelocity: number,
	impactPosition: Vector3,
}>

local serviceTrove = Trove.new()
local authorizedLaunchUntil: { [Player]: number } = {}
local lastObjectVelocities: { [string]: Vector3 } = {}
local activeTetherFolder: Folder? = nil

local function getTetherFolder(): Folder
	if activeTetherFolder and activeTetherFolder.Parent then
		return activeTetherFolder
	end
	local folder = Instance.new("Folder")
	folder.Name = "ActiveTethers"
	folder.Parent = Workspace
	activeTetherFolder = folder
	return folder
end

function PhysicsTetherService.GrantLaunchWindow(player: Player, durationSeconds: number)
	authorizedLaunchUntil[player] = os.clock() + math.clamp(durationSeconds, 0.2, 4.0)
end

function PhysicsTetherService.BindRopeTether(session: CoopPhysicsTypes.DuoSession, config: CoopPhysicsTypes.TetherConfig): RopeConstraint?
	if not session.PlayerB then
		return nil
	end
	local charA = session.PlayerA.Character
	local charB = session.PlayerB.Character
	local attachPartA = charA and (charA:FindFirstChild("UpperTorso") or charA:FindFirstChild("Torso") or charA:FindFirstChild("HumanoidRootPart"))
	local attachPartB = charB and (charB:FindFirstChild("UpperTorso") or charB:FindFirstChild("Torso") or charB:FindFirstChild("HumanoidRootPart"))
	if not (attachPartA and attachPartA:IsA("BasePart") and attachPartB and attachPartB:IsA("BasePart")) then
		return nil
	end

	-- NEVER call SetNetworkOwner(nil) on live characters; keep client ownership & guard via PostSimulation watchdog
	local att0 = Instance.new("Attachment")
	att0.Name = "TetherAtt0"
	att0.Parent = attachPartA

	local att1 = Instance.new("Attachment")
	att1.Name = "TetherAtt1"
	att1.Parent = attachPartB

	local rope = Instance.new("RopeConstraint")
	rope.Name = `Rope_{session.SessionId}`
	rope.Attachment0 = att0
	rope.Attachment1 = att1
	rope.Visible = true
	rope.Length = config.BaseLength
	rope.Restitution = config.Restitution or 0.1
	rope.WinchEnabled = false
	rope.Parent = getTetherFolder()

	table.insert(session.ActiveConstraints, rope)
	return rope
end

-- Server-owned shared fragile object (Carry the Glass Together!)
function PhysicsTetherService.BindCarriedGlass(session: CoopPhysicsTypes.DuoSession, glassModel: Model, shatterDeltaV: number?)
	session.SharedAssembly = glassModel
	glassModel.Parent = getTetherFolder()
	if glassModel.PrimaryPart then
		-- Lock shared carried object to server to prevent ownership-handoff jitter & verify shatter truthfully
		glassModel.PrimaryPart:SetNetworkOwner(nil)
		lastObjectVelocities[session.SessionId] = glassModel.PrimaryPart.AssemblyLinearVelocity
	end
	glassModel:SetAttribute("ShatterDeltaV", shatterDeltaV or DEFAULT_SHATTER_DELTA_V)
end

local function clampExploitVelocity(player: Player)
	if authorizedLaunchUntil[player] and os.clock() < authorizedLaunchUntil[player] then
		return
	end
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	if not (root and root:IsA("BasePart")) then
		return
	end

	local vel = root.AssemblyLinearVelocity
	local pos = root.Position
	-- Guard NaN / Inf
	if pos ~= pos or vel ~= vel or math.abs(vel.X) == math.huge or math.abs(vel.Y) == math.huge or math.abs(vel.Z) == math.huge then
		root.AssemblyLinearVelocity = Vector3.zero
		return
	end

	if vel.Magnitude > MAX_SAFE_CHAR_SPEED then
		root.AssemblyLinearVelocity = vel.Unit * MAX_SAFE_CHAR_SPEED
	end
end

function PhysicsTetherService.Init()
	getTetherFolder()
end

function PhysicsTetherService.Start()
	-- Anti-Exploit Velocity Watchdog + Fragile Shatter Delta Check
	serviceTrove:Connect(RunService.PostSimulation, function()
		for _, session in DuoManager.GetAllSessions() do
			if session.IsResetting then
				continue
			end

			clampExploitVelocity(session.PlayerA)
			if session.PlayerB then
				clampExploitVelocity(session.PlayerB)
			end

			local shared = session.SharedAssembly
			local prim = if shared then shared.PrimaryPart else nil
			if shared and prim then
				local prevVel = lastObjectVelocities[session.SessionId] or Vector3.zero
				local curVel = prim.AssemblyLinearVelocity
				local deltaV = (curVel - prevVel).Magnitude
				lastObjectVelocities[session.SessionId] = curVel

				local threshold = (shared:GetAttribute("ShatterDeltaV") :: number?) or DEFAULT_SHATTER_DELTA_V
				if deltaV > threshold then
					lastObjectVelocities[session.SessionId] = nil
					session.SharedAssembly = nil
					local impactPos = prim.Position
					shared:Destroy()
					PhysicsTetherService.ObjectShatteredSignal:Fire({
						session = session,
						impactVelocity = deltaV,
						impactPosition = impactPos,
					})
					task.delay(1.5, function()
						DuoManager.RespawnDuoAtCheckpoint(session)
					end)
				end
			end
		end
	end)
end

return PhysicsTetherService
```

### 3.3 `StarterPlayerScripts/Controllers/AsymmetricCameraController.luau` (*Be My Camera* Interpolation)

Receives the 18-byte `buffer` over `UnreliableRemoteEvent` (`15–20 Hz`), interpolates smoothly using exponential decay `alpha = 1 - math.exp(-lambda * dt)`, and streams distant terrain via `RequestStreamAroundAsync`:

```luau
--!strict
-- StarterPlayerScripts/Controllers/AsymmetricCameraController.luau

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local CFrameBuffer = require(ReplicatedStorage.Shared.Utils.CFrameBuffer)

local SMOOTHING_LAMBDA = 15
local STREAM_INTERVAL = 0.75

local AsymmetricCameraController = {}

local isCameraOverridden = false
local targetCFrame: CFrame? = nil
local currentCFrame: CFrame? = nil
local lastStreamRequest = 0

function AsymmetricCameraController.EnableRemoteView(enabled: boolean)
	isCameraOverridden = enabled
	local cam = Workspace.CurrentCamera
	if not cam then
		return
	end
	if enabled then
		cam.CameraType = Enum.CameraType.Scriptable
		currentCFrame = cam.CFrame
	else
		cam.CameraType = Enum.CameraType.Custom
		targetCFrame = nil
		currentCFrame = nil
	end
end

function AsymmetricCameraController.OnCameraPacket(buf: buffer)
	local decoded = CFrameBuffer.unpack(buf)
	if decoded then
		targetCFrame = decoded
	end
end

function AsymmetricCameraController.Init() end

function AsymmetricCameraController.Start()
	RunService:BindToRenderStep("CoopAsymmetricCam", Enum.RenderPriority.Camera.Value + 1, function(dt: number)
		if not isCameraOverridden or not targetCFrame then
			return
		end
		local cam = Workspace.CurrentCamera
		if not cam then
			return
		end

		local cur = currentCFrame or targetCFrame
		local alpha = 1 - math.exp(-SMOOTHING_LAMBDA * dt)
		local nextCF = cur:Lerp(targetCFrame, alpha)
		currentCFrame = nextCF
		cam.CFrame = nextCF

		if Workspace.StreamingEnabled and (os.clock() - lastStreamRequest) >= STREAM_INTERVAL then
			lastStreamRequest = os.clock()
			task.spawn(function()
				pcall(function()
					Players.LocalPlayer:RequestStreamAroundAsync(nextCF.Position)
				end)
			end)
		end
	end)
end

return AsymmetricCameraController
```

---

## 4. Level Design Metrics & Viral Co-op Checklist

Standard R15 spatial reference (`JumpHeight ~= 7.2` studs, `WalkSpeed = 16`, `JumpDist ~= 12.5`):

| Act / Stages | Theme | Spatial & Physics Parameters |
|---|---|---|
| **Act I (1–10)** | Synchronized Basics | Platforms `12x12`, gaps `6–8` studs, `RopeLength = 14`, `ShatterDeltaV = 35 studs/s`. |
| **Act II (11–25)** | Momentum & Launching | `2.5`-stud beams, pendulum swings `16–20` studs, moving platforms `T = 4s`. |
| **Act III (26–40)** | Fragility & Asymmetry | `ShatterDeltaV = 22 studs/s`, lateral `VectorForce` wind, `1.5s` collapsing tiles. |
| **Act IV (41–50+)** | The Gauntlet | Timing windows `<= 0.8s`, multi-step launch chains, partial fall ledges. |

- **Sightline Rule:** Always keep the next checkpoint beacon visible from the current platform (critical in *Be My Camera*).
- **Recovery Ledges:** Place lower rescue ledges below Act II+ paths so a dangling partner triggers a 5-second rescue rather than instant void death.
- **Checkpoint Spacing:** Target **30–60s** ideal traversal per checkpoint (never > 90s).
- **Retention & Viral Hooks:** Zero-text visual metaphors (chain / glass pane / blindfold icon), `SocialService:PromptGameInvite()`, `PartnerCheckpoints` per friend (`<= 20` cached partners), and **Targeted Sabotage Fallback Token** in `ProcessReceipt` if the target leaves mid-checkout.
