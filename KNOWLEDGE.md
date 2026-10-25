# SYNTAXIS HUB (`qualcom`) - PROJECT KNOWLEDGE BASE

> **Notice:** This document is the living knowledge repository for the Syntaxis Hub project. It contains all architectural decisions, reverse-engineered game mechanics, remote endpoints, algorithm designs, executor compatibility rules, priority systems, anti-cheat mitigations, and known issues. **Every time new knowledge is discovered, implemented, or refactored, this document must be updated.**

---

## 1. Project Overview & Architecture

### Repositories & Distribution
- **Main Automation Script:** [`tenobarillas-ship-it/sae`](https://github.com/tenobarillas-ship-it/sae)
  - Primary script entry: `qualcom` (Luau script executed via standard Roblox loadstring).
  - Standalone loader:
    ```lua
    loadstring(game:HttpGet("https://raw.githubusercontent.com/tenobarillas-ship-it/sae/main/qualcom"))()
    ```
- **Modular GUI Library:** [`tenobarillas-ship-it/GUILib`](https://github.com/tenobarillas-ship-it/GUILib)
  - Liquid Glass iOS-inspired UI library cleanly separated from the automation logic.
  - Sidebar uses a dynamic `ScrollingFrame` with automatic Y-resizing to accommodate unlimited categories and tabs without layout clipping or text overlap.
  - Bundled single-request loader: `SyntaxisLib.lua`.
    ```lua
    local SyntaxisLib = loadstring(game:HttpGet("https://raw.githubusercontent.com/tenobarillas-ship-it/GUILib/main/SyntaxisLib.lua"))()
    ```
- **Reference Friend Codebase (Private):** [`tenobarillas-ship-it/friend`](https://github.com/tenobarillas-ship-it/friend)
  - Reference script: `chilihubstealanegg.lua` (1.8MB reverse-engineered codebase containing all game remote APIs, event logic, and asset definitions).

### Local Workspace & Sync
- **Local Workspace Working File:** `qualcom_clean.lua`
- **Settings File (on executor):** `SyntaxHubSettings.json` (persisted via `writefile` / `readfile`).

---

## 2. Anti-Cheat (UGI) Mitigation & Executor Stealth

### Anti-Cheat Heuristics in Steal An Egg
The game employs a custom client-side anti-cheat ("UGI") in tandem with CoreScript integrity monitoring:
1. **Synthetic Input Hook Detection:**
   - Calling `VirtualUser:CaptureController()` or `VirtualUser:ClickButton2()` emits synthetic input signals from the engine root that immediately trigger detection heuristics.
   - Calling `VirtualInputManager:SendKeyEvent()` or `SendMouseButtonEvent()` generates synthetic OS-level keystroke packets that flag the client and cause an immediate kick (`"You were kicked from this experience"`).
2. **Stealth Remediation Implemented:**
   - **Zero Synthetic Input Services:** Completely removed all references to `VirtualUser` and `VirtualInputManager` from `qualcom` and `UniversalCompat`.
   - **Connection-Only Anti-AFK:** Disables idle disconnection purely by iterating `getconnections(LocalPlayer.Idled)` and calling `conn:Disable()` or disconnecting the handler. This operates entirely within memory without firing synthetic inputs, remaining 100% undetected.
   - **Stealth Proximity Prompts:** Proximity prompts are triggered via standard `fireproximityprompt(prompt, 0)` with zero hold duration and line-of-sight bypass, avoiding synthetic keypress injection.

---

## 3. The 13-Task Movement Priority Arbitrator & Dimension Coordination

### Strict Movement-Task Specialization
In accordance with system design requirements, the Priority Panel is strictly specialized as a **Movement-Task Manager**. Only automations requiring character physical movement, positioning, teleportation, or route navigation participate in the Priority Panel.

Stationary background tasks (such as Auto Upgrades, Auto Claim Mastery, Auto Equip Best, Auto Sell, Auto Fuse, Auto Mutate, and Seasonal Store interactions) run asynchronously via `WorkerScheduler` and **never compete for or block character movement**.

### The 13 Canonical Movement-Dependent Automations:
| Rank | Canonical Name | Target Dimension | Character Movement Behavior |
| :---: | :--- | :---: | :--- |
| **1** | `"Auto Steal"` | Normal | Outbound glide/hop along ground to target egg, instant grab, lane hop return with mid-route checkpoints, boundary checkpoint, safe corridor micro-hops, and delivery hopper drop. |
| **2** | `"Auto Steal Halloween Eggs"` | Halloween | Dimension transfer into Halloween, navigation across Graveyard/Swamps/Manor, egg grab, boundary return, and delivery. |
| **3** | `"Auto Catch Shooting Star"` | Normal | Detects falling `ShootingStarClient` in workspace, paths to landing coordinate, and invokes `RE/ShootingStar/Catch`. |
| **4** | `"Auto Capture Event Egg"` | Normal | Continuously tracks and chases roaming event egg in workspace, closing distance to interact or equip. |
| **5** | `"Auto Butterfly Bloom"` | Normal | Paths to spawned butterflies in Enchanted Forest, collects nectar, and fulfills wisp quests. |
| **6** | `"Candy Cane Monster Farm"` | Halloween | Navigates to nearest seasonal monster in Halloween dimension, equips combat gear, and executes combat sequence. |
| **7** | `"Auto Pumpkin"` | Halloween | Paths to runaway pumpkins in Halloween dimension, grabs and carries them to delivery drop-off. |
| **8** | `"Auto Ghost Dog Quest"` | Halloween | Navigates character directly to Ghost Dog NPC in Halloween dimension to fulfill quest stages (Talk, Cast, Dive, Surface, Claim). |
| **9** | `"Auto Fishing"` | Halloween | Navigates character to Halloween fishing waters/dock, equips rod tool, casts line, and reels catches. |
| **10** | `"Auto Phantom Express"` | Halloween | Navigates character to the train station platform in Halloween dimension and boards the train. |
| **11** | `"Auto Vacuum Ghouler Boss"` | Halloween | Navigates within combat radius of GhostBoss/GhoulerBoss and engages vacuum suction remote. |
| **12** | `"Auto Use Admin Treadmill"` | Normal | Detects active `AdminTreadmill` spawn in workspace, travels to treadmill platform, and mounts belt for high multiplier. |
| **13** | `"Auto Treadmill"` | Normal | Default idle task. Navigates to base treadmill when no higher-priority movement tasks are active. |

---

## 4. Automation Features & Architecture

### Stealing & Egg Management
- **Specific Egg Targeting:** Multi-select dropdown filtering by egg species.
- **Steal Missing Index Eggs:** Queries player's Codex/Index to prioritize stealing uncollected egg breeds.
- **Auto Steal Minigame Egg:** Automates egg stealing during Red Light Green Light (`workspace:GetAttribute("Event_RedLightGreenLight")`) and T-Rex Run.
- **Auto Capture Event Egg / Take From Holder:** Grabs roaming world event eggs; if held by another player, snatches it and elevates to a safe holding altitude (+45 studs) to prevent counter-steals.
- **Auto Catch Shooting Star:** Detects `workspace.ShootingStarClient` spawns, tweens to the landing zone, and claims falling stars via `RE/ShootingStar/Catch`.

### Halloween & Dimension Events
- **Halloween Auto Steal & Instant Steal:** Remote-assisted and teleport-assisted stealing inside the Halloween Dimension.
- **Auto Buy Halloween Dimension Egg:** Purchases event eggs when soul/candy thresholds are reached.
- **Ghost Dog Quest:** Automates Ghost Dog NPC interaction: `RF/GhostDogQuest/Talk`, `Dive`, `Surface`, and `Claim`.
- **Auto Fishing:** Auto-casts (`RF/Fishing/Cast`) with configurable luck settings ("Max", "High", "Normal") and reels in catches (`RF/Fishing/Reel`).
- **Phantom Express & Ghouler Boss:** Boards the ghost train (`RF/TrainEvent/Action` -> `Board`) and targets the Ghouler boss with suction weaponry (`RE/GhostBoss/Suction`).
- **Auto Trade Souls:** Trades collected souls at dimension stalls.

### Pet & Egg Placement Management
- **Auto Place Egg:** Scans base plot pen grids and backpack eggs. Places eggs into empty slots (`RF/EggWorld/AskPlaceEgg`) sorted by Highest Value, Biggest, or Backpack Order, with minimum income thresholds.
- **Auto Hatch:** Automatically hatches inventory eggs (`RF/EggWorld/AskHatch`).
- **Egg Predictor:** Analyzes drop tables and rolls to anticipate hatching odds.
- **Auto Equip Best:** Calls `RF/Haul/WearBest` to recalculate and equip top income/multiplier pets.
- **Auto Sell Pets & Eggs:** Automatically liquidates low-rarity pets and unplaced eggs below configured rarity limits, while strictly preserving mutated pets (`KeepMutatedPets`) and favorites.
- **Auto Fuse Machine:** Loads triplets of identical pets into the fusion station (`RF/Fusery/LoadPet`), confirms briefing, begins fusion, and reveals the upgraded pet (`RF/Fusery/FinishReveal`).
- **Auto Favorite:** Locks high-tier and mutated pets via `RE/PetSatchel/WriteFavourite` to prevent accidental loss.

### Training & Movement
- **Auto Treadmill & Stay On Treadmill:** Automatically navigates to player's plot treadmill and enforces position on the belt to prevent falling off.
- **Auto Use Admin Treadmill:** Finds and trains on restricted admin/VIP event treadmills for accelerated walkspeed gain.
- **Speed Boost:** Safely multiplies Humanoid WalkSpeed with slider configuration (16-300).
- **Infinite Jump:** Listens to `UserInputService.JumpRequest` and sets state to `Jumping` repeatedly.

### Character Protections & Utilities
- **Anti-Trap Immunity:** Scans `CollectionService:GetTagged("PlacedTrap")` and evades player-placed traps.
- **Anti-Ragdoll Recovery:** Detects ragdoll/knockdown states and immediately resets humanoid state to normal standing/running.
- **Stealth Anti-AFK:** Disconnects `LocalPlayer.Idled` connection without any synthetic input, completely safe from anti-cheat kicks.
- **Instant Prompts:** Eliminates ProximityPrompt hold duration across all interactables.
- **Auto Rejoin On Disconnect:** Listens to CoreGui error prompts and reconnects automatically upon server disconnect.

### Visuals & Optimization
- **Egg ESP:** Renders color-coded BillboardGuis showing Egg Name, Distance, and Income Value with filters for Fixed Size, Own Base, Other Base, and Min Rarity.
- **Guard & Player ESP:** Highlights base guards and nearby players with distance indicators.
- **Optimizer:** Strips shadows, particles, fog, and complex textures to maximize FPS.
- **Disable 3D Render:** Pauses 3D world rendering via `RunService:Set3dRenderingEnabled(false)` for ultra-low background CPU/GPU usage.
- **FPS Cap:** Configurable framerate limit via `setfpscap`.
- **Hide Game UI:** Toggles CoreGui elements to reduce HUD rendering overhead.
- **Farm HUD:** Sleek draggable live stats overlay displaying current task, eggs stolen, earnings, and FPS.

---

## 5. Reverse-Engineered Remotes & Endpoints Directory

| Category | Remote Endpoint | Type | Purpose |
| :--- | :--- | :--- | :--- |
| **Egg Placement** | `RF/EggWorld/AskPlaceEgg` | RemoteFunction | Places egg into pen slot `{ Uid, LocalCFrame }` |
| **Hatching** | `RF/EggWorld/AskHatch` | RemoteFunction | Hatches egg by UID |
| **Equipment** | `RF/Haul/WearBest` | RemoteFunction | Recalculates and equips best pets |
| **Pet Selling** | `RE/PetSatchel/SellSelection` | RemoteEvent | Sells batch of pet UIDs |
| **Pet Locking** | `RE/PetSatchel/WriteFavourite` | RemoteEvent | Sets favorite status on pet UID |
| **Fusion** | `RF/Fusery/LoadPet` | RemoteFunction | Loads pet into fusion machine |
| **Fusion** | `RF/Fusery/ConfirmBriefing` | RemoteFunction | Confirms fusion recipe |
| **Fusion** | `RF/Fusery/BeginFuse` | RemoteFunction | Starts fusion process |
| **Fusion** | `RF/Fusery/FinishReveal` | RemoteFunction | Claims fusion result |
| **Shooting Star**| `RE/ShootingStar/Catch` | RemoteEvent | Claims fallen shooting star |
| **Ghost Dog** | `RF/GhostDogQuest/Talk` | RemoteFunction | Initiates Ghost Dog quest |
| **Ghost Dog** | `RF/GhostDogQuest/Dive` | RemoteFunction | Starts ghost dive |
| **Ghost Dog** | `RF/GhostDogQuest/Surface` | RemoteFunction | Surfacing diver |
| **Ghost Dog** | `RF/GhostDogQuest/Claim` | RemoteFunction | Claims quest reward |
| **Fishing** | `RF/Fishing/Cast` | RemoteFunction | Casts fishing rod with luck parameter |
| **Fishing** | `RF/Fishing/Reel` | RemoteFunction | Reels in caught fish |
| **Train Event** | `RF/TrainEvent/Action` | RemoteFunction | Boards Ghost Train (`Action = "Board"`) |
| **Ghouler Boss** | `RE/GhostBoss/Suction` | RemoteEvent | Fires vacuum suction at Ghost Boss |
| **Banjo Cricket**| `RF/BanjoCricket/AskClaim` | RemoteFunction | Claims Banjo Cricket musical buff |
| **Halloween Buy**| `RF/HalloweenDimension/AskBuyEgg` | RemoteFunction | Buys Halloween event shop eggs |

---

## 6. Known Initialization Issues & Scope Constraints

### "Attempt to index nil with 'PromptShown'" Fix
- **Symptom:** Script loading halted on line 3342 with `attempt to index nil with 'PromptShown'` inside `UtilityManager:Initialize()`.
- **Root Cause:** The top-level `Services` dictionary did not define `ProximityPromptService`, causing `Services.ProximityPromptService` to return `nil`.
- **Permanent Solution:**
  1. Explicitly registered `ProximityPromptService`, `Lighting`, `CoreGui`, and `StarterGui` in `Services`.
  2. Implemented a dynamic metatable on `Services` (`__index = function(self, key) ... return game:GetService(key) end`) to guarantee that any present or future Roblox service lookups never return `nil` if the service exists on the engine.
  3. Wrapped all service-level event connections (`PromptShown`, `JumpRequest`, `ChildAdded`) in `pcall` defensive checks to prevent script halts on restricted or non-standard executor environments.

### Lua 200 Local Variable Limit Audit
- **Audit Findings:** The entire codebase was parsed and audited via AST analysis:
  - Top-level chunk local variables: **58** (limit is 200).
  - Maximum local variables in any nested function/scope: **20** (line 4286).
  - **Verdict:** The script is nowhere near the 200 local variable threshold.

---

## 7. Auto Steal Hop-Drop-Regrab & Delivery Mechanics

### The Anti-Cheat Carry Constraint
The server enforces strict limits on continuous egg-carrying distance and velocity without ground interaction. If a character teleports directly into the base with an egg or travels across the boundary without dropping and regrabbing, the delivery fails ("Delivery was rewound, too fast for your speed").

### Complete 4-Phase Delivery Sequence:
1. **Phase 1: Periodic Mid-Route Checkpoints**
   - As the player travels from the farm zone down to the boundary (`CONFIG.BoundaryCheckpointX` = 600.2), intermediate ground checkpoints are performed at configured intervals (`CONFIG.CheckpointInterval`).
   - Ground checkpoints settle position at ground altitude, clear velocities, execute direct drop (`dropHeldEggDirect`), wait for `not isHoldingEgg()`, and regrab via `EggState.CarryFieldEgg(uid)` (with fallback to physical prompt grab).
2. **Phase 2: Mandatory Boundary Checkpoint at `BoundaryCheckpointX` (600.2)**
   - Before crossing from the open farm lane into the player base plot, a final boundary checkpoint is mandatory.
   - Character settles at `(boundaryX, groundY, laneZ)`, drops the egg, verifies drop, regrabs, and verifies holding.
3. **Phase 3: Safe Zone Corridor Micro-Hops**
   - The script **NEVER** executes an instantaneous 450-stud teleport across the base border.
   - Instead, the character steps through the base corridor using progressive micro-hops (35–45 studs per hop) with brief `HopGap` stabilization pauses until reaching `CONFIG.BaseDropPosition.X` (~150).
4. **Phase 4: Delivery Location Drop & Settle Sequence**
   - Character settles at the delivery hopper position.
   - Executes `dropHeldEggDirect()` (client `EggState.DropFieldEgg("PlayerRequest")` + network remotes + GUI drop button).
   - Monitors `not isHoldingEgg()` with up to 2 seconds confirmation timeout and retry drop.

### Multi-Source `isHoldingEgg()` Detection
To prevent premature delivery attempts or false-positive state synchronization:
1. `EggDetector:CheckHolding()`: Checks active visibility of `DropHeldEgg` and `RunBackEffects` ScreenGuis.
2. `EggState.CarryUid`: Queries client game engine `EggState` to see if an egg UID is registered as held.
3. Character Rig Inspection: Inspects character model for active `Tool` with egg UID or parts containing `EggWeld`.

---

## 8. Complete GUI Structure & Sidebar Navigation

The Liquid Glass UI interface in `qualcom` now mounts all categories across 6 dedicated tabs:

1. **Farm Tab:**
   - `Stealing & Egg Management` (Auto Steal, Fast Steal, Specific Egg Species multi-dropdown, Steal Missing Index Eggs, Auto Steal Minigame Egg, Auto Capture Event Egg, Take From Holder, Auto Catch Shooting Star, Target Areas, Min Rarity)
   - `Halloween & Dimension Events` (Halloween Auto Steal, Instant Halloween Steal, Auto Buy Halloween Egg, Monster Farm, Auto Pumpkin, Ghost Dog Quest, Auto Fishing + Luck dropdown, Phantom Express, Vacuum Ghouler Boss, Soul Trading)
   - `Enchanted & Bloom Events` (Butterfly Bloom, Travel Speed slider, Wisp Companion, Banjo Cricket, Trade Up, Craft Essence, Mutate Eggs)
   - `Pet & Egg Management` (Auto Place Egg, Placement Order dropdown, Auto Hatch, Equip Best, Sell Pets + Max Rarity, Keep Mutated Pets, Sell Eggs, Fuse Machine, Favorite)
   - `Training & Movement` (Auto Treadmill, Stay On Treadmill, Admin Treadmill, Speed Boost, WalkSpeed slider 16-300, Infinite Jump)
   - `Automation Upgrades` (Treadmill Upgrade, Base Expansion, Claim Rewards)
2. **Visuals Tab:**
   - `Egg ESP & Filters` (ESP Eggs, Fixed Size, Own Base, Other Base, Min ESP Rarity)
   - `Entity & Player ESP` (Guard ESP, Player ESP)
3. **Utility Tab:**
   - `Character Protections` (Anti-Trap Immunity, Anti-Ragdoll Recovery, Stealth Anti-AFK, Instant Prompts, Auto Rejoin On Disconnect)
   - `Performance & Optimization` (Optimizer, Disable 3D Render, FPS Cap slider 30-240, Hide Game UI, Farm HUD)
4. **Priority Tab:**
   - Reorderable 12-task priority hierarchy with Move Up, Move Down, and Enable/Disable toggles.
5. **Steal Panel Tab:**
   - Real-time world egg scanner with individual targeted steal triggers.
6. **Settings Tab:**
   - Theme customization and UI transparency slider.

## 9. Halloween Dimension Portal Stability & Dimension Supervisor

### Root Cause of Immediate Portal Ejection Bug
Previously, the Dimension Supervisor inside the 250ms `StealAndTreadmill` worker aggressively checked if `topTask` was Normal dimension or `nil`. Whenever a player entered the Halloween Dimension manually to explore, or when idle tasks like `Auto Treadmill` were active, `targetDim == "Normal" and inHalloween` triggered immediately, calling `HalloweenPortalManager.ExitDimension(5.0)` within 250ms and kicking the player out.

### Technical Resolution:
1. **Zero Ejection on Idle / Nil Tasks:** If `topTask == nil`, or if `topTask` is an idle task (`Auto Treadmill`, `Auto Use Admin Treadmill`), the supervisor **never** initiates an automated exit. Players can freely enter and explore Halloween Dimension without being booted.
2. **Active Work Requirement for Preemption:** Dimension transitions only occur when an active, runnable movement task (`PriorityManager:CanRun(task) == true`) with confirmed pending work (`PriorityManager:HasWork(task) == true`) demands character relocation.
3. **Graceful Preemption:** If an active Halloween task is running and a higher-priority Normal task (e.g., `Auto Steal` spotting a target egg) claims priority, combat and pumpkin tasks are aborted safely before dimension transfer.
