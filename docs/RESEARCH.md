# RESEARCH — Can DFHack give Dwarf Fortress NPCs a voice?

> Full evidence chain, compiled 2026-09-28. Every claim cites a source file, a line, or an
> official document. Claims that could not be verified are marked as such rather than smoothed
> over.
>
> **Method note:** this machine is behind a restrictive network. GitHub was reached through
> `gh api` (authenticated) and the wiki/docs through a local fetch script. Where a model's
> summary disagreed with primary source, the primary source won — and one summary *was* wrong
> (see § 3.9, where an earlier draft of this research overstated what could be written).

---

## 0. Verdict

| Capability | Verdict | Evidence |
|---|---|---|
| Read dialogue state | ✅ strong | `interactions.lua::M.conversation()` |
| Read dialogue **semantics** | ✅ strong | `utterancest.rumor` (full `entity_event`) + `incident_id`; `talk_choice` union |
| Read NPC personality | ✅ very strong | `character_details.lua`, 22 sections |
| External process ↔ DFHack, bidirectional | ✅ | RPC TCP 5000; `RunCommand` → `lua --file` |
| Detect "player entered a conversation" | ✅ event-driven | `SC_VIEWSCREEN_CHANGED` + `matchFocusString` |
| Inject player-side options / keywords | ✅ official precedent | `convo.lua::new_choice()` |
| Draw custom UI in the dialogue screen | ✅ official precedent | `AdvRumorsOverlay` + `gui/dialogs.lua` + `adv_announcementst.str` |
| Send an NPC somewhere; make them follow | ✅ official precedent | `move_unit()` / `gui/companion-order` |
| Persist memory across saves | ✅ official API | `dfhack.persistent.saveWorldData` |
| **Replace an NPC's spoken line** | ❌ **impossible** | `utterancest` has no free-text field |
| Write to the dialogue state machine | ❌ **do not** | reported segfaults |
| Background simulation across days | ⚠️ **unverified** | see § 7 |

**One sentence:** you can read everything, render anything, and move people around — but you
cannot edit what an NPC says, because what they say was never stored as text.

---

## 1. The dialogue system is fully readable

Source: [`ma9o/df-llm`](https://github.com/ma9o/df-llm), `dfharness/interactions.lua`.

```lua
local c = df.global.game.main_interface.adventure.conversation
```

Readable fields:

- `c.open` — is a conversation open
- `c.selecting_conversation` — is the player picking a target
- `c.conv_act` — the conversation activity
- `c.conv_actce` — the `conversationst` event object
- `c.conv_choice_info` — **every currently selectable dialogue option**
- `c.tact_cci` — the current tact topic
- `c.conv_string_filter` / `c.entering_conv_string_filter` — the conversation text filter
- `c.tact_topic.tact_required`

Type enums:

- `df.talk_choice_type` — topic kinds (`Interrogate`, `AskAboutHf`, `AskForDirectionsToHf`,
  `FishForMaster`, `FishForPlots`, …)
- `df.conversation_tact_type` — speech acts (`Persuade`, `Intimidate`, …)
- `df.conversation_state_type` — conversation states (`AskDirections`, …)

**Turn history** (`activity_summary()`):

```lua
event.turns[i]   -- { speaker=turn.speaker, type=turn.type, year=…, ticks=… }
event.participants[i].unit_id
event.floor_holder
#event.turns
```

Who said what kind of thing, and when. Limit: `conversation_turns = 128`.

---

## 2. NPC identity is richly structured

Source: [`ma9o/df-llm`](https://github.com/ma9o/df-llm), `dfharness/character_details.lua`.

The module exports 22 sections:

```
abilities  affiliation  appearance  career      combat  companions  conditions
coverage   history      knowledge   native_sheet obligations      performance_skills
physiology possessions  preferences relationships reputation       senses
```

The ones that matter for playing a character:

- **personality** — native personality vector
- **relationships** — the relationship graph (`hf_visual` / `hf_historical` / `hf_identity`)
- **history** / **knowledge** — what this person has lived through and knows
- **reputation**, **preferences**, **career**, **obligations**
- **native_sheet** — raw attributes

### 2.1 Where mood actually lives

`unit.status.current_soul.personality.stress` — confirmed by DwarfTalk's `adjust_mood`
implementation (§ 6.1).

---

## 3. Writing: what works, what does not

### 3.1 The official precedent — DFHack rewrites dialogue itself

Source: [`DFHack/scripts`](https://github.com/DFHack/scripts),
`internal/advtools/convo.lua` (`default_enabled=true`, ships with DFHack).

This script **creates dialogue options from nothing and injects them into the menu**:

```lua
local function new_choice(choice_type, title, keywords)
    local choice = df.adventure_conversation_choice_infost:new()   -- ← construct
    choice.choice = df.talk_choice:new()
    choice.choice.type = choice_type
    local text = df.new("string")
    text.value = title                                            -- ← arbitrary string
    choice.title.text:insert("#", text)
    if keywords ~= nil then addKeywords(choice, keywords) end
    return choice
end
```

```lua
adventure.conversation.conv_choice_info:insert(
    #adventure.conversation.conv_choice_info-1, choice)          -- ← inject before "back"
```

`rumorUpdate()` runs in the `AskDirections` state and injects *every relationship the adventurer
knows* as a "where are they?" option.

### 3.2 Keyword generation

```lua
local function addKeyword(choice, keyword)
    local keyword_ptr = df.new('string')
    keyword_ptr.value = dfhack.toSearchNormalized(keyword)
    choice.keywords:insert('#', keyword_ptr)
end
local function generateKeywordsForChoice(choice)   -- derives keywords from the title text
```

Dialogue options carry a keyword table, which is what the conversation filter searches. This
gives LLM-authored topics a path to being findable.

### 3.3 Custom UI in the dialogue screen

```lua
AdvRumorsOverlay = defclass(AdvRumorsOverlay, overlay.OverlayWidget)
AdvRumorsOverlay.ATTRS{
    desc='Adds keywords to conversation entries.',
    default_enabled=true,
    viewscreens='dungeonmode/Conversation',   -- ← binds to the dialogue screen
    frame={w=0, h=0},
}
function AdvRumorsOverlay:render()
    -- detects first-entry pointer change → triggers rumorUpdate()
end
```

### 3.4 `plugins.overlay` is a real GUI framework

Source: `plugins/lua/overlay.lua`.

```lua
local _ENV = mkmodule('plugins.overlay')
local gui = require('gui')
local widgets = require('gui.widgets')
```

Built on DFHack's full `gui` framework and `gui.widgets` library, plus
`gui/dialogs.lua` which exports `showInputPrompt` / `InputBox` / `MessageBox` /
`DialogScreen` / `ListBox`, and `gui/widgets/edit_field.lua` / `text_area.lua`.

**Not just "a small widget" — arbitrary panels, labels, input fields, multiline text.**

### 3.5 Official framing

`docs/advtools.rst` describes `advtools.conversation` as an **overlay feature enhancement** —
"add additional searchable keywords to conversation topics", "add additional conversation options
for asking whereabouts of your relationships".

It is content/UI enhancement, not a cheat. Work built on the same mechanism has the same character.

### 3.6 Free text into the announcement area

`adv_announcementst` has a free-text `stl-string str`, renderable via
`showPopupAnnouncement` — a second, simpler path for displaying generated text.

### 3.7 🔴 CORRECTION: NPC lines have no writable text field

Source: `DFHack/df-structures`, `df.activity.xml` L350–362. Full definition of `utterancest` —
the structure representing **each thing an NPC says**:

```xml
<struct-type type-name='utterancest'>
    <int32_t name='speaker' ref-target='unit'/>
    <int32_t name='speaker_hfid' ref-target='historical_figure'/>
    <enum type-name='talk_choice_type' name='type'/>
    <compound type-name='entity_event' name='rumor'/>
    <int32_t name='incident_id' ref-target='incident'/>
    <int16_t name='foreground'/><int16_t name='background'/><int16_t name='bright'/>
    <int32_t name='year'/><int32_t name='ticks'/>
</struct-type>
```

**No free-text field of any kind.** The whole of `df.activity.xml` contains exactly one
`stl-string`, and it belongs to an unrelated `get_idle_string` vmethod.

DF assembles the sentence at **render time** from the enum plus the rumor data. It is never
stored as text.

> **This corrected an earlier draft of this research**, which asserted that because `convo.lua`
> can write `title.text`, arbitrary text could be written into the conversation. The two are
> different structures: `convo.lua` writes **player-side options**; `utterancest` is **NPC-side
> speech**, and it has no text to write.

**Consequence: "replace what the NPC says" is off the table. Take over the *presentation*
instead.**

### 3.8 But the same structure is a goldmine of meaning

`utterancest` carries `rumor` (a complete `entity_event`) and `incident_id`.

**The NPC's meaning — which event, whom it concerns, what happened — is fully structured.**

```
read type + rumor + incident_id   → what the NPC is communicating
layer character_details           → who they are
model writes the line
overlay / adv_announcementst.str  → render it
```

Better than replacement: it obeys the "never write the state machine" rule for free.

### 3.9 A second goldmine: `talk_choice`

Source: `df.activity.xml` L426–448. The semantic layer of a dialogue option, a large union:

`adventure_desire` / `opinion` / `trouble_type` / `emotion` / `value_type` / `belief_system_id` /
`invocation_target_hfid` / `banter_item_id` / `squad_order_type` / `agreement_id` /
`sleep_permission_zone` / `main_relevant_id` …

This is *better* input for a model than the rendered English would be.

---

## 4. The segfault line

Source: [`magnus-ISU/dfhack-commands`](https://github.com/magnus-ISU/dfhack-commands),
`dfhack/adv/keep-talking.lua` — a third-party script that auto-reopens conversations.

> "HOW IT REOPENS — all through DF's own input, **no fragile state writes (those segfault)**"

Its accumulated traps:

1. **Writing dialogue state directly segfaults.** Reopening must go through DF's own input:
   `gui.simulateInput(dfhack.gui.getDFViewscreen(true), key)`.
2. **DF zeroes state on reopen.** `choice_scroll_position` is reset while the panel still reads as
   open — the native UI was never designed for external incremental rebuild. The script must
   RE-ASSERT for 400 ms.
3. **UI state is unstable across frames.** On the frame a topic is clicked, the scroll position is
   zeroed while the panel still reports open; a saved value is only adopted after surviving two
   consecutive frames.
4. **Overlay lifecycle matters.** Must register as an overlay and implement `overlay_onupdate`;
   click and reopen differ by a frame (`simulateInput` runs screen logic synchronously, key-fed
   reopen lands a frame later).
5. **Screen text is readable character by character** via `df.global.gps` (`dimx`/`dimy`).
6. **Environment isolation:** overlays and RPC `reqscript` can hold *different* script envs; share
   debug state through `_G` (`rawget(_G, …)`).

**Rule: never touch the dialogue state machine. Render in an overlay; drive input through DF's
own path.**

---

## 5. External channel & event hooks

### 5.1 Transport

- RPC over TCP, default port 5000 (`df-llm` uses 5001 on macOS where 5000 is taken)
- `remote` plugin + `dfhack-run`
- `RunCommand` accepts arbitrary commands including `lua <code>` and `lua --file <path>`
  (write to disk + trigger — best fit for an LLM bridge)
- Clients exist: [`dfhack-client-python`](https://github.com/McArcady/dfhack-client-python),
  [`dfhack-remote-node`](https://github.com/alexanderolvera/dfhack-remote-node)
- `luasocket` is available for DFHack to dial *out*

### 5.2 Event hooks

`eventful` exposes 16 event types (TICK / JOB_* / UNIT_DEATH / REPORT / INTERACTION / …) —
**none of them dialogue- or UI-related.**

But `SC_VIEWSCREEN_CHANGED` fires every frame (source: `library/Core.cpp:1532-1596`), and
combined with `dfhack.gui.matchFocusString('dungeonmode/Conversation')` it **can capture "player
entered a conversation" event-driven**, without polling.

Conversation *content* changes (the option list changing) can only be polled — as the official
`convo.lua` does, by comparing the first-entry pointer.

### 5.3 Two dead ends

| Path | Verdict |
|---|---|
| `RemoteFortressReader` adventure-mode RPC | ❌ **commented out wholesale** (`plugins/remotefortressreader/adventure_control.cpp`); adventure mode did not exist at v50 and was never restored |
| `custom-raw-tokens` | ❌ a **read-only** token reader, not a write path |

---

## 6. Prior art

### 6.1 DwarfTalk — the only prior DF attempt

[`CarlosNahuelcoy/dwarftalk`](https://github.com/CarlosNahuelcoy/dwarftalk) (2026-02, Steam
Workshop, Go bridge to player2.game).

LLM replies become real game effects:

```lua
local handlers = {
    change_job        = action_engine.change_job,
    adjust_mood       = action_engine.adjust_mood,
    create_work_order = action_engine.create_work_order,
    refuse_work       = action_engine.refuse_work,
    assign_military   = action_engine.assign_military,
}
```

`adjust_mood` writes mood directly:

```lua
local soul = unit.status.current_soul
local old_stress = soul.personality.stress or 0
local stress_delta = change * -50000
local new_stress = math.max(...)
```

**Three things to know:**

1. ⚠️ **The Workshop page claims adventure mode; the code does not support it.**
   Every one of the eight core Lua/Go source files was read: the string `adventure` appears
   **zero times**. All five actions are fortress-mode concepts. The claim rests on the author's
   Workshop text alone.
2. ⚠️ **Its persistence destroys non-ASCII.** `persistence.lua`:
   ```lua
   text:gsub("[^\32-\126\n\t]", "")   -- deletes all CJK
   ```
   Do not copy this. Store UTF-8 as UTF-8.
3. ⚠️ **Trust crisis.** The bundled `.exe` is flagged by Defender/ESET/VirusTotal; Workshop
   comments report users refusing it as unauditable.

### 6.2 The neighbouring successes

**[RimTalk](https://github.com/jlibrary/RimTalk)** (RimWorld, 86★, active) — the mature pattern.
Four-stage pipeline: intercept → assemble context on the main thread → API call off-thread →
parse and render back on the main thread. Memory: 25 entries per pawn, half-life decay (4 days,
7 for trauma). Its context field list is directly reusable:

- pawn: name/sex/age/xenotype/genes/backstories/traits/ideology/faction/role/job/skills/health/
  mood%/mental state/recent thoughts/social relations and opinions/equipment
- environment: hour/date/quadrum/season/year/outdoor temp/weather/room stats/terrain
- events: raids, psychic phenomena, solar flares, quest letters, map notifications

**[Cataclysm-AOL](https://github.com/josihosi/Cataclysm-AOL)** (Cataclysm: DDA, 19★, active) —
**the closest architecture to this project.** `src/llm_intent.cpp`: the model chooses an
**intent**, then the game's *native AI* executes up to 3 consecutive actions. The author's
roadmap states the goal outright — free text and tool calls replacing preset branches.

Two field notes from that project worth more than their code:

- Prompt tuning matters enormously. Changing *"You have decided to team up with the player for
  now, and must answer as the NPC."* to *"You must answer as the NPC."* made the model sometimes
  attack the player instead.
- 10–20 s latency on modest hardware → the LLM call is made **async** and injected when ready.

### 6.3 The rest

[`df-llm`](https://github.com/ma9o/df-llm) — adventure-mode LLM harness, structured state +
semantic actions, tested on DF 53.16 / DFHack 53.16-r1.1. The **read layer, already solved**.
Note its `text` action is capped at 1–200 **printable ASCII** characters — a low-level keyboard
primitive, *not* free speech.

[`fort-gym`](https://github.com/lemoz/fort-gym) — autonomous-agent benchmark harness.
[`llm_dwarf_fortress_eval`](https://github.com/ChromiteExabyte/llm_dwarf_fortress_eval) —
"let an LLM learn Dwarf Fortress", $10 budget guard.
[`dwarf_fortress_mcp`](https://github.com/Dicklesworthstone/dwarf_fortress_mcp) (23★) —
semantic transactional control plane, but **explicitly read-only**: "no command, Lua, keyboard,
arbitrary RPC, direct memory-write, filesystem, or mutation route".
[`it_was_inevitable`](https://github.com/BenLubar/it_was_inevitable) (2021) — "Sentient Dwarf
Fortress was inevitable." A bot narrating DF logs. The idea has been waiting five years.

**Every one of these is AI playing Dwarf Fortress. None is AI living in it.**

### 6.4 The community is hostile to this

`r/dwarffortress`'s "AI modding rules" thread was locked by moderators. DwarfTalk's reception is
dominated by refusal to run an unauditable binary. The DFHack repo's Discussions contain **zero**
LLM-dialogue threads; Bay12 forum searches for chatbot/LLM/AI-dialogue return nothing relevant.

This is a context note, not a technical obstacle.

---

## 7. Movement — proven, with one open question

### 7.1 Sending an NPC somewhere works, and it is real pathfinding

Source: `DFHack/scripts/gui/companion-order.lua`.

```lua
function move_unit( unit,tx,ty,tz )
    unit.idle_area.x=tx; unit.idle_area.y=ty; unit.idle_area.z=tz
    unit.idle_area_type=df.unit_station_type.Commander
    unit.follow_distance=50
    unit.path.dest=copyall(unit.idle_area)
    unit.path.goal=df.unit_path_goal.SeekStation   -- ← native pathfinding
    unit.path.path.x:resize(0)                     -- clear stale path
end
```

Set a station, set `SeekStation`, and **DF's own AI walks them there**. Not a teleport.

Official orders (`docs/gui/companion-order.rst`):

| Order | Effect |
|---|---|
| `:move` | Move to a location (capped at 3 tiles from you if following) |
| `:equip` / `:pick-up` / `:unequip` / `:unwield` | Item handling |
| `:wait` | Temporarily leave the party |
| `:follow` | Rejoin |
| `:leave` | Permanently leave (**rejoinable by talking**) |
| `-c` | Cheats: `patch up`, `get in` |

`modtools/create-unit` / `spawnunit` can also create units.

### 7.2 The native intent vocabulary

**`unit_station_type`** (44 entries): `Commander` / `MeetingLocation` /
`MeetingLocationBuilding` / `SeekCommander` / `Guard` / `WaitOrder` / `MoveToSite` /
`ClaimSite` / `PatternPatrol` / `AmbushPatrol` / `LairHunter` / `Graze` / `Owner` / …

**`unit_path_goal`** (~100 entries, excerpt): **`AdventureMove`** / `ConflictDefense` /
`SeekUnitForJob` / `GoToGiveWaterTarget` / `SleepBed` / `SeekFoodForTarget` /
`CarryPatientToBed` / `ThiefTarget` / `FleeTerrain` / …

There is **no CDDA-style needs/mission/faction decision tree** in adventure mode. There is this
vocabulary instead — the model's job is to **choose among native intents**, not to replace the
decision logic. Expressiveness is sufficient: "wait at the tavern door" = `AdventureMove` plus a
target, and `MeetingLocation` is a literal match.

### 7.3 🔴 The open question: does it survive time?

**Verified:**
- Native `wait` exists — wiki: *"Hit `.` allows you to stay in one place and **wait for other
  things to move**"* → NPCs act while the player waits.
- Fast travel shows other creatures moving as blue dots → time passing moves creatures.
- `df-llm`'s `wait` action documents: *"advances game time and settles"*.

**Not verified:**
- Whether a specific NPC **holds a travel goal** while the player is **absent from the area**
- Whether scheduling is stable across **day boundaries**
- Whether adventure mode has any **background simulation** (fortress mode does; adventure mode
  is unknown)

This is the single largest unknown in the project. It is the load-bearing assumption under the
"meet me tomorrow" scenario. An experiment is designed for it (local, not shipped) and must run
**before** M4 is designed.

### 7.4 Agreements exist natively, and are probably the wrong tool

`df.agreement.xml` — `agreement_details_type` includes `JoinParty` (originally
`JOIN_AS_COMPANION`) / `Parley` / `OfferService` / `Residency` / `Citizenship` /
`RetrieveArtifact` / `PlotAssassination` / …

`agreement_details_data_join_party` carries `member` / `party` / `site` / `figure` /
`end_year` / `end_season_tick` — including an expiry.

`df.activity_meeting.xml` — `activity_info` has `unit_actor` / `unit_noble` / `place` (civzone) /
`delay` / `meeting_done`.

**But:** these structures serve fortress-mode diplomacy and are semantically weighted toward
conspiracy. **Recommendation: manage agreements in `persistent` storage instead** — more
controllable, less coupling.

---

## 8. Memory: officially supported

`dfhack.persistent` — stores tool state **in the world savegame directory**:

```
saveWorldData / getWorldData / saveWorldDataString / getWorldDataString
saveSiteData  / getSiteData  / saveSiteDataString  / getSiteDataString
deleteWorldData / deleteSiteData
```

```lua
local memory = dfhack.persistent.getWorldDataString('soulforge/memory/'..unit_id)
```

World-dependent state belongs here; world-independent state belongs in
`dfhack-config/`. **Per-NPC memory surviving saves is a supported operation.**

---

## 9. Time and latency

The concern that a model cannot keep up with turn-based input is real in general, and does not
apply here.

| Fact | Source | Consequence |
|---|---|---|
| The adventurer's keypresses are internally called **"instants"**; one key = one tick | DF Wiki / Time | high-frequency input genuinely exists |
| **Purely turn-based**; the real-world clock does not participate | DF Wiki / Time | an idle game costs an idle model |
| Dialogue is a **modal menu**; the world does not advance while it is open | DF Wiki / Talking | ⭐ **latency is absorbed by the freeze** |
| Native `wait`/`rest` advance multiple ticks and settle activity | `df-llm` capabilities | a clean "let time pass" primitive |
| Fast travel crosses large spans but **cannot be done while talking** | DF Wiki / Adventure mode | mutually exclusive with dialogue; harmless |

**Conclusion:** the model is only ever invoked inside a frozen window. A three-second call costs
zero game time. The real risk is not latency but **races** — the player acting in the menu while
generation is in flight, colliding with DF's own state-reset windows (§ 4). Mitigation: render in
an overlay, and lock or interrupt-ably gate input during generation.

⚠️ **Unverified:** whether DF responds to `SetPause` in an adventure-mode context. The known pause
APIs are fortress-mode oriented. Not required — the modal dialogue is sufficient.

---

## 10. Chinese output

Two independent failure modes.

**1. Storage.** DwarfTalk's sanitizer deletes all non-ASCII (§ 6.1). Store UTF-8 as UTF-8; use
`saveWorldDataString`, not a JSON round-trip through a text sanitizer.

**2. Rendering.** DF's native bitmap font has no CJK glyphs.
[`wodzys/dwarf-fortress-chinese`](https://github.com/wodzys/dwarf-fortress-chinese) is a DFHack
plugin that localizes the entire UI in real time **from game memory**, and renders Chinese via
**dynamically loaded SDL2_ttf** with multiple sizes and rendering modes. It hooks the render
layer, which is the same layer an overlay draws into.

Verify this early. It is a cheap check that invalidates the display strategy if it fails.

---

## 11. Version targeting

| Component | Version |
|---|---|
| DFHack stable | **53.16-r1.1** (released 2026-08-08) |
| Dwarf Fortress | **53.16** |
| Next DFHack | 53.16-r2 (rc2 as of 2026-09-27) |

`df-llm` was tested on exactly DF 53.16 / DFHack 53.16-r1.1 — the current stable pair. DF and
DFHack versions must match; DFHack does not tolerate skew.

---

## 12. Open questions

- [ ] Does a travel goal survive player absence and day boundaries? (§ 7.3) — **experiment designed**
- [ ] Can an NPC be *pinned* at a destination on arrival, or do they wander off? (`WaitOrder` /
      `MeetingLocation` need testing)
- [ ] Is free-text player input possible? `conv_string_filter` exists and suggests yes; unverified
- [ ] Does `SetPause` work in adventure mode?
- [ ] How far can `choice.type` deviate from the native enum before the game rejects it?
- [ ] Are there per-server rate limits / terms issues with the chosen model provider for
      high-frequency short calls?
- [ ] Does an overlay survive the dialogue screen being rebuilt mid-generation?

---

## Appendix: sources

**Repositories**
- `DFHack/dfhack`, `DFHack/scripts`, `DFHack/df-structures` — engine, official Lua, structure defs
- `ma9o/df-llm` — adventure-mode LLM harness (read layer)
- `magnus-ISU/dfhack-commands` — `adv/keep-talking.lua` (segfault field notes)
- `CarlosNahuelcoy/dwarftalk` — the only prior DF LLM-NPC attempt
- `jlibrary/RimTalk`, `josihosi/Cataclysm-AOL` — the neighbouring successes
- `wodzys/dwarf-fortress-chinese` — CJK rendering

**Key source files**
- `DFHack/scripts/internal/advtools/convo.lua` — writing dialogue options
- `DFHack/scripts/gui/companion-order.lua` — `move_unit()`, companion orders
- `plugins/lua/overlay.lua`, `library/lua/gui/dialogs.lua` — UI framework
- `df.unit.xml` L2633/2640, `df.d_basics.xml` L3897/3943 — intent enums
- `df.activity.xml` L350–362 — `utterancest` (no text field), L426–448 `talk_choice`, L594 `activity_event_conversationst`
- `df.adventure.xml` — `adventure_conversation_choice_infost`
- `df.agreement.xml`, `df.activity_meeting.xml` — agreements and meetings
- `library/Core.cpp:1532-1596` — `SC_VIEWSCREEN_CHANGED`

**Docs**
- https://docs.dfhack.org/en/stable/docs/dev/Lua%20API.html
- https://docs.dfhack.org/en/stable/docs/dev/Remote.html
- https://docs.dfhack.org/en/stable/docs/tools/advtools.html
- https://docs.dfhack.org/en/stable/docs/dev/overlay-dev-guide.html
- DF Wiki: [Time](https://dwarffortresswiki.org/index.php/Time) ·
  [Talking](https://dwarffortresswiki.org/index.php/Talking) ·
  [Adventure mode](https://dwarffortresswiki.org/index.php/Adventure_mode)

**Scraping notes for whoever continues this:** `raw.githubusercontent.com` is unreachable from
this network; use `gh api repos/{owner}/{repo}/contents/{path}` (authenticated, 5000/hr vs 60/hr
anonymous — the anonymous limit was exhausted before this was noticed). Bay12 uses Anubis
proof-of-work and needs a rendered fetch. Reddit's `search.json` and `old.reddit.com` 403, but a
rendered fetch of `www.reddit.com/search` works.
