# roblox-game — Antigravity & Claude Code Skill

> **Antigravity ile uyumludur.**

The ultimate Roblox game development skill for **Google Antigravity** and **Claude Code**. A self-contained, full-lifecycle development companion that serves all skill levels with deep `--!strict` Luau expertise, interactive game ideation, autonomous game building via MCP Studio integration, and comprehensive knowledge of game design, monetization, security, and performance optimization.

## Stats

- **17 deep reference files** | **10 game templates** | **8 guided workflows**
- `--!strict` Luau & modular `ModuleScript` Service-Controller architecture enforced by default
- 3-mode MCP Studio integration (full/standard/offline)

## Architecture

**Router + Reference Library** — lightweight `SKILL.md` dispatcher enforcing mandatory `--!strict` Luau, modular `ModuleScript` Service-Controller architecture, server-authoritative Client-Server security, and 3-layer Data management, with lazy-loaded deep knowledge.

```
roblox-game/
├── SKILL.md                              # Router + non-negotiable engineering charter
├── references/                           # 17 deep reference files
│   ├── modular-service-architecture.md   # Mandatory --!strict, Signal bus, RemoteGateway, 3-layer data
│   ├── luau-mastery.md
│   ├── architecture-patterns.md
│   ├── mcp-orchestration.md
│   ├── security-hardening.md
│   ├── datastore-persistence.md
│   ├── gui-systems.md
│   ├── performance-optimization.md
│   ├── monetization-systems.md
│   ├── combat-systems.md
│   ├── inventory-systems.md
│   ├── game-design-roblox.md
│   ├── tooling-ecosystem.md
│   ├── animation-vfx.md
│   ├── multiplayer-networking.md
│   ├── testing-patterns.md
│   └── sharp-edges.md                    # 19 severity-rated critical gotchas
├── templates/                            # 10 game templates
│   ├── game-scaffold.md                  # Universal --!strict modular base (any genre)
│   ├── genre-coop-physics-obby.md        # 2-player tethered/carried/asymmetric physics obby
│   ├── genre-elimination-survival.md     # Round-based trials (Death Penalty / Ink Game / Simon Says)
│   ├── genre-survival-expedition.md      # Co-op central-anchor survival/expedition (99 Nights / Drill)
│   ├── genre-simulator.md                # Pet Simulator style
│   ├── genre-tycoon.md                   # Retail Tycoon style
│   ├── genre-obby.md                     # Tower of Hell style
│   ├── genre-rpg.md                      # Blox Fruits style
│   ├── genre-horror.md                   # Doors style
│   └── genre-battle-royale.md            # Battle royale style
└── workflows/                            # 8 guided workflows
    ├── game-ideator.md                   # Interactive 3-phase ideation & viral pitch builder
    ├── new-game.md
    ├── debug-loop.md
    ├── performance-audit.md
    ├── security-audit.md
    ├── publish-checklist.md
    ├── monetization-audit.md
    └── code-review.md
```

## Features

### MCP Studio Integration (3 Modes)

| Mode | Server | Tools | Capabilities |
|------|--------|-------|-------------|
| **Full** | boshyxd/robloxstudio-mcp | 39 | Project exploration, autonomous building, iterative debugging, bulk ops, Creator Store |
| **Standard** | Roblox/studio-rust-mcp-server | 6 | Code execution, model insertion, console reading, play mode control |
| **Offline** | None | 0 | Pure Luau code generation with placement instructions |

Auto-detects which server is available and adapts all workflows accordingly.

### Genre Templates

Each template includes complete, production-ready Luau code for every core system:

- **Co-op Physics Obby** — `DuoManager`, `PhysicsTetherService` (`PostSimulation` velocity watchdog), 18-byte `CFrameBuffer` + `UnreliableRemoteEvent` camera sync, atomic checkpoint reset
- **Elimination Survival** — 7-state Global FSM, RuleCommandEngine, Ragdoll/Execution FX, Spectator Sabotage, `RoundTrove` cleanup
- **Survival / Expedition** — Co-op central-anchor loop (Campfire/Drill), fuel/vitals, Boss Walls, progression math, MemoryStore rejoin
- **Simulator** — Collection, pet system (hatching + rarity), zones, upgrades, prestige/rebirth
- **Tycoon** — Plot claiming, droppers/collectors, button pad purchasing, income, tiers
- **Obby** — Checkpoints, 5 obstacle types, timer/speedrun, leaderboards
- **RPG** — Character stats + leveling, quest system, NPC AI, dialog trees, zones
- **Horror** — Atmosphere control, monster AI (5-state FSM), event sequencer, jumpscares, flashlight
- **Battle Royale** — Match lifecycle, shrinking storm, loot spawning, elimination tracking, spectator

### Guided Workflows

- **Game Ideator & Builder** — 7-genre selector, 3 viral pitch generator with thumbnail/monetization hooks, and full modular build
- **New Game** — Build a complete game from a description in 8 steps
- **Debug Loop** — Iterative error detection and fixing (max 5 attempts)
- **Performance Audit** — Part count, script, memory, and network analysis
- **Security Audit** — RemoteEvent validation, client trust, data exposure review
- **Publish Checklist** — 60+ item pre-launch verification across 10 categories
- **Monetization Audit** — GamePass, DevProduct, Premium, and Ad optimization
- **Code Review** — A-F quality rating with specific fixes

### Deep References

17 reference files covering every aspect of Roblox development:
- Modular service architecture (`--!strict`, `Init`/`Start`, `Signal` `EventBus`, `RemoteGateway`, 3-layer Data separation)
- Luau language mastery with `--!strict` type system and OOP patterns
- Game architecture (services, client-server, modules, frameworks)
- Security hardening (never trust client, validation, rate limiting, `SetNetworkOwner(nil)`, `DotProduct` LoS)
- DataStore & MemoryStore persistence (ProfileService, session locking, migration, rejoin state)
- And 12 more specialized topics

### Sharp Edges

19 severity-rated gotchas that save you from disaster (Critical → Low):
- DataStore data loss, client-side currency, ProcessReceipt bugs, hidden map secret leaks (ESP)
- Memory leaks, RemoteEvent flooding, BindToClose timeout, `tick()` clock desync & lag-switch abuse
- Vehicle/carried-object ownership, tethered velocity flings, StreamingEnabled desync, 60 Hz `CFrame` spam, constraint reset flings
- And 6 more common pitfalls with Luau code fixes

## Usage

The skill is automatically invoked when you mention Roblox, Luau, game development, or any related keyword.

**Example prompts:**
- "Build me a co-op survival game with campfire fuel and night raids"
- "Build me a pet simulator with 3 zones"
- "How do I save player data and rejoin state in Roblox?"
- "Review my game's security"
- "Optimize this game for mobile"
- "Add a shop system with GamePasses"

## Installation

Antigravity ile uyumludur — global skill dizini: `~/.gemini/config/skills/roblox-game/` (veya Claude Code için `~/.claude/skills/roblox-game/`).

```bash
git clone https://github.com/mxxnthefinest/roblox-game-skill.git ~/.gemini/config/skills/roblox-game
```

## Design Documents

See `docs/plans/` for the full design document and 32-task implementation plan.
