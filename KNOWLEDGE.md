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

### Platform Breakdown: Delta Mobile vs PC Executors
- **Delta Mobile (Android):** Employs full C-level closures for `fireproximityprompt(prompt, 0)` and `firetouchinterest(part1, part2, 0/1)`. Exploit hooks operate in a unified 32/64-bit ARM environment.
- **PC Executors (Wave, Synapse Z, Solara, Celery, Macsploit, etc.):**
  - **Proximity Prompts:** Often lack `fireproximityprompt` completely (`nil`), or require 2 arguments (`prompt, 0`), or throw exceptions if called without `pcall`. Direct engine calls (`prompt:InputHoldBegin()`) fail unless `RequiresLineOfSight` is disabled, distance checks pass, and hold durations are allowed at least 1-2 frames.
  - **Touch Interests:** Many budget PC executors have stubbed `firetouchinterest` implementations that do not replicate `Touched` events across distances. If a character floats above a monster (e.g. 14 studs), `firetouchinterest` does not hit unless the weapon handle physically intersects the target's bounding box.
  - **Button Clicks:** `getconnections` is frequently stubbed or lacks `:Fire()`. Official Roblox `VirtualInputManager` is required as a rock-solid engine fallback.

### Universal Compatibility & Polyfill Layer (`UniversalCompat`)
All script automation modules now route through `UniversalCompat` rather than making raw executor calls:

1. **`UniversalCompat.FireProximityPrompt(prompt)`**:
   - Zeroes `prompt.HoldDuration = 0`, disables `prompt.RequiresLineOfSight = false`, and increases `prompt.MaxActivationDistance = 50`.
   - Method A: Multi-signature pcall cascade: `fireproximityprompt(prompt, 0)`, `(prompt, 1)`, and `(prompt)`.
   - Method B: Safe engine hold: `prompt:InputHoldBegin()`, yields `task.wait(0.03)`, and `prompt:InputHoldEnd()`.
   - Method C: `VirtualInputManager:SendKeyEvent` simulation fallback using the prompt's `KeyboardKeyCode`.

2. **`UniversalCompat.FireTouch(part1, part2)`**:
   - Method A: Dual-stage `firetouchinterest(part1, part2, 0)` $\rightarrow$ `wait` $\rightarrow$ `(part1, part2, 1)`.
   - Method B: Forces `CanTouch = true` on both parts.
   - For Monster Combat on PC: Sweeps weapon handle momentarily into `monsterPart.CFrame` during attack strokes so the engine fires physical contact even without executor touch injection.
   - For Portals: Applies slight downward velocity (`Vector3.new(0, -0.5, 0)`) when snapping to portal CFrames so physics broadphase collision registers the step.

3. **`UniversalCompat.ClickButton(btn)`**:
   - Tri-level click dispatcher:
     1. `firesignal(btn.Activated)` / `firesignal(btn.MouseButton1Click)`
     2. `getconnections` loop invoking `c:Fire()` or `c.Function()`
     3. `VirtualInputManager:SendMouseButtonEvent` at button center (`AbsolutePosition + AbsoluteSize/2`).

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
| **Pet Haul** | `Remotes.WearBest` | RemoteFunction | Automatically equips player's highest multiplier pets (`RF/Haul/WearBest`) |
| **Pen Placements** | `Remotes.AskPlaceEgg` | RemoteFunction | Places eggs in pen slots (`RF/EggWorld/AskPlaceEgg`) |
| **Index / Codex** | `Remotes.AskRedeemAll` | RemoteFunction | Redeems all index rewards (`RF/Codex/AskRedeemAll`) |
| **Index / Codex** | `Remotes.AskRedeemLimitedEgg` | RemoteFunction | Redeems limited event index rewards (`RF/Codex/AskRedeemLimitedEgg`) |
| **Offline Cash** | `Remotes.AskCollectOffline` | RemoteFunction | Collects pending offline money (`RF/AwayEarnings/AskCollect`) |
| **Boss Mastery** | `Remotes.AskUseMutationConsumable` | RemoteFunction | Consumes mutation boosters (`RF/BossMastery/AskUseMutationConsumable`) |

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
  - Uses discrete altitude hops (`groundY = CONFIG.GroundAltitude + 1.0`).
  - Hop step size dynamically scales with player WalkSpeed:
    $$\text{stepSize} = (\text{playerSpeed} < 100) \mathbin{?} \max(34, \text{playerSpeed} \times 0.45) : \min(\text{baseStep}, \max(60, \text{playerSpeed} \times 0.75))$$
  - **Critical Fix:** Dropped the destructive mid-route checkpoint dropping (`performGroundCheckpoint`). Previously, dropping the egg mid-route caused low-speed players and PC executors to fail the prompt re-grab and lose eggs on the conveyor floor. Continuous dynamic hopping now maintains egg possession throughout the entire transit and drops the egg **strictly** upon reaching the home base plot drop target (`CONFIG.BaseDropPosition`).
  - Server rollback detection: if `pulled` occurs, immediately slows step size and increases `HopGap` to seamlessly recover.

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

### 9. Character Protection & Immunity System (Anti-Trap, Anti-Ragdoll, Anti-AFK)
- **Anti-Trap:** Disables `CanTouch` on all `PlacedTrap` instances not owned by the local player via `CollectionService:GetTagged("PlacedTrap")`. Immune to bear traps while running/stealing.
- **Anti-Ragdoll:** Monitors `Humanoid.StateChanged` for `Physics`, `Ragdoll`, or `FallingDown`. Strips ragdoll constraints (`BallSocketConstraint`, `NoCollisionConstraint`, `HingeConstraint`), ensures `Motor6D` and `AnimationConstraint` instances remain enabled, and forces `Humanoid:ChangeState(GettingUp)`.
- **Anti-AFK:** Disables `LocalPlayer.Idled` connections and attaches a `VirtualUser:CaptureController()` click fallback to prevent the 20-minute Roblox idle kick.

### 10. Reward Claims & Auto Equip Best
- **Auto Claim Rewards:** Background worker automatically checks and redeems:
  - `RF/AwayEarnings/AskCollect` with `{ Kind = "Claim" }` for offline earnings.
  - `RF/Codex/AskRedeemAll` for codex/index progression rewards.
  - `RF/Codex/AskRedeemLimitedEgg` for event rewards.
- **Auto Equip Best:** Periodically invokes `RF/Haul/WearBest` to ensure the highest-income and speed-boost pets from inventory are equipped.

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
| `bc4ca8615c00b895b78cbe33f89ea12dd7b9d10a` | `fix(executor-compat): universal polyfills for PC...` | UniversalCompat layer for PC executors (prompts, touch, buttons) & safe low-speed delivery |
| `f089157247197e015bbb5bb6c2ea573ce0183d81` | `feat(automation): integrate Anti-Trap, Anti-Ragdoll...` | Added CharacterProtection (Anti-Trap, Anti-Ragdoll, Anti-AFK), Auto Equip Best & Reward Claims from ChiliHub |
