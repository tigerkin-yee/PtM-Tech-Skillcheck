# GAMEPLAY_Tavern — Skill Check Integration Notes

Short reference for how **Stats, dialogue, flags, Osiris, the locked door, and the discreet entry** fit together for the **Lilith** guard encounter. Level: **GAMEPLAY_Tavern**.

---

## 1. Encounter overview

The player must enter a locked tavern. **Lilith**, a drow guard, blocks the door and turns outsiders away. Entry options:

| Route | Type | Intended outcome |
|------|------|------------------|
| Drow identity | Dialogue · `DROW` tag only (**no skill roll**) | Lilith lets the player in; key or permission |
| Deception | Dialogue · Deception | Success: let in; failure: **Liar.** → combat |
| Perception (in dialogue) | Dialogue · Perception → empathy | Read her state, offer to fetch a drink, let in |
| Leave | Dialogue | End; no flags |
| Discreet | Level · world Perception | Reveal hidden button; unlock door |
| Combat | Failed Deception / kill | Key on corpse (loot) |

**In-dialogue Perception (read Lilith)** and **world Perception (find the button)** are separate: different flags and no shared lines.

---

## 2. NPC — Stats

| Field | Value |
|------|-----|
| **Stats entry** | `Lilith` |
| **File** | `Data/Public/PTM_SkillCheck_…/Stats/Generated/Data/Character.txt` |
| **Base** | `_Elf_Drow` |
| **Level** | 8 |
| **HP (Vitality)** | 80 |
| **AC (Armor)** | 13 |
| STR 15 · DEX 13 · CON 11 · INT 12 · WIS 13 · **CHA 15** |
| **Gear / proficiencies** | Light armor, shield, shortsword, bows, etc. (see `Proficiency Group` in Character.txt) |

**Design intent:** Level 8 drow guard with decent Charisma; **Deception** DC around **14–17**. The empathy branch (drink offer) after in-dialogue Perception may use a follow-up Persuasion check or auto-success per dialog setup. **Drow identity is not a Persuasion gate**—recognition among kin is enough for Lilith to stand aside.

**Level binding:** Dialogue speaker 0 maps to level character list `bc026460-9511-4d5c-9e4f-20cc6e96e2b7` (`Player, Lilith.lsj` → `speakerlist`).

---

## 3. Dialogue asset

| Field | Value |
|------|-----|
| **File** | `Data/Mods/PTM_SkillCheck_…/Story/Dialogs/Player, Lilith.lsj` |
| **Dialog UUID** | `9919b6fc-d435-7ebb-73d7-f4a86dfc300b` |
| **Category** | Generic NPC Dialog |
| **Speakers** | 0 = Lilith · 2 = GROUP_Players · narration = Narrator (Answer node) |

### 3.1 Branch layout (current design)

```
Greeting — Lilith: tavern closed, no outsiders
├─ [DROW] This door is closed to surfacers, not to Us… → Lilith reply → Set SC_Lilith_Gave_Key / key (no roll)
├─ Deception: captain sent for supplies → Success: Hurry… / Fail: Liar. → End → combat
├─ Perception: Watch her at the door → Success: Narrator observation → drink/empathy → Lilith lets in + key
│                                      → Fail: noticed nothing → back to hub
└─ Fine, I'll leave → End
```

**Roll type (Toolkit):** **Skill** (not Raw_Ability) for **Deception** and **Perception** only. The drow line uses a **Player tag check (`DROW`)** and goes straight to Lilith’s response—**no Active/Passive Roll** on that branch.

**Liar. branch:** On the Deception **failure** Lilith line, **Set** `SC_Lilith_StartCombat` → **End Conversation** (do not return to hub).

---

## 4. Flags

Register all flags in **Story Editor → User Flags** as **Global**. Document them in dialog comments and goal file headers.

| Flag | Where set (dialog / level) | Purpose |
|------|----------------------------|---------|
| `SC_Met_Lilith` | First time entering dialog (Greeting or hub entry) **Set** | README first-encounter flag; shorter repeat greeting |
| `SC_Lilith_Read_Mood` | Perception **Success** (Roll Result or narrator Answer) **Set** | Unlocks drink / empathy player line |
| `SC_Lilith_Gave_Key` | Drow identity let-in, empathy success, Deception success, etc. **Set** | Osiris **grants key** or entry permission |
| `SC_Lilith_StartCombat` | Deception fail **Liar.** node **Set** | After **End**, Osiris **EnterCombat** |
| `SC_Tavern_Secret_Button_Revealed` | World Perception success on hidden object (Osiris / interact) **Set** | Show **hidden button**; press unlocks door |

**Checks (optional):** Repeat Greeting uses `if SC_Met_Lilith` / `if SC_Lilith_Gave_Key`. Drow player line uses **check tag `DROW`** on the player (not Lilith).

**Do not mix:** `SC_Lilith_Read_Mood` = in-dialogue read only; `SC_Tavern_Secret_Button_Revealed` = world Perception only.

---

## 5. Osiris / Story wiring

Goals live under:  
`Data/Mods/PTM_SkillCheck_…/Story/RawFiles/Goals/`

### 5.1 Deception failure → combat

**Chain:** Dialog sets `SC_Lilith_StartCombat` → **DialogEnded** → goal runs.

```osiris
IF
DialogEnded((DIALOGINSTANCE)_Inst, _)
AND
GlobalGetFlag((FLAG)SC_Lilith_StartCombat, 1)
AND
DialogGetInvolvedPlayer(_Inst, _Player, 1)
AND
DialogGetInvolvedNPC(_Inst, _Lilith, 1)
THEN
EnterCombat(_Lilith, _Player);
ClearFlag((FLAG)SC_Lilith_StartCombat, _Player, 0, 0);
```

**Example goal name:** `PTM_SC_LilithCombat.goal`

### 5.2 Let in → key

**Chain:** Dialog node sets `SC_Lilith_Gave_Key` (drow identity Lilith reply, empathy success, Deception success, etc.) → on **DialogEnded** or **FlagReactionAfterDialog**:

- `TemplateAddTo` / spawn **Tavern Key** item on the player, or  
- `Unlock` the door directly (if not using a key item)

One flag covers multiple let-in paths. Avoid duplicate keys: only grant if `SC_Lilith_Gave_Key` was not already set before this dialog instance ends.

### 5.3 Key on corpse (README)

Configure the **same key** Root Template on Lilith **loot** (Stats or level instance).  
`SC_Lilith_Gave_Key` and corpse loot should share the **same item template**; the door’s **Key** field points to that template.

### 5.4 Locked door

- Door entity: **Locked**, key = template above; or  
- Osiris: when `SC_Lilith_Gave_Key` or button flag is set, `Unlock` / `Open` the door instance.

### 5.5 Hidden button (discreet way)

**Not** part of the Lilith dialog tree:

1. Hidden interactable starts **non-interactable / hidden**.  
2. Player **Perception** on a placed object or zone (suggested DC **10**, Skill roll not raw Wisdom).  
3. Success → set `SC_Tavern_Secret_Button_Revealed` → goal shows button or enables interact.  
4. Button press → goal **Unlock** the same tavern door.

---

## 6. End-to-end player flows

```
                    ┌─ [DROW] surfacers / Us (no roll) ──→ SC_Lilith_Gave_Key ──→ key / open door
                    │
Talk to Lilith ─────┼─ Deception success ───────→ let in (+ Set SC_Lilith_Gave_Key as designed)
                    │
                    ├─ Deception failure ───────→ SC_Lilith_StartCombat ──→ End ──→ EnterCombat
                    │
                    ├─ Perception success ──────→ SC_Lilith_Read_Mood ──→ empathy ──→ SC_Lilith_Gave_Key
                    │
                    └─ Leave

Skip dialog ─── world Perception ──→ SC_Tavern_Secret_Button_Revealed ──→ button ──→ open door

Kill Lilith ─── loot key ──→ open door
```

**First conversation:** Opening `Player, Lilith` **Sets** `SC_Met_Lilith` (README first-encounter requirement).

---

## 7. Deliverable checklist

| Element | Path / notes |
|------|----------------|
| Lilith stats | `Data/Public/…/Stats/Generated/Data/Character.txt` |
| Dialog | `Data/Mods/…/Story/Dialogs/Player, Lilith.lsj` |
| Dialog binary | `Story/DialogsBinary/Player, Lilith.lsf` (from Build) |
| Story goals | `Story/RawFiles/Goals/*.goal` (combat, key, button) |
| Flag defs | Story Editor → Generate Definitions |
| Level | `Levels/GAMEPLAY_Tavern` — Lilith, door, hidden button |
| Key item | Public/Stats or derived template + door Key field |

---

## 8. Build and test

1. Preferences: **disable** “Add impossible flag to dialogue”.  
2. **Build** dialog → Story **Generate Definitions, Build and Reload**.  
3. **File → Reload Level** before Game Mode (see README Patch 8 notes).  
4. Test matrix: **drow PC sees identity line, no Persuasion roll** · non-drow does not · Deception pass/fail · in-dialogue Perception pass/fail · button Perception · post-combat loot · repeat dialog with `SC_Met_Lilith`.

---

## 9. Git / naming

Branch example: `feature/GAMEPLAY_Tavern_<YourName>_OP` (see CONTRIBUTING.md).  
For full PTM, dialog files may later use `PTM_GP_Tavern_Lilith`; skill check can keep `Player, Lilith`.

**Local docs:** English deliverable = this file (`GAMEPLAY_Tavern_Documentation.md`). Chinese mirror = `GAMEPLAY_Tavern_Documentation_CN.md` (local only, not committed). Keep both in sync when editing.

---

*Matches dialogue design: outsider Greeting; drow “surfacers / Us” with **no Persuasion roll**; Deception “Liar.” → combat; in-dialogue Perception + drink empathy; world Perception for hidden button.*
