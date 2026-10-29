# Engineering Feedback & Behavioral Operating Guide (SAE Project)

> **Purpose:** This file contains all extracted user corrections, preferences, architectural constraints, anticheat guidelines, and lessons learned across development sessions on `tenobarillas-ship-it/sae`. **This file must be automatically loaded and reviewed at the beginning of every session.**

---

## 1. Core User Communication & Interaction Preferences

### A. Communication Style
- **Simple, Direct & Concise:** Deliver answers directly without unnecessary fluff or lengthy unsolicited explanations.
- **Strict Format Adherence:** If the user asks for a simple list with "no description", output strictly the list items—zero descriptions, zero filler.
- **Do Not Argue or Make Assumptions:** If the user states a requirement or reports that an approach does not work, believe them immediately. Never assume or fill gaps with guesswork.

### B. "Clean Rebuild" vs. "Incremental Patching" Rule
- **Never Stack Patches on Broken Logic:** When an implementation fails or behaves erratically across multiple iterations, **STOP patching**. 
- **The 4-Step Clean Rebuild Protocol:**
  1. **Complete Removal:** Remove all feature code, dedicated functions, state variables, connections, background loops, task priority registrations, and UI controls.
  2. **Clean Baseline Commit:** Commit and push the clean state to GitHub so there is an untainted baseline.
  3. **Study Reference Code:** Inspect the user's/friend's reference code thoroughly before writing a single line.
  4. **Direct Modular Integration:** Rebuild the feature cleanly inside an isolated block (`do ... end`), keeping the proven logic intact and wrapping it only for UI toggle on/off and dimension compatibility.

### C. Reference Code Fidelity (Don't Re-Engineer)
- **Do Not Reinvent the Wheel:** When the user provides a reference file (e.g. from their friend's repository or `.txt` attachment), **they want that exact working logic used**.
- **Minimal Wrapping Only:** Do not change its algorithms, movement math, or workflows. Only wrap it cleanly to connect it to the GUI toggle, configuration table, and dimension checks.

### D. Version Control & Reversions
- **Immediate Reverts When Requested:** When the user provides a specific commit SHA (e.g. `033b6cb`, `3a1611c`, `d45bdd24`), revert immediately and cleanly to that exact commit.
- **GitHub Raw API Verification:** Always verify pushed commits against the GitHub raw endpoint (`application/vnd.github.v3.raw`) to ensure byte-for-byte fidelity between local workspace and remote repository.
- **Maintain `KNOWLEDGE.md`:** Document all architectural changes, feature additions/removals, and commit SHAs in `KNOWLEDGE.md`.

---

## 2. Luau Engine & Anticheat Restrictions

### A. The 200 Local Variable Limit (Critical)
- **Luau Engine Limitation:** Luau strictly enforces a maximum of 200 local variables per function / scope. Exceeding this causes immediate script compilation failure or silent crashes on load.
- **Enforcement:**
  - Wrap large subsystems and self-contained features in `do ... end` blocks or dedicated sub-tables.
  - Never dump dozens of loose `local` variables into the main file scope.
  - Re-use shared services and scoped registries.

### B. Anticheat & Movement Constraints
- **NO Elevation / Floating:**
  - Flying or elevating along the Y axis flags server anticheat immediately, leading to instant death or disconnection.
  - All normal dimension movement and delivery must remain strictly at **ground level**.
- **Preserve Character Walking Animations:**
  - Never suppress or freeze the player's walking animation during travel or delivery. Suppressed animations trigger automated anticheat heuristics.
- **Speed Stat Integrity:**
  - Do NOT modify or override player travel speed arbitrarily during standard stealing routines. Use the player's legitimate speed (or calculate based on real speed stat).
- **Safe Zone Routing (T-Shaped Map):**
  - In the normal dimension, the map is shaped like a **T** (top bar = safe zone bases; vertical stem = egg spawns).
  - Never path straight through walls or collision barriers from the outer edges of the safe zone to eggs. Route properly from base into the center path.
- **Delivery Sequence (Hop - Drop - Regrab):**
  - The delivery sequence requires precise hopping, dropping, and regrabbing so the server validates possession before deposit. Do not alter or omit this sequence.
- **Input Method Anticheat:**
  - Avoid `VirtualInputManager` or simulated low-level keystrokes. They trigger client anticheat bans, especially on mobile executors. Use remote events and native interactions instead.

### C. Normal Dimension Auto Steal is SACRED
- **Never Break Normal Auto Steal:** The normal dimension `Auto Steal` implementation in `qualcom` is proven and trusted.
- **Isolation:** Under no circumstances should normal dimension steal logic be entangled with seasonal/Halloween logic. It must remain 100% untouched and byte-identical when updating event features.

---

## 3. Dimension Management & Portal Navigation

### A. Category Dimension Association
- **Halloween Event 2026 = Halloween Dimension:**
  - Any feature under the Halloween Event category belongs to the Halloween Dimension (with `Auto Buy Halloween Egg` as an exception).

### B. Action-Gated Portal Entry (Strict Rule)
- **Never Enter Portal Just Because Toggle is ON:**
  - Turning on a toggle (e.g. `Auto Steal Halloween Eggs`, `Auto Boss`, etc.) must **NOT** immediately force portal entry.
  - The script must first verify that an **action actually needs to be performed**:
    - For Halloween Egg Steal: Only enter if the player is currently carrying an egg OR an eligible target egg matching the area/rarity filters is detected.
    - For Auto Boss: Only enter if the Pumpkin King boss is active or pending combat.
  - If no action is needed, the player must stay in their current dimension.

### C. Dimension Exit Protection
- **No Unintended Ejection:**
  - Toggling off one feature or completing a single sub-task must **never** force-exit the player from the Halloween Dimension if other tasks or features are active.
  - Do not call `ExitDimension` blindly in cleanup hooks.

### D. Map Closed Logic (`isHalloweenMapClosed`)
- **No Main Map Bleedthrough:**
  - `isHalloweenMapClosed()` must check ONLY the Halloween dimension's barrier state.
  - **NEVER** include `isMapClosed()` (the normal world day/night cycle) inside `isHalloweenMapClosed()`. Normal day/night must never pause or interrupt actions in the Halloween dimension.

---

## 4. Feature-Specific Requirements & Workflows

### A. Auto Steal Halloween Eggs
- **Two Distinct Area Workflows:**
  1. **Graveyard ($\le 550$ studs):** Local direct steal and delivery.
  2. **Haunted Swamps & Abandoned Manor ($> 550$ studs):** Long-distance navigation using portal/ground navigation with collision bypass, grabbing, and delivery.
- **Filters:**
  - **Target Areas (MultiDropdown):** Filter by spatial bounding boxes (`Graveyard`, `Haunted Swamps`, `Abandoned Manor`, or `All`).
  - **Min Rarity (Dropdown):** Dynamic rarity resolution; only target eggs meeting or exceeding the chosen numeric rank.
- **Clean Toggle Abort:**
  - When disabled, abort ongoing tweens, reset body movers/velocity, and restore collision immediately.

### B. Auto Boss (Pumpkin King)
- Check dimension before traveling.
- Wait for boss stage, handle cannon firing and phase transitions cleanly.

### C. Auto Pumpkin
- Detect runaway pumpkins, grab, and deposit at Altar.

### D. Auto Fishing
- Cast timing precision selector (`"x1.5 Perfect"`, `"x1.2 Good"`, `"x1.0 Normal"`).
- Mobile-safe reel timing.

### E. Permanently Removed Features (Do Not Re-add Unless Explicitly Requested)
- `Auto Phantom Express` (Train Event)
- `Auto Vacuum Ghouler`
- `Auto Soul Trade In`

---

## 5. What to Do Differently Next Time (Agent Retrospective)

| Past Mistake / Trap | Root Cause | Future Action Rule |
| :--- | :--- | :--- |
| **Patching broken code repeatedly** | Reluctance to discard faulty logic | When a feature fails twice after bug reports, initiate a clean removal to baseline, commit, and rebuild fresh from reference. |
| **Re-engineering user's reference code** | Agent tried to "improve" or standardize code into existing patterns | Keep reference code algorithms intact; only wrap with minimal glue for GUI toggle and dimension gating. |
| **Entering portal when idle** | Checking toggle state instead of pending task action | Always gate dimension travel behind `hasAction()` check (e.g. `isCarryingEgg() or targetEgg ~= nil`). |
| **Bleedthrough of normal map day/night** | Calling global `isMapClosed()` inside Halloween functions | Strictly isolate dimension checks; Halloween dimension is unaffected by normal world day/night. |
| **Hitting 200 local variable limit** | Declaring variables loosely in file scope | Always wrap new modules in self-contained `do ... end` blocks or modular tables. |
| **Modifying normal steal while fixing Halloween steal** | Shared naming / global functions | Keep normal dimension Auto Steal completely isolated; never modify its functions or delivery pipeline. |
| **Overwriting player speed / elevating** | Assuming faster travel was better | Respect Roblox anticheat: ground-level only, preserve animations, use player's natural speed. |

---

## 6. How to Reload This Feedback in Future Sessions
At the start of every session:
1. Fetch and read `feedback.md` from the repo root.
2. Confirm active rules:
   - 200 local limit check.
   - Action-gated portal entry.
   - Ground-level movement & walking animation preserved.
   - Normal dimension Auto Steal untouched.
   - Direct reference code fidelity.
