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

## 3. The 12-Task Priority Arbitrator & Preemption Hierarchy

The centralized `PriorityManager` oversees task execution, coordinating character movement, dimension routing, and mutual exclusion. Lower-priority tasks yield immediately to higher-priority tasks and resume when work completes.

### Canonical Priority Hierarchy (Configurable in UI)
1. **Auto Steal:** Stealing eggs from normal world base plots or minigames (RLGL / T-Rex).
2. **Shooting Star:** Navigating to and collecting falling stars (`RE/ShootingStar/Catch`).
3. **Event Egg Capture:** Grabbing and holding roaming world event eggs at a safe altitude.
4. **Candy Cane Farm:** Monster combat and candy collection in the Halloween Dimension.
5. **Auto Pumpkin:** Catching runaway pumpkins and depositing them at the dimension altar.
6. **Ghost Dog & Fishing:** Automating Ghost Dog quest dialogue/dive/claim and auto-fishing.
7. **Phantom Express:** Boarding the Ghost Train event and aiming suction at the Ghouler boss.
8. **Enchanted / Bloom Event:** Butterfly catching, wisp companion deliveries, and mutation crafting.
9. **Auto Place Egg:** Placing backpack eggs into empty base plot pen slots with sorting/filters.
10. **Auto Hatch & Fuse:** Auto-hatching inventory eggs, loading triplets into the fusion machine, and auto-selling.
11. **Auto Upgrades:** Homestead base expansions, treadmill tier upgrades, and codex rewards.
12. **Auto Treadmill:** Training walkspeed on plot or admin/VIP treadmills.

### Dynamic Preemption Mechanism
- Every frame/heartbeat, `PriorityManager:GetTopWantedTask()` evaluates the enabled state (`IsTaskEnabled`) and availability of work (`HasWork`) from rank 1 down to 12.
- If Auto Steal is active, it claims character movement.
- When Auto Steal finishes or enters idle cooldown, lower tasks like Candy Cane Farm or Auto Treadmill immediately take control.
- If a higher-priority task (e.g., Shooting Star or Auto Steal) detects work, lower tasks yield (`Abort()` / `Release()`) and hand off character movement cleanly.

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

### Egg Dropping Architecture (`dropHeldEggDirect`)
- **Direct Client Invocation:** The game client relies on `EggState.DropFieldEgg("PlayerRequest")` to update local character carry state, clear attachments, and notify the server.
- **Remote Function Cascade:** If `EggState` is absent or pending require, falls back to multi-signature invokes on `Remotes.AskFieldEggDrop` (`"PlayerRequest"`, `{ Reason = "PlayerRequest" }`, and `()`).
- **GUI Trigger Fallback:** If character remains flagged as holding an egg, searches `PlayerGui` for the `DropHeldEgg` / `Drop` button and triggers it via `UniversalCompat.ClickButton`.

### Intermediate Checkpoint Delivery (`performGroundCheckpoint`)
- **Hop-Drop-Regrab Sequence:** At each checkpoint interval along the lane ($currentX > 1050$), the character settles onto the ground, drops the held egg to reset physics anti-teleport checks, and regrabs it.
- **Stealth Carry Restoration:** Uses `EggState.CarryFieldEgg(uid)` directly (matching ChiliHub line 17589) followed by `instantGrabAtPosition` as physical fallback, preventing low-speed delivery drops from failing mid-transit.

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
