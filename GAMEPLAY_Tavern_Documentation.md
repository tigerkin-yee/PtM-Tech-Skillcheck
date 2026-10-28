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
├─ [DROW] This door is closed to surfacers, not to Us… → Lilith reply → Set `Global_Lilith_Gave_Key` (no roll; Osiris grants key)
├─ Deception: captain sent for supplies → Success: Hurry… / Fail: Liar. → End → combat
├─ Perception: Watch her at the door → Success: Narrator observation → drink/empathy → Lilith lets in + key
│                                      → Fail: noticed nothing → back to hub
└─ Fine, I'll leave → End
```

**Roll type (Toolkit):** **Skill** (not Raw_Ability) for **Deception** and **Perception** only. The drow line uses a **Player tag check (`DROW`)** and goes straight to Lilith’s response—**no Active/Passive Roll** on that branch.

**Liar. branch:** On the Deception **failure** Lilith line, **Set** `Global_Lilith_StartCombat` → **End Conversation** (do not return to hub).

---

## 4. Flags

Register all flags in **Story Editor → User Flags** as **Global**. Document them in dialog comments and goal file headers.

**Naming (matches current `Player, Lilith`)**

- **Dialog flags:** `Lilith-Met`, `Global_Lilith_Gave_Key`, `Global_Lilith_StartCombat`, etc. (Osiris `(FLAG)` names must match Story Editor exactly, including the `Global_` prefix and the hyphen in `Lilith-Met`).
- **Level / goals / items:** `PTM_SC_*` (e.g. `PTM_SC_TavernDoor`, `PTM_SC_TavernKey`, `PTM_SC_Tavern_Secret_Button_Revealed` below).

| Flag | Where set (dialog / level) | Purpose |
|------|----------------------------|---------|
| `Lilith-Met` | First time entering dialog (Greeting or hub entry) **Set** | README first-encounter flag; shorter repeat greeting |
| `Lilith_Read_Mood` | Perception **Success** (Roll Result or narrator Answer) **Set** | Unlocks drink / empathy player line (when wired in dialog) |
| `Global_Lilith_Gave_Key` | Drow identity let-in, empathy success, Deception success, etc. **Set** | Osiris **grants key** or entry permission |
| `Global_Lilith_StartCombat` | Deception fail **Liar.** node **Set** | After **End**, Osiris **EnterCombat** |
| `PTM_SC_Tavern_Secret_Button_Revealed` | World Perception success on hidden object (Osiris / interact) **Set** | Show **hidden button**; press unlocks door |

**Checks:** Repeat Greeting uses `if Lilith-Met` / `if Global_Lilith_Gave_Key`. Drow player line uses **check tag `DROW`** on the player (not Lilith).

---

## 5. Osiris / Story wiring

| File | Responsibility |
|------|----------------|
| `PTM_SC_TavernDoor.goal` | Level ready: **Lock** the tavern entry door, `DB_PTM_SC_TavernDoor`, seed key on **Lilith** for loot |
| `PTM_SC_LilithKey.goal` | After dialog **Sets** `Global_Lilith_Gave_Key` → **DialogEnded** → `TemplateAddTo` on **player** (dialog does not give the item) |
| `PTM_SC_LilithCombat.goal` | `Global_Lilith_StartCombat` → **DialogEnded** → `EnterCombat` |
| `PTM_SC_TavernSecretButton.goal` | World Perception / button → `PTM_SC_Tavern_Secret_Button_Revealed` → **Unlock** |

| User flag (Story Editor) | Osiris `(FLAG)` suffix UUID |
|--------------------------|-----------------------------|
| `Global_Lilith_Gave_Key` | `5a6dfef9-21b4-68e9-ef37-e656bc18f9bc` |
| `Global_Lilith_StartCombat` | `179d19ac-ceb5-68f6-dc16-ff1b9761eeb9` |
| `PTM_SC_Tavern_Secret_Button_Revealed` | `b2a8a327-c7a9-d103-f5b0-f483b228e1b4` |

**Key item root template:** `(ITEMROOT)2da08dbc-5c81-4cbc-beaa-6953bddd7e1e` (`PTM_SC_TavernKey`).

### 5.1 Deception failure → combat

**Chain:** Dialog sets `Global_Lilith_StartCombat` → **DialogEnded** → `PTM_SC_LilithCombat.goal` runs.

```osiris
IF
DialogEnded(_Dialog, _InstanceID)
AND
GetFlag((FLAG)Global_Lilith_StartCombat_179d19ac-ceb5-68f6-dc16-ff1b9761eeb9, NULL_00000000-0000-0000-0000-000000000000, 1)
AND
DialogGetInvolvedPlayer(_InstanceID, 1, _Player)
AND
DialogGetInvolvedNPC(_InstanceID, 1, _Lilith)
THEN
EnterCombat(_Lilith, _Player);
ClearFlag((FLAG)Global_Lilith_StartCombat_179d19ac-ceb5-68f6-dc16-ff1b9761eeb9, NULL_00000000-0000-0000-0000-000000000000, 0);
```

### 5.2 Let in → key

**Chain:** Dialog node **Set** `Global_Lilith_Gave_Key` only (drow identity, empathy, Deception success, etc.) → on **DialogEnded**, `PTM_SC_LilithKey.goal` grants the item:

```osiris
IF
DialogEnded(_Dialog, _InstanceID)
AND
DialogGetInvolvedPlayer(_InstanceID, 1, _Player)
AND
GetFlag((FLAG)Global_Lilith_Gave_Key_5a6dfef9-21b4-68e9-ef37-e656bc18f9bc, NULL_00000000-0000-0000-0000-000000000000, 1)
AND
TemplateIsInInventory((ITEMROOT)2da08dbc-5c81-4cbc-beaa-6953bddd7e1e, _Player, _KeyCount)
AND
_KeyCount == 0
THEN
TemplateAddTo((ITEMROOT)2da08dbc-5c81-4cbc-beaa-6953bddd7e1e, _Player, 1, 1);
```

One flag covers multiple let-in paths. The player uses the key on the locked door (no automatic `Unlock` in these goals).

### 5.3 Key on corpse (README)

Configure the **same key** Root Template on Lilith **loot** (Stats or level instance).  
`Global_Lilith_Gave_Key` and corpse loot should share the **same item template** (`PTM_SC_TavernKey`); the door **Key** field is `PTM_SC_Tavern`.

### 5.4 Locked door

**Scope:** Only the **outer tavern entry** is managed by `PTM_SC_TavernDoor` — level item `91e2f5fa-0be3-4f95-b23b-61c68a4965cb` (`Levels/GAMEPLAY_Tavern/Items/…`).

On **LevelGameplayReady** for `GAMEPLAY_Tavern`:

- `Lock(_Door, "PTM_SC_Tavern")` for each row in `DB_PTM_SC_TavernDoor`.
- Level instance **Key** field = `PTM_SC_Tavern` (matches key item stats name).

Wired: `PTM_SC_TavernSecretButton.goal` **Unlock**s the door after the button is used while `PTM_SC_Tavern_Secret_Button_Revealed` is set.

### 5.5 Hidden button (discreet way)

**Not** part of the Lilith dialog tree. **Shipped** on level `GAMEPLAY_Tavern`.

| Asset | ID / path |
|------|-----------|
| Root template | `PTM_SC_Tavern_SecretButton` · `25d4ffb7-a73d-4fd8-9199-d36862fc4d93` (parent `PUZ_GEN_Lever_Secret_Selune_A`) |
| Level instance | `477b9cb1-95c3-4fa5-9445-da040c6918e3` |
| Scripts on root | `PUZZLE_Lever` + **`PUZZLE_HiddenPerception`** (`68f5f134-…`, DC **10**, MinDistance **5**) |
| Hide | Explicit `Opacity=0`; HiddenPerception `SetVisible(0)` until reveal |
| Goal | `PTM_SC_TavernSecretButton` — `SetVarInteger(…,"DC",10)`, `SetDisableUse` until reveal, `StoryReveal` → Flag, `UseFinished` → `Unlock` door |

Flow:

1. Interactable starts **hidden / use-disabled**.  
2. Approach (~5 m) → engine **Perception** Skill check DC **10** via HiddenPerception.  
3. Success → `GLO_Skillchecks` fires `StoryReveal` → set `PTM_SC_Tavern_Secret_Button_Revealed` → enable use.  
4. Button press → goal **Unlock** the same tavern door.

---

## 6. End-to-end player flows

```
                    ┌─ [DROW] surfacers / Us (no roll) ──→ Global_Lilith_Gave_Key ──→ Osiris key / open door
                    │
Talk to Lilith ─────┼─ Deception success ───────→ let in (+ Set Global_Lilith_Gave_Key)
                    │
                    ├─ Deception failure ───────→ Global_Lilith_StartCombat ──→ End ──→ EnterCombat
                    │
                    ├─ Perception success ──────→ Lilith_Read_Mood ──→ empathy ──→ Global_Lilith_Gave_Key
                    │
                    └─ Leave

Skip dialog ─── world Perception ──→ PTM_SC_Tavern_Secret_Button_Revealed ──→ button ──→ open door

Kill Lilith ─── loot key ──→ open door
```

**First conversation:** Opening `Player, Lilith` **Sets** `Lilith-Met` (README first-encounter requirement).

---

## 7. Deliverable checklist

| Element | Path / notes |
|------|----------------|
| Lilith stats | `Data/Public/…/Stats/Generated/Data/Character.txt` |
| Dialog | `Data/Mods/…/Story/Dialogs/Player, Lilith.lsj` |
| Dialog binary | `Story/DialogsBinary/Player, Lilith.lsf` (from Build) |
| Story goals | `PTM_SC_TavernDoor` + `PTM_SC_LilithKey` / `PTM_SC_LilithCombat` + `PTM_SC_TavernSecretButton` |
| Flag defs | Story Editor → Generate Definitions |
| Level | `Levels/GAMEPLAY_Tavern` — Lilith `bc026460-…`, outer door `91e2f5fa-…`, secret button `477b9cb1-…` |
| Key item | Root `2da08dbc-…` · stats `PTM_SC_TavernKey` · door key id `PTM_SC_Tavern` |
| Lilith loot key | `PTM_SC_TavernDoor` seeds one key on Lilith if missing (`TemplateIsInUserInventory`) |

---

## 8. Build and test

1. Preferences: **disable** “Add impossible flag to dialogue”.  
2. **Build** dialog → Story **Generate Definitions, Build and Reload**.  
3. **File → Reload Level** before Game Mode.  
4. Test matrix: **drow PC sees identity line, no Persuasion roll** · non-drow does not · Deception pass/fail · in-dialogue Perception pass/fail · button Perception · post-combat loot · repeat dialog with `Lilith-Met` / key-already-given greeting.

---

*Matches dialogue design: outsider Greeting; drow “surfacers / Us” with **no Persuasion roll**; Deception “Liar.” → combat; in-dialogue Perception + drink empathy; world Perception for hidden button.*
