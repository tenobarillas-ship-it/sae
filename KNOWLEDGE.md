# SYNTAXIS HUB (`qualcom`) - PROJECT KNOWLEDGE BASE

> **Notice:** This document is the living knowledge repository for the Syntaxis Hub project. It contains all architectural decisions, reverse-engineered game mechanics, remote endpoints, algorithm designs, executor compatibility rules, priority systems, and known issues. **Every time new knowledge is discovered, implemented, or refactored, this document must be updated.**

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
  - Modular distribution: `src/` (`Window`, `Components`, `Panels`, `Themes`, `Animations`, `Utils`, `Modals`).
  - Bundled single-request loader: `SyntaxisLib.lua`.
    ```lua
    local SyntaxisLib = loadstring(game:HttpGet("https://raw.githubusercontent.com/tenobarillas-ship-it/GUILib/main/SyntaxisLib.lua"))()
    ```

### Local Workspace & Sync
- **Local Workspace Working File:** `qualcom_clean.lua`
- **Settings File (on executor):** `SyntaxHubSettings.json` (persisted via `writefile` / `readfile`).

---

## 2. Platform & Executor Compatibility Rules

### Safe Container Parenting (`GetSafeParent`)
Roblox executors have varying levels of GUI protection and context permissions:
1. **Delta Mobile (Android):** Uses `gethui()`.
2. **PC Executors (Synapse Z, Wave, Macsploit, etc.):** `gethui()` might not return a valid container or may throw an error.
3. **Fallback Order:**
   ```lua
   local safeContainer = nil
   if typeof(gethui) == "function" then
       local ok, res = pcall(gethui)
       if ok and typeof(res) == "Instance" then safeContainer = res end
   end
   if not safeContainer then
       pcall(function()
           local cg = game:GetService("CoreGui")
           if cg and typeof(cg) == "Instance" then safeContainer = cg end
       end)
   end
   if not safeContainer then
       safeContainer = LocalPlayer:WaitForChild("PlayerGui")
   end
   ```

### Screen-Wide Mouse & Touch Dragging (`MakeDraggable`)
- **Bug identified:** Using `handle.InputChanged` drops the drag on PC if the user moves the mouse faster than the GUI element's frame update.
- **Rule:** Always attach `UserInputService.InputChanged` and `UserInputService.InputEnded` dynamically when drag starts, and disconnect on drag end.

---

## 3. Remote Endpoints & Game Systems

| Category | Remote / Method | Type | Function / Description |
| :--- | :--- | :--- | :--- |
| **Egg State** | `EggState.ReadFieldEggs()` | Function | Returns live table of eggs on conveyor belts with Uid, State, AssetCategory, AreaId, AssetScale, Mutations, BoundsCFrame |
| **Treadmill** | `Remotes.AskWearStill` | RemoteFunction | Places character onto base treadmill for speed training |
| **Treadmill** | `Remotes.AskDoff` | RemoteFunction | Takes character off base treadmill |
| **Upgrades** | `Remotes.TierRaise` | RemoteEvent | Upgrades treadmill tier |
| **Upgrades** | `Remotes.HomesteadRaise` | RemoteEvent | Upgrades homestead base expansion |
| **Butterflies** | `Remotes.AskClaimNet` | RemoteFunction | Claims butterfly net (`RF/Butterflies/AskClaimNet`) |
| **Butterflies** | `Remotes.EquipNet` | RemoteFunction | Equips butterfly net (`RF/Butterflies/Equip`) |
| **Wisps** | `Remotes.WispRequest` | RemoteFunction | Submits eggs / completes wisp companion quests (`RF/WispCompanion/Request`) |
| **Trade Up** | `Remotes.ButterflyTrade` | RemoteFunction | Exchanges 5 lower-tier butterflies into 1 higher-tier butterfly |
| **Essence** | `Remotes.CraftEssence` | RemoteFunction | Crafts Enchanted Essence at the Mutation Shrine |
| **Character** | `Remotes.AskRigWipe` | RemoteEvent | Cleans up flagged / corrupted character models |

### Valuation & Earning Formulas
- **Egg Score Calculation:**
  $$\text{Score} = (\text{Rank} \times 1000) + (\text{Scale}^{1.85} \times \text{MutationBonus})$$
  - `MutationBonus`: $2.5$ if mutated, $1.0$ otherwise.
  - Scale factor: If scale $> 5.0$, factor is $\left(\frac{\text{scale}}{5}\right)^{1.2} \times 19.637875755794113$, else $\text{scale}^{1.85}$.
- **Rarity Ranks:**
  1. Common
  2. Uncommon
  3. Rare
  4. Epic
  5. Legendary
  6. Mythic
  7. Cosmic
  8. Secret
  9. Eternal
  10. Divine

---

## 4. Automation Modules & Behaviors

### 1. Auto Steal & Fast Steal (Normal Dimension)
- **Target Areas:** Forest, Lake, Desert, Jungle, Snow, Volcano, Abyss Ocean, Prehistoric, Cosmic, Cherry Blossom, Titan Temple, Light Dark, Enchanted Forest.
- **Fast Steal:** Applicable to Titan Temple, Light Dark, and Enchanted Forest. Uses direct approach and instant proximity prompt triggers.
- **Low-Speed Dynamic Hopping (`executeCheckpointedHopsHome`):**
  - Uses discrete altitude hops (`FlightAltitude = 71.5`, `HopAltitude = 71.8`).
  - Hop step size dynamically scales with player WalkSpeed:
    $$\text{stepSize} = \text{math.clamp}(\text{speed} \times \text{dt} \times \text{factor}, \text{minStep}, \text{maxStep})$$
  - Prevents server-side rewind rollbacks on lower-speed players.
  - Final drop position: `Vector3.new(528.7, 70.8, -364.1)`.

### 2. Steal Panel
- Live scanner showing: Name, Rarity tier, Value ($/s), Scale multiplier, and Area.
- Sorting modes: `Value (Money/s)`, `Rarity`, `Weight (Scale)`.
- Selecting a specific egg forces absolute priority over general Auto Steal.
- *Halloween eggs are excluded from the Steal Panel.*

### 3. Candy Cane Monster Farm (Halloween Dimension)
- Automatically equips tools/weapons from inventory (`GetWeaponList()`).
- Alternates weapons, hovers above monster (`HoverHeight = 12-14`), and invokes `firetouchinterest` on tool Handle and monster root.
- Cleanly yields and aborts via `HalloweenCombatManager:Abort()` when preempted.

### 4. Auto Pumpkin (Halloween Dimension)
- Detects runaway pumpkins in `HalloweenDimension.RunawayPumpkins`.
- Glides to target pumpkin, grabs it, and carries it to the Altar.
- Marked **non-interruptible** while actively carrying (`IsCarrying() = true`) to prevent pumpkin dropping.

### 5. Auto Steal Halloween Eggs (Temporarily Hidden)
- *Status:* Hidden from the GUI and disabled by default.
- *Reason:* Resolved issues with gate returns / dimension desync are pending a clean physics bypass. Underlying code is preserved in `HalloweenStealManager`.

### 6. Enchanted / Bloom Event
- Automatically detects `workspace.ButterflyBloomSwarm` and individual butterflies.
- Claims and equips net via remotes.
- Travels to butterflies using tweening at user-controlled `ButterflySpeed` (100–600).
- Automatically completes Wisp quests, trade-ups, and essence crafting.

### 7. Auto Upgrades
- Passive background tasks: `tryUpgradeTreadmill` (TierRaise) and `tryUpgradeBase` (HomesteadRaise).
- Runs every 6.0 seconds without moving the character or blocking other automations.
- Works anywhere on the map (removed `isAtBasePlot()` requirement).

### 8. Auto Treadmill
- Idle speed training fallback.
- Walks to plot treadmill, verifies anchoring, and trains.
- Features auto-recovery: resets character if stuck (`State.treadmillFailCount >= 8`).
- Immediately yields and unanchors (`hopOffTreadmill()`) as soon as ANY higher-priority task requires the character.

---

## 5. Unified Priority System & Dimension Supervisor

### How the Priority Hierarchy Works
1. **Canonical Tasks in Priority Order:**
   1. `Auto Steal`
   2. `Candy Cane Farm`
   3. `Auto Pumpkin`
   4. `Enchanted / Bloom Event`
   5. `Auto Upgrades`
   6. `Auto Treadmill`
2. **`PriorityManager:GetTopWantedTask()`**:
   - Evaluates tasks from **Rank 1** to **Rank 6**.
   - Checks `IsTaskEnabled()` and `HasWork()`.
   - The highest-ranked task with active work becomes the **Sole Authority Task**.
3. **`PriorityManager:CanRun(taskName)`**:
   - Strictly enforces that only the top wanted task (or non-conflicting background upgrades) can run.
4. **Dimension Supervisor**:
   - Automatically switches dimensions between **Normal Dimension** and **Halloween Dimension** based on `GetTaskDimension(topTask)`.
   - If player is in Halloween dimension and a Normal Dimension task (e.g. Auto Steal) gets work:
     - Halts combat immediately (`HalloweenCombatManager:Abort()`).
     - Teleports through the exit portal (`HalloweenPortalManager.ExitDimension(5.0)`).
     - Steals the egg in the Normal Dimension.
     - When finished and no more eggs remain, returns to Halloween dimension and resumes combat!
5. **Non-Interruptible Carrying Protection**:
   - `isHoldingEgg()` and `HalloweenPumpkinManager:IsCarrying()` guarantee that deliveries cannot be interrupted mid-flight.

---

## 6. Commit History & Revert Reference

| Commit SHA | Commit Title / Name | Description |
| :--- | :--- | :--- |
| `2f51383cda7eaf2f5ac12c6dc7cbab0245a4c917` | `Initial Clean Baseline` | Baseline before Halloween dimension overhaul |
| `5a3e82b8260bc2602685225a6e79183c455bfb5e` | `fix(halloween): resolve dimension routing conflicts...` | Initial dimension routing fixes |
| `8bd71b0eaaad4a598be3528dc100fd2277caaa05` | `fix: auto upgrades, bloom event, low-speed delivery...` | Removed plot check from upgrades, fixed Net claim |
| `bca0e25f55161b1200145862f52d09c883589bf2` | `refactor(gui): integrate modular GUILib...` | Decoupled ~2,500 lines of GUI boilerplate into GUILib |
| `e762c321a1038d6e7e3beb8b6b3185e009277f1e` | `fix(priority): resolve task hierarchy preemption...` | Complete priority preemption & bidirectional supervisor |
