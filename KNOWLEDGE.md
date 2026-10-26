# SYNTAXIS HUB (`qualcom`) - PROJECT KNOWLEDGE BASE

> **Notice:** This document is the living knowledge repository for the Syntaxis Hub project. It contains all architectural decisions, reverse-engineered game mechanics, remote endpoints, algorithm designs, executor compatibility rules, priority systems, and known issues. **Every time new knowledge is discovered, implemented, or refactored, this document must be updated.**

---

## Table of Contents
1. [Project Overview & Multi-Repository Architecture](#1-project-overview--multi-repository-architecture)
2. [Platform & Universal Executor Compatibility Layer (`UniversalCompat`)](#2-platform--universal-executor-compatibility-layer-universalcompat)
3. [Client Engine Hooking & Game Module Architecture](#3-client-engine-hooking--game-module-architecture)
4. [Complete Network Remotes Catalog (93 Total)](#4-complete-network-remotes-catalog-93-total)
5. [Valuation Formulas, Scoring & Rarity Tiers](#5-valuation-formulas-scoring--rarity-tiers)
6. [Automation Engines & Core Subsystems](#6-automation-engines--core-subsystems)
   - 6.1 [Stealing & Delivery Engine](#61-stealing--delivery-engine)
   - 6.2 [Egg Placement & Incubation-Aware Hatching](#62-egg-placement--incubation-aware-hatching)
   - 6.3 [Pet Management, Satchel & Fusion Chamber](#63-pet-management-satchel--fusion-chamber)
   - 6.4 [Treadmill & Homestead Upgrades](#64-treadmill--homestead-upgrades)
   - 6.5 [Halloween Dimension & World Automations](#65-halloween-dimension--world-automations)
   - 6.6 [Character Integrity, GodMode & Anti-Ragdoll](#66-character-integrity-godmode--anti-ragdoll)
7. [Unified Priority System & Concurrency Architecture](#7-unified-priority-system--concurrency-architecture)
8. [Anti-Cheat Detection Vectors & Hardening Guidelines](#8-anti-cheat-detection-vectors--hardening-guidelines)
9. [Commit History & Version Milestones](#9-commit-history--version-milestones)

---

## 1. Project Overview & Multi-Repository Architecture

### 1.1 Repositories & Distribution
- **Main Automation Script:** [`tenobarillas-ship-it/sae`](https://github.com/tenobarillas-ship-it/sae)
  - Primary script entry: `qualcom` (Luau script executed via standard Roblox `loadstring`).
  - Standalone loader:
    ```lua
    loadstring(game:HttpGet("https://raw.githubusercontent.com/tenobarillas-ship-it/sae/main/qualcom"))()
    ```
- **Modular GUI Library:** [`tenobarillas-ship-it/GUILib`](https://github.com/tenobarillas-ship-it/GUILib)
  - Liquid Glass iOS-inspired UI library cleanly separated from automation logic.
  - Modular distribution: `src/` (`Window`, `Components`, `Panels`, `Themes`, `Animations`, `Utils`, `Modals`).
  - Bundled single-request loader: `SyntaxisLib.lua`.
    ```lua
    local SyntaxisLib = loadstring(game:HttpGet("https://raw.githubusercontent.com/tenobarillas-ship-it/GUILib/main/SyntaxisLib.lua"))()
    ```
- **Reference Repository (Private):** [`tenobarillas-ship-it/friend`](https://github.com/tenobarillas-ship-it/friend)
  - Houses the monolithic 49,217-line reference script `chilihubstealanegg.lua`.
  - Used strictly for reverse engineering, remote discovery, and algorithm comparison.

### 1.2 Local Workspace & Settings Persistence
- **Local Working File:** `qualcom_clean.lua` (4,479 lines / 171 KB).
- **Settings Files (on executor):**
  - `SyntaxHubSettings.json` / `SyntaxHub_Config.json`: Persists UI states, toggles, filter dropdowns, and custom priority rankings via `readfile` / `writefile`.
  - `ChilliLibrary/SAE_Priority.json` (Reference implementation format).

---

## 2. Platform & Universal Executor Compatibility Layer (`UniversalCompat`)

### 2.1 Platform Breakdown: Mobile vs PC Executors
- **Delta Mobile (Android / iOS):**
  - Pure touchscreen environment (`UserInputService.TouchEnabled = true`, `KeyboardEnabled = false`, `MouseEnabled = false`).
  - Native C-level closures for `fireproximityprompt(prompt, 0)` and `firetouchinterest(part1, part2, 0/1)`.
  - **Critical Mobile Rule:** Calling `VirtualInputManager:SendKeyEvent` or `SendMouseButtonEvent` on mobile devices dispatches synthetic PC hardware keystrokes/clicks with impossible device flags, raising engine exceptions or triggering immediate anti-cheat detection kicks.
- **PC Executors (Wave, Synapse Z, MacSploit, Solara, Celery):**
  - Keyboard and mouse environments where standard C-closures must be invoked with proper safety wrappers.

### 2.2 Universal Compatibility & Polyfill Layer (`UniversalCompat`)
All script automation modules route through `UniversalCompat` rather than making raw executor calls. It is engineered to be **100% Touch & Mobile-Safe** with zero synthetic input flags:

1. **`UniversalCompat.FireProximityPrompt(prompt)`**:
   - Matches the friend reference implementation (`chilihubstealanegg.lua` line 30064):
     ```lua
     local function FirePrompt(prompt)
         if prompt and type(fireproximityprompt) == "function" then
             pcall(function()
                 prompt.HoldDuration = 0
                 fireproximityprompt(prompt)
             end)
         end
     end
     ```
   - **Zero Property Tampering:** Never modifies `MaxActivationDistance` or `RequiresLineOfSight` on server-replicated prompts (which causes client/server property desync and triggers anti-tamper kicks).
   - **Zero VirtualInputManager:** Completely eliminates `VirtualInputManager:SendKeyEvent` simulation, avoiding synthetic keyboard event injection on touchscreen devices (Delta Android/iOS).
   - **Legitimate Physical Positioning:** The player character is positioned within interaction reach ($\le 4$ studs) before firing, ensuring prompts trigger naturally within server distance boundaries.

2. **`UniversalCompat.FireTouch(part1, part2)`**:
   - **Method A:** Dual-stage `firetouchinterest(part1, part2, 0)` $\rightarrow$ `task.wait(0.01)` $\rightarrow$ `(part1, part2, 1)`.
   - **Method B:** Fallback ensures `CanTouch = true` on participating BaseParts.
   - Works uniformly across PC executors and mobile executors without raising execution exceptions.

3. **`UniversalCompat.ClickButton(btn)`**:
   - Pure event dispatching matching friend source (line 37655):
     1. `firesignal(btn.Activated)` / `firesignal(btn.MouseButton1Click)`
     2. `getconnections(btn.Activated)` loop invoking `c:Fire()` or `c.Function()`
     3. `getconnections(btn.MouseButton1Click)` loop invoking `c:Fire()` or `c.Function()`
   - **Zero MouseButtonEvent:** Avoids `VirtualInputManager:SendMouseButtonEvent` which fails or triggers security flags on pure touch environments where `UserInputService.MouseEnabled == false`.

### 2.3 Safe Container Parenting (`GetSafeParent`)
Roblox executors have varying levels of GUI protection and context permissions:
```lua
local safeContainer = nil
if typeof(gethui) == "function" then
    local ok, res = pcall(gethui)
    if ok and typeof(res) == "Instance" then safeContainer = res end
end
if not safeContainer and typeof(get_hidden_gui) == "function" then
    local ok, res = pcall(get_hidden_gui)
    if ok and typeof(res) == "Instance" then safeContainer = res end
end
if not safeContainer then
    safeContainer = Services.CoreGui:FindFirstChild("RobloxGui") or Services.CoreGui
end
```

---

## 3. Client Engine Hooking & Game Module Architecture

### 3.1 Intercepted Game Modules
Instead of scraping 3D models with expensive spatial raycasts or UI text labels, Syntaxis Hub hooks directly into the game's replicated client modules:

| Module Identifier | Location in `ReplicatedStorage` | Extracted State / Purpose |
| :--- | :--- | :--- |
| `EggState` | `ReplicatedStorage.Client.EggState` | Field egg positions, nest records, ownership, grab states (`ReadFieldEggs`). |
| `PlotState` | `ReplicatedStorage.Client.PlotState` | Base boundaries, plot IDs, pen center coordinates (`ResolvePlot`). |
| `Assets` | `ReplicatedStorage.Data.Assets` | Pet master catalog (`Directory`), base income rates, mesh assets. |
| `Mutations` | `ReplicatedStorage.Shared.Modules.Mutations`| Mutation identifiers, essence crafting formulas, multiplier maps. |
| `Rarity` | `ReplicatedStorage.Data.Rarity` | Rarity rankings, display colors, drop tier thresholds. |
| `Save` | `ReplicatedStorage.Data.Save` | Local inventory snapshot, currency balances, index progression (`Get`, `Peek`). |
| `Bases` | `ReplicatedStorage.Data.Bases` | Max pen capacity, upgrade tier costs, speed power requirements. |
| `HalloweenPets` | `ReplicatedStorage.Data.HalloweenPets` | Seasonal pet specs, model-to-id mapping, event egg sources (`Specs`). |

---

## 4. Complete Network Remotes Catalog (93 Total)

### 4.1 Stealing, Carrying & Field Actions
| Remote Name | Type | Direction | Payload & Technical Usage |
| :--- | :---: | :---: | :--- |
| `RF/EggWorld/AskCarryFieldEgg` | RF | Client $\rightarrow$ Server | `(eggUid)`: Picks up field egg into hands. |
| `RF/EggWorld/AskFieldEggDrop` | RF | Client $\rightarrow$ Server | `("PlayerRequest")`: Drops currently held egg to ground. |
| `RF/EggWorld/AskFieldEggCollect`| RF | Client $\rightarrow$ Server | `(eggUid)`: Deposits egg into base nest upon crossing home line. |
| `RE/RigSync/Refresh` | RE | Server $\rightarrow$ Client | Syncs character position. Action `"Relocate"` indicates rollback. |

### 4.2 Egg Placement & Hatching
| Remote Name | Type | Direction | Payload & Technical Usage |
| :--- | :---: | :---: | :--- |
| `RF/EggWorld/AskPlaceEgg` | RF | Client $\rightarrow$ Server | `({Uid = uid, LocalCFrame = cframe})`: Places egg onto plot pen slot. |
| `RF/EggWorld/AskHatch` | RF | Client $\rightarrow$ Server | `(uid)`: Starts hatching animation on placed egg. |
| `RF/EggWorld/AskFinishHatch` | RF | Client $\rightarrow$ Server | `(uid)`: Completes hatch, claims new pet into satchel. |
| `RF/PenRoster/AskWearLimit` | RF | Client $\rightarrow$ Server | `()`: Queries maximum allowable equipped pets + placed eggs. |

### 4.3 Pet Satchel, Fusion & Favoriting
| Remote Name | Type | Direction | Payload & Technical Usage |
| :--- | :---: | :---: | :--- |
| `RF/Haul/WearBest` | RF | Client $\rightarrow$ Server | `()`: Server equips highest-earning pets from satchel. |
| `RF/Fusery/LoadPet` | RF | Client $\rightarrow$ Server | `(slotIndex, petUid)`: Inserts pet into fusion chamber. |
| `RF/Fusery/BeginFuse` | RF | Client $\rightarrow$ Server | `()`: Starts fusion process. |
| `RF/Fusery/FinishReveal` | RF | Client $\rightarrow$ Server | `()`: Claims fused result. |
| `RF/Fusery/EjectPet` | RF | Client $\rightarrow$ Server | `(slotIndex)`: Ejects loaded pet. |
| `RE/PetSatchel/SellSelection` | RE | Client $\rightarrow$ Server | `({petUids})`: Liquidates selected pets for money. |
| `RE/PetSatchel/WriteFavourite`| RE | Client $\rightarrow$ Server | `(petUid, isFav)`: Toggles lock/favorite status on pet. |

### 4.4 Treadmill & Homestead Upgrades
| Remote Name | Type | Direction | Payload & Technical Usage |
| :--- | :---: | :---: | :--- |
| `RF/Treadmill/AskWearStill` | RF | Client $\rightarrow$ Server | `()`: Mounts base treadmill for speed generation. |
| `RF/Treadmill/AskDoff` | RF | Client $\rightarrow$ Server | `()`: Dismounts treadmill. |
| `RF/Base/AskUpgrade` | RF | Client $\rightarrow$ Server | `()`: Upgrades base plot tier (expands egg capacity). |
| `RF/Treadmill/AskUpgrade` | RF | Client $\rightarrow$ Server | `()`: Upgrades treadmill tier (increases speed multiplier). |

### 4.5 World Events & Minigames
| Remote Name | Type | Direction | Payload & Technical Usage |
| :--- | :---: | :---: | :--- |
| `RE/ShootingStar/Catch` | RE | Client $\rightarrow$ Server | `(starModel)`: Collects landed star. |
| `RF/TrainEvent/Action` | RF | Client $\rightarrow$ Server | `("Board")`: Boards the Phantom Express ghost train. |
| `RF/GhostDogQuest/Talk` | RF | Client $\rightarrow$ Server | `(stageId)`: Advances Ghost Dog questline dialog. |
| `RF/GhostDogQuest/Claim` | RF | Client $\rightarrow$ Server | `(stageId)`: Claims Ghost Dog quest reward. |
| `RF/WispCompanion/Request` | RF | Client $\rightarrow$ Server | `(wispId)`: Fulfills wisp request with flowers/nectar. |

### 4.6 Halloween Dimension & Seasonal Systems
| Remote Name | Type | Direction | Payload & Technical Usage |
| :--- | :---: | :---: | :--- |
| `RF/HalloweenDimension/AskCarryEgg`| RF | Client $\rightarrow$ Server | Grabs event egg in Halloween Dimension. |
| `RF/HalloweenDimension/AskDropEgg` | RF | Client $\rightarrow$ Server | Drops event egg in Halloween Dimension to clear displacement accumulator. |
| `RF/HalloweenDimension/AskBuyEgg` | RF | Client $\rightarrow$ Server | Purchases Halloween egg using Candy Corn currency. |
| `RE/HalloweenDimension/MonsterHit` | RE | Client $\rightarrow$ Server | Registers weapon damage against Halloween monsters. |

---

## 5. Valuation Formulas, Scoring & Rarity Tiers

### 5.1 Egg Valuation Formula
Egg valuation uses the non-linear scale multiplier:
$$\text{Value} = \text{EarningRate} \times \text{Factor} \times \text{MutationBonus}$$
Where:
- If $\text{Scale} > 5.0$:
  $$\text{Factor} = \left(\frac{\text{Scale}}{5}\right)^{1.2} \times 19.637875755794113$$
- Else:
  $$\text{Factor} = \text{Scale}^{1.85}$$

### 5.2 Rarity Ranks Table
| Rank | Tier Name | Base Multiplier | Color Tag |
| :---: | :--- | :---: | :--- |
| **1** | Common | $1.0\times$ | Gray |
| **2** | Uncommon | $1.5\times$ | Green |
| **3** | Rare | $2.5\times$ | Blue |
| **4** | Epic | $4.0\times$ | Purple |
| **5** | Legendary | $7.5\times$ | Gold / Yellow |
| **6** | Mythic | $15.0\times$ | Red |
| **7** | Cosmic | $30.0\times$ | Cyan |
| **8** | Secret | $75.0\times$ | White / Sparkle |
| **9** | Eternal | $150.0\times$ | Orange Fire |
| **10** | Divine | $350.0\times$ | Rainbow |

---

## 6. Automation Engines & Core Subsystems

### 6.1 Stealing & Delivery Engine

#### Normal Dimension Target Areas:
Forest, Lake, Desert, Jungle, Snow, Volcano, Abyss Ocean, Prehistoric, Cosmic, Cherry Blossom, Titan Temple, Light Dark, Enchanted Forest.

#### Low-Speed Dynamic Hopping vs 4-Phase SafeCarry:
1. **Dynamic Hopping Mode (`executeCheckpointedHopsHome` in `qualcom`):**
   - Uses discrete altitude hops (`groundY = CONFIG.GroundAltitude + 1.0`).
   - Hop step size dynamically scales with player WalkSpeed:
     $$\text{stepSize} = (\text{playerSpeed} < 100) \mathbin{?} \max(34, \text{playerSpeed} \times 0.45) : \min(\text{baseStep}, \max(60, \text{playerSpeed} \times 0.75))$$
   - Eliminates blind mid-route dropping that caused low-speed players and PC executors to lose eggs on the conveyor.
2. **4-Phase SafeCarry (`SafeCarry.LineDropHome` & `Home` from ChiliHub):**
   - Computes two intermediate waypoints: `MidDrops = {0.33, 0.66}`.
   - Steps character to waypoint $\rightarrow$ calls `EggState.DropFieldEgg("PlayerRequest")` $\rightarrow$ waits `0.10s` (`MidRest`) $\rightarrow$ re-grabs with `EggState.CarryFieldEgg(uid)`.
   - **Why it succeeds:** The 0.10s drop resets the server's cumulative displacement counter to zero without triggering physics desync.

---

### 6.2 Egg Placement & Incubation-Aware Hatching
1. **Pen Proximity Enforcement:**
   - Must be within $< 35$ studs of base pen anchor (`PetArea`). Calling `AskPlaceEgg` from outside your pen triggers server placement verification failure (`CODE BAC-7516`).
2. **Tool Equipping Flow:**
   - Calls `EggState.WearEggTool(uid)`, awaits confirmation tool attachment, then invokes `RF/EggWorld/AskPlaceEgg`. Cleans up with `EggState.DoffEggTool()`.

---

### 6.3 Pet Management, Satchel & Fusion Chamber
- **Auto Sell:** Scans backpack via `Save.Get().PetInventory` and bulk-liquidates pets below target rarity using `RE/PetSatchel/SellSelection`.
- **Auto Fuse:** Auto-loads pets matching duplicate requirements, starts fusion, and claims rewards.
- **Auto Equip Best:** Invokes `RF/Haul/WearBest` to maximize earnings.

---

### 6.4 Treadmill & Homestead Upgrades
- Mounts base treadmill via `RF/Treadmill/AskWearStill`.
- Periodic worker checks upgrade timers and triggers `AskUpgrade` for base capacity and treadmill speed.

---

### 6.5 Halloween Dimension & World Automations

- **Auto Halloween Egg Steal (Friend Source Architecture — `chilihubstealanegg.lua` lines 30036–30973):**
  - **Zone Indexing & Spatial Filtering:**
    - Resolves BaseParts `Zone1` (Graveyard), `Zone2` (Haunted Swamps), and `Zone3` (Abandoned Manor) under `HalloweenDimension.Zones`.
    - Determines zone index using precise object-space bounding box calculations:
      $$\Delta X \le \frac{\text{Size.X}}{2} - \text{inset}, \quad \Delta Z \le \frac{\text{Size.Z}}{2} - \text{inset}$$
  - **Egg Valuation & Identification:**
    - Resolves raw egg instances to `Data.HalloweenPets.Specs` identifiers and `Data.Assets.Directory` records.
    - Computes egg value using non-linear scale valuation:
      $$\text{Factor} = (\text{Scale} > 5.0) \mathbin{?} \left(\frac{\text{Scale}}{5}\right)^{1.2} \times 19.637875755794113 : \text{Scale}^{1.85}$$
      $$\text{Value} = \text{EarningRate} \times \text{Factor}$$
    - Evaluates filters: `TargetAreas` (Zone matching), `MinRarity` (Rank matching), and internal `OrderIndex` priority.
  - **Approach & Grab Routine:**
    - When `InstantSteal = true`, steps character towards target egg using Heartbeat-clamped displacement (`StepSpeed`, default 40 studs/step).
    - Approaches within 3 studs, positions character at `eggPos + Vector3.new(0, 3, 0)`, and fires `StealPrompt` every 100ms via pure `fireproximityprompt`.
    - If grabbing fails after 3 attempts, blacklists egg for 30s (`BadEggs[egg] = os.clock() + 30`).
  - **SafeCarry Delivery Sequence:**
    - **Edge Stepping & Pullback Check:** Steps character to the outer boundary of `SafeZone`. Evaluates anti-cheat pullback up to 4 iterations (detecting if server displacement exceeds 100 studs).
    - **Drop & Regrab at Boundary:** Asynchronously calls `RF/HalloweenDimension/AskDropEgg` to reset cumulative displacement, waits for `HalloweenCarrying == false`, then re-grabs the egg at the edge.
    - **SafeZone Inset Velocity Glide:** Noclips character BaseParts (`CanCollide = false`), clamps target point inside SafeZone with inset (6 studs for Instant, 40 studs for Normal), and applies planar linear velocity towards target combined with vertical Y-correction:
      $$\mathbf{v}_y = (Y_{\text{ground}} - Y_{\text{root}}) \times 8$$
    - Restores BasePart collision and verifies delivery confirmation via `LocalPlayer:GetAttribute("HalloweenDelivered")` change.
- **Dimension Boundary Supervisor & Portal Manager:**
  - Prevents instant bounce-backs by verifying spatial bounding boxes (`workspace:FindFirstChild("HalloweenDimension")` and character CFrame bounds).
  - Teleports through portals with downward physics nudges (`Vector3.new(0, -0.5, 0)`) to guarantee broadphase collision registration.
- **Candy Cane Monster Farm:**
  - Identifies active monsters in `HalloweenDimension.Monsters`.
  - Cycles weapons via `GetWeaponList()` to bypass attack cooldowns.
  - Hovers at `HoverHeight = 12–14` studs and triggers swept touch collisions.
  - Cleanly yields via `HalloweenCombatManager:Abort()` when preempted.
- **Auto Pumpkin:**
  - Detects runaway pumpkins in `HalloweenDimension.RunawayPumpkins`.
  - Glides to pumpkin, grabs, and carries to the Altar.
  - Marked **non-interruptible** while `IsCarrying() = true` to prevent pumpkin dropping.
- **Live World Events:**
  - **Shooting Stars:** Intercepts falling `ShootingStarClient` models, tracks landing position, and fires `RE/ShootingStar/Catch`.
  - **Phantom Express:** Navigates to ghost train station platform and boards via `RF/TrainEvent/Action("Board")`.
  - **Ghost Dog Quest:** Executes complete 5-stage quest pipeline (`Talk` $\rightarrow$ `Cast` $\rightarrow$ `Dive` $\rightarrow$ `Surface` $\rightarrow$ `Claim`).
  - **Butterfly Bloom & Wisps:** Detects butterfly swarms in Enchanted Forest, collects nectar with equipped net, and fulfills wisp companion requests (`RF/WispCompanion/Request`).

---

### 6.6 Character Integrity, GodMode & Anti-Ragdoll
To prevent death loops, ragdoll exhaustion, and speed resets:
- Disables falling down and physics state transitions.
- Cleans ragdoll constraints only upon character initialization, explicitly ignoring tools to prevent breaking egg display attachments.

---

## 7. Unified Priority System & Concurrency Architecture

### 7.1 Hierarchy of Tasks
1. `Auto Steal` (Normal Dimension)
2. `Halloween Steal` (Halloween Dimension)
3. `Candy Cane Farm` (Halloween Dimension)
4. `Auto Pumpkin` (Halloween Dimension)
5. `Enchanted / Bloom Event` (Normal Dimension)
6. `Auto Upgrades` (Independent non-blocking)
7. `Auto Treadmill` (Normal Dimension Idle Fallback)

---

## 8. Anti-Cheat Detection Vectors & Hardening Guidelines

1. **Virtual Input Simulation vs Direct Remotes:**
   - **Vector:** Using `VirtualInputManager:SendKeyEvent` or `SendMouseButtonEvent` is monitored by Byfron/Hyperion on 64-bit clients and crashes/flags mobile delta clients.
   - **Mitigation:** Zero VIM. Use native `fireproximityprompt` and `firesignal` closures with legitimate character positioning.
2. **Carry Displacement Accumulation:**
   - **Vector:** Continuous velocity displacement $> 400$ studs while `isHoldingEgg() == true`.
   - **Mitigation:** Intermediate waypoint drop-and-regrab to reset cumulative server displacement.
3. **Chunk Size & Variable Limits:**
   - **Vector:** Exceeding 200 local variables in a single scope causes Luau compilation failure.
   - **Mitigation:** Keep all module state within tables. Current maximum scope locals: 48.
4. **`CODE BAC-7516` (Bittersweet Anti-Cheat Egg Tool & Placement Flag):**
   - **Trigger Signature:** Triggered when an egg is held or placed on plot, as evidenced in engine logs:
     ```text
     01:00:33 -- EggToolDisplay: [Debug] Attached egg tool display for <Uid>
     01:00:35 -- Server Kick Message: You have been removed for cheating, please remove any cheats to play | CODE BAC-7516; Metadata:
     01:00:35 -- PlacedEggRenderer: [Debug] Rendered placed egg <Uid> for owner <UserId>
     ```
   - **Root Cause:** The server-side anti-cheat (`BAC`) verifies player proximity, physics constraints, and displacement state during egg tool equipping and placement. If a character teleports, crosses map boundaries without resetting carry displacement accumulators, or thrashes constraints/Motor6Ds (e.g. continuous `applyGodMode` loops scanning descendants every 250ms), the server triggers a hard anti-cheat kick (`CODE BAC-7516`).
   - **Mitigation:**
     1. Stop continuous constraint destruction in loop timers; only invoke character configuration on initial spawn (`CharacterAdded`).
     2. Completely ignore instances under `Tool` during descendant constraint sweeps.
     3. Enforce legitimate pen proximity and waypoint-based drop-and-regrab before carrying an egg into plot pens.

---

## 9. Commit History & Version Milestones

| Commit SHA | Commit Title / Name | Description |
| :--- | :--- | :--- |
| `2f51383cda7eaf2f5ac12c6dc7cbab0245a4c917` | `Initial Clean Baseline` | Clean baseline prior to Halloween dimension integration. |
| `5a3e82b8260bc2602685225a6e79183c455bfb5e` | `fix(halloween): resolve dimension routing conflicts` | Initial dimension routing fixes and portal isolation. |
| `8bd71b0eaaad4a598be3528dc100fd2277caaa05` | `fix: auto upgrades, bloom event, low-speed delivery` | Removed plot proximity check from upgrades; fixed net claim. |
| `bca0e25f55161b1200145862f52d09c883589bf2` | `refactor(gui): integrate modular GUILib` | Decoupled ~2,500 lines of GUI boilerplate into external `GUILib`. |
| `e762c321a1038d6e7e3beb8b6b3185e009277f1e` | `fix(priority): resolve task hierarchy preemption` | Complete priority preemption & bidirectional dimension supervisor. |
| `bc4ca8615c00b895b78cbe33f89ea12dd7b9d10a` | `fix(executor-compat): universal polyfills for PC` | `UniversalCompat` layer for PC executors (prompts, touch, buttons) & safe low-speed delivery. |
| `ca9cbc44f19ece1217971ecd1c302ee307d89fd7` | `revert(qualcom): exact byte-for-byte restore to bc4ca86` | Reverted `qualcom` to stable working baseline commit. |
| `2925295fc800ed6081da4e3b827eb23395436f48` | `docs(knowledge): comprehensive knowledge base restore` | Reverted `KNOWLEDGE.md` to stable baseline before expanded documentation update. |
