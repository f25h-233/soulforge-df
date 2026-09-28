# MVP — Forging a Voice

> The blueprint. What gets built first, how the modules fit, and what "done" means.
> Evidence for every claim lives in [RESEARCH.md](RESEARCH.md).

---

## 1. The vision, in stages

The end goal is an NPC you can make a promise to. The path there is five stages, and each one
ships something usable on its own.

```
MVP   Voice      ── the NPC says something in character
 M1   Options    ── you can say something back that the model wrote
 M2   Memory     ── they remember it tomorrow
 M3   Movement   ── they walk to where you agreed to meet
 M4   Agreements ── "tomorrow" actually arrives
```

**MVP ships stages 1–2 only.** Not because the rest is unbuildable, but because the rest
depends on things we have proven less well. Read [§ 6](#6-what-we-are-not-promising-yet) before
believing the roadmap.

---

## 2. Design principle

> **The model is a director, never a puppeteer.**

It reads state, writes prose, and chooses from a vocabulary the game already understands. It does
not replace the dialogue state machine, and it does not invent game mechanics.

Two consequences that shape every module:

1. **Generation is always driven by native semantics.** The model is told *what the NPC is
   actually communicating* (`utterancest.type` + `rumor` + `incident_id`) and *who they are*
   (`character_details`). It rewrites the delivery, not the content. A model that invents facts
   contradicts the game and breaks the illusion.
2. **Game writes go through a whitelist.** Only fields with a verified precedent are written.
   Everything else is read-only. See [§ 5](#5-safety-boundaries).

---

## 3. Module layout

Decoupled so that each stage of the roadmap is *an addition*, never an edit to a core loop.

```
soulforge-df/
├── lua/                      # runs inside DFHack
│   └── soulforge/            #   Lua-side: reads state, renders, writes back
└── src/                      # runs outside the game (the bridge)
    ├── core/                 #   transport, model client, memory store
    ├── contexts/             #   state → prompt brief
    ├── renderers/            #   generated text → screen
    └── actions/              #   model intent → game write
```

### The boundary that matters

`src/contexts/` and `src/renderers/` never write game state.
`src/actions/` never talks to the model.
`core/` knows about neither.

A new capability is a new module in `contexts/` or `actions/` — registered, not wired in. The
movement layer (M3) will be exactly this: a `NpcMover` action plus a `WorldContext` provider,
with no changes to `core/` or to the dialogue path.

---

## 4. MVP scope

### 4.1 Module contracts

These are written now, in full, even though only two are implemented in the MVP — because the
contracts are what make the roadmap additive rather than invasive.

```python
# src/core/ — knows nothing about Dwarf Fortress

class Transport(Protocol):
    """Bidirectional channel to DFHack. TCP 5000 RPC."""
    def read(self, request: dict) -> dict: ...
    def exec_lua(self, code: str) -> dict: ...

class ModelClient(Protocol):
    """Any chat-completion backend."""
    def complete(self, *, system: str, messages: list[dict]) -> str: ...

class MemoryStore(Protocol):
    """Per-NPC persistent state. Survives saves."""
    def get(self, npc_id: str, key: str) -> str | None: ...
    def set(self, npc_id: str, key: str, value: str) -> None: ...
```

```python
# src/contexts/ — read-only. Never writes.

class ContextProvider(Protocol):
    """Contributes a section to the prompt brief."""
    name: str
    def gather(self, npc_id: str) -> dict: ...

# MVP providers:
#   IdentityContext   ← character_details.lua (personality, relationships, history)
#   UtteranceContext  ← utterancest.type + rumor + incident_id
#   SceneContext      ← location, time, who else is present
# Roadmap providers:
#   WorldContext      ← recent world events, legends   (M2)
#   PartyContext      ← companions, obligations        (M3)
```

```python
# src/actions/ — writes only through verified fields.

class Action(Protocol):
    """A capability the model may invoke."""
    name: str
    schema: dict          # JSON schema, given to the model as a tool
    def apply(self, npc_id: str, args: dict) -> None: ...

# MVP actions:
#   AdjustMood        → unit.status.current_soul.personality.stress
#                       (whitelisted: precedent in DwarfTalk's adjust_mood)
#   RecordInteraction → MemoryStore (rapport delta, notable lines)
# Roadmap actions:
#   InjectOption      ← M1, the advtools/convo.lua new_choice() pattern
#   NpcMover          ← M3, move_unit() + SeekStation
#   FormAgreement     ← M4, pending the time-span experiment
```

```python
# src/renderers/ — display only. Never writes, never calls the model.

class Renderer(Protocol):
    def show(self, npc_id: str, text: str, *, speaker: str) -> None: ...

# MVP renderers:
#   OverlayRenderer    ← registered overlay on 'dungeonmode/Conversation'
#   AnnouncementRenderer ← adv_announcementst.str / showPopupAnnouncement
#   ConsoleRenderer    ← development
```

### 4.2 The MVP flow

```
player opens a conversation and picks a topic
        │
        ▼
[SC_VIEWSCREEN_CHANGED]  ── event-driven, no polling
        │  matchFocusString('dungeonmode/Conversation')
        ▼
ContextProvider.gather()
        │  IdentityContext + UtteranceContext + SceneContext
        ▼
prompt brief  ──►  ModelClient.complete()   ── async, world is frozen
        │
        ▼
[model returns line + optional action calls]
        │
        ├──▶ Renderer.show(line)              ← the player sees prose
        │
        └──▶ Action.apply()                   ← whitelisted writes only
                 AdjustMood / RecordInteraction
```

**The async question is a non-issue here** — see [RESEARCH.md § 9](RESEARCH.md). The dialogue
menu is modal; the world does not advance while it is open. A 3-second model call costs 0 game
time. We still make it async for UI responsiveness, but we are not racing the simulation.

### 4.3 Acceptance criteria

The MVP is done when, in a real adventure-mode save:

- [ ] **AC1** — Talking to a town NPC produces a line that reflects their `character_details`
      (a paranoid dwarf sounds paranoid; a dwarf who lost family to goblins is not cheerful
      about goblins). *Not* generic fantasy filler.
- [ ] **AC2** — The line is **consistent with what the NPC actually said natively**. Ask about a
      rumor they have; the generated prose must be about that rumor. This is the anti-hallucination
      test and the most important one.
- [ ] **AC3** — The rendered line appears in the dialogue screen **without disrupting the native
      menu** — the player can still pick options, Esc still works, nothing segfaults.
- [ ] **AC4** — Repeated interaction shifts rapport: `personality.stress` moves, and the shift is
      observable in subsequent interactions.
- [ ] **AC5** — Opening 50 conversations in a row does not crash the game.
- [ ] **AC6** — Chinese output renders correctly (see [§ 5.3](#53-chinese-output)).

### 4.4 Non-goals for the MVP

Explicitly **not** in scope, so they do not creep:

- ❌ Player free-text input (that is M1)
- ❌ Anything surviving a save (that is M2)
- ❌ Any movement or position write (that is M3)
- ❌ Replacing native NPC speech (impossible — see RESEARCH.md § 3.7)
- ❌ Any change to the dialogue state machine (dangerous — see § 5.1)

---

## 5. Safety boundaries

### 5.1 The segfault line

Directly writing dialogue state is reported to crash the game. The `adv/keep-talking.lua`
author, who has been deeper into this than anyone:

> "all through DF's own input, **no fragile state writes (those segfault)**"

**Rule: the dialogue state machine is read-only.** We render in an overlay and drive input
through DF's own path (`gui.simulateInput`). We do not write `conv_choice_info` except via the
exact `advtools/convo.lua` pattern, which has an official precedent.

There is a second, subtler trap: DF **zeroes `choice_scroll_position`** when a conversation
reopens. The native UI was never designed for external incremental rebuild. Expect to discover
more of these; budget for them.

### 5.2 The write whitelist

| Field | Why it is allowed |
|---|---|
| `unit.status.current_soul.personality.stress` | Precedent: DwarfTalk's `adjust_mood` writes it |
| `dfhack.persistent.*` | Official API, designed for exactly this |
| `adv_announcementst.str` | Official, free-text by design |
| Overlay widget registration | Official `plugins.overlay` |

Everything else is read-only until individually verified in the live game.

### 5.3 Chinese output

`soulforge-df` is built by a Chinese developer and should speak Chinese.

Two things that will silently ruin it if ignored:

1. **Do not copy DwarfTalk's persistence.** Its `sanitize_text` strips every non-ASCII character:
   ```lua
   text:gsub("[^\32-\126\n\t]", "")   -- ← this deletes all Chinese
   ```
   Store UTF-8 as UTF-8. Use `dfhack.persistent.saveWorldDataString`, not a JSON round-trip
   through a sanitizer.

2. **Font rendering.** DF's native bitmap font has no CJK glyphs. The
   [wodzys/dwarf-fortress-chinese](https://github.com/wodzys/dwarf-fortress-chinese) DFHack
   plugin solves this — it hooks the render layer — and dynamically loads SDL2_ttf. Either
   depend on it or do the same thing. **Verify early; it is cheap to check and expensive to
   discover late.**

---

## 6. What we are *not* promising yet

### M4 is load-bearing and unproven

The best demo this project could ever have is: *you make a promise to an NPC in a tavern, and the
next day they are waiting where they said they would be.*

It is also the **only** part of this roadmap with no evidence behind it.

**Proven:** an NPC can be given a travel goal and will genuinely pathfind there
(`move_unit()` → `path.goal = SeekStation`, native AI does the walking — not a teleport).
**Proven:** companions can be ordered to follow, wait, or leave (`gui/companion-order`).
**Unproven:** whether that goal *survives the player leaving the area and days passing.*

The honest possibilities:

| Outcome | Consequence |
|---|---|
| Goal holds across days | M4 proceeds as designed. |
| Goal holds only while the player watches | Fall back to an anchor-based design: the NPC is *placed* when the player arrives. Narratively still "he kept his word"; technically a summon. |
| No background simulation at all | "Meeting tomorrow" must be reframed as "meet me here in a moment" — same scene only. |

**The experiment is designed and kept locally (not shipped).** We run it before designing M4, not after. If it fails, the repo updates its claims —
we would rather correct a roadmap than ship a broken promise.

### The rest of the honest list

- The movement layer's *arrival* behaviour is open: an NPC that walks to a spot may simply wander
  off. Pinning them there (`WaitOrder` / `MeetingLocation` station types) needs its own test.
- `df.agreement.xml` exists and is tempting, but it serves fortress-mode diplomacy and is
  semantically a poor fit. **We will manage agreements ourselves** via `persistent` rather than
  reuse native structures.
- The reverse-engineering surface is large. Every new written field is a new crash risk until
  tested.

---

## 7. Why not just use an existing tool

| Existing | Why it does not do this |
|---|---|
| [DwarfTalk](https://github.com/CarlosNahuelcoy/dwarftalk) | Fortress mode. Its Workshop page claims adventure mode; `adventure` appears 0 times in its 8 core source files. |
| [df-llm](https://github.com/ma9o/df-llm) | An *agent playing* the game, not a character in it. Excellent read layer — we should use its patterns. |
| [fort-gym](https://github.com/lemoz/fort-gym) | Benchmark harness for AI play. |
| [dwarf_fortress_mcp](https://github.com/Dicklesworthstone/dwarf_fortress_mcp) | Explicitly read-only by design — no mutation route. |
| The MCP ecosystem generally | Built to let an agent *operate* a fortress. Not to give a character a voice. |

The gap is real and it is specific: **everyone is building AI that plays Dwarf Fortress. Nobody
is building AI that lives in it.**

---

## 8. Build order

```
1. Transport + fake NPC                        ← no game needed, unblocks everything
2. IdentityContext against a real save          ← AC1
3. UtteranceContext + anti-hallucination check  ← AC2, the hard one
4. OverlayRenderer on a real conversation       ← AC3
5. AdjustMood + RecordInteraction               ← AC4
6. Soak test                                    ← AC5
7. CJK renderer verification                    ← AC6
────────────────────────────────────────────────
   → MVP done. Then M1 (options), M2 (memory), M3 (movement).
   → M4 only after the experiment.
```

Step 7 is listed last but **should be smoke-tested first** — it is a 15-minute check that
invalidates the whole display strategy if it fails.
