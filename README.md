# soulforge-df

**Forge souls for Dwarf Fortress NPCs.**

Give the people you meet in Dwarf Fortress adventure mode a mind worth talking to — their own
voice, their own memory, their own reasons to walk to the tavern and wait for you.

> **Status: research complete, MVP not yet started.**
> Everything claimed here is backed by source code or official documentation.
> See [docs/RESEARCH.md](docs/RESEARCH.md) for the full evidence chain, and
> [docs/MVP.md](docs/MVP.md) for what gets built first.

---

## The idea

Dwarf Fortress generates worlds with centuries of history, thousands of historical figures,
and NPCs that already know who they are, who they hate, and what they have survived. What it
does *not* do is let them **say** any of it in their own words. Every utterance is assembled at
render time from an enum and a rumor record.

`soulforge-df` reads that structure — the meaning, not the pixels — and hands it to a language
model that plays the character. The NPC's *semantics* stay native. Only the **voice** is new.

```
read what the NPC is actually communicating
    → utterancest.type + utterancest.rumor + incident_id
    → talk_choice opinion / emotion / value_type
layer in who they are
    → character_details: personality, relationships, history, knowledge, reputation
let a model write the line
render it in the dialogue screen
write back what should persist
    → personality.stress  (mood, rapport)
```

## Why this is possible

Three things make Dwarf Fortress an unusually good target — and none of them are obvious.

**1. It is turn-based, and dialogue freezes the world.**
Nothing moves unless the player presses a key. Opening a conversation opens a *modal menu*, and
while it is open the world does not advance a single tick. A model that takes three seconds to
answer costs **zero game time**. The "AI is too slow to keep up" problem dissolves — the
plausible-sounding objection is answered by the game's own 2006-vintage UI design.

**2. DFHack exposes the meaning, not just the state.**
This is the point. You do not need computer vision, screenshots, or OCR — and you should not use
them. `DFHack` hands you structured access to:

| What | Where |
|---|---|
| Dialogue options + their semantics | `df.global.game.main_interface.adventure.conversation` |
| What an NPC is actually saying | `utterancest.rumor` (a full `entity_event`) + `incident_id` |
| Who an NPC is | `character_details.lua` — 22 sections incl. personality, relationships, history |
| Where an NPC can be sent | `unit_path_goal` (~100 goals) + `unit_station_type` (44 stations) |
| Memory that survives saves | `dfhack.persistent.saveWorldData` |

**3. The community has already proven the pattern — just not here.**
[RimTalk](https://github.com/jlibrary/RimTalk) (RimWorld, 86★) and
[Cataclysm-AOL](https://github.com/josihosi/Cataclysm-AOL) (Cataclysm: DDA) both ship
LLM-driven NPC dialogue today. The latter is the closest to this project's architecture: the model
chooses an **intent**, then hands it to the game's *native AI* to execute. See
[docs/RESEARCH.md § 6.2](docs/RESEARCH.md#62-the-neighbouring-successes) for what to steal from each.

## Architecture

The model is a **director**, never a puppeteer. It reads state, writes prose, and chooses from a
vocabulary the game already understands. It never replaces the dialogue state machine.

```
                    ┌─────────────────────────────┐
   DFHack Lua  ────▶│  contexts/   read + assemble │
   (game state)     └──────────────┬──────────────┘
                                   │  structured brief
                    ┌──────────────▼──────────────┐
                    │  core/       transport + LLM │──── HTTP ────▶ model
                    │              + memory        │
                    └──────────────┬──────────────┘
                                   │  generated line / intent
              ┌────────────────────┼────────────────────┐
              │                    │                    │
     ┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
     │ renderers/      │  │ actions/        │  │ (reserved)      │
     │ overlay +       │  │ mood, relation, │  │ mover           │
     │ announcements   │  │ dialogue opts   │  │ → roadmap       │
     └─────────────────┘  └─────────────────┘  └─────────────────┘
```

Modules are **decoupled by design** — a renderer never touches game writes, an action never
touches the model client. Add a new capability by adding a module, not by editing a core loop.
The movement layer (roadmap) plugs into the same `actions/` boundary.

## Roadmap

| Stage | Scope | Status |
|---|---|---|
| **MVP** | Voice: re-render native dialogue in character. Mood: write `personality.stress` so rapport actually changes. | 📋 designed ([docs/MVP.md](docs/MVP.md)) |
| **M1** | Player-side options: inject LLM-authored choices into the native menu (the `advtools/convo.lua` pattern). | planned |
| **M2** | Memory: persist per-NPC conversation history across sessions via `dfhack.persistent`. | planned |
| **M3** | ⭐ **Movement**: send NPCs to places and have them follow you (`move_unit` + `SeekStation` + `gui/companion-order`). Interface reserved from day one. | planned |
| **M4** | Agreements: "meet me at the tavern tomorrow" — depends on the time-span experiment (below). | **research blocked** |

### The one thing we are not sure about

Everything above is backed by code, **except M4**. Whether an adventure-mode NPC will *hold a
travel goal while you are away and time passes by days* is unverified. It might work. It might
only work while you watch. It might not work at all.

This is the load-bearing assumption under the best demo this project could ever have — and we
refuse to promise it before testing it. The experiment is designed and kept locally (not shipped). It runs before M4 is designed, not after.

## Honest limits

- ❌ **NPC utterances cannot be replaced.** `utterancest` has no free-text field — only an enum
  and a rumor record. DF assembles the sentence at render time. We therefore **render our own
  text in an overlay**, driven strictly by the native semantics. A model that free-associates
  will contradict the game and the illusion dies.
- ❌ **Dialogue state machines must not be written to.** Third-party tooling that tried it reports
  segfaults. We go through DF's own input path or we do not go.
- ⚠️ **Adventure mode is the empty part of the map.** [DwarfTalk](https://github.com/CarlosNahuelcoy/dwarftalk)
  is the only prior DF attempt, and despite its Workshop page claiming adventure-mode support, the
  string `adventure` appears **zero times** in its eight core source files. It is a fortress-mode tool.
- ⚠️ **DFHack's adventure-mode RPC is dead.** `RemoteFortressReader`'s adventure control was
  commented out wholesale during v50 and never restored. Build on the Lua layer, not that.
- ⚠️ **DF 53.16 / DFHack 53.16-r1.1** is the tested target. Version skew between DF and DFHack
  is not tolerated by DFHack.

## Prior art

| Project | Game | What it proves |
|---|---|---|
| [RimTalk](https://github.com/jlibrary/RimTalk) | RimWorld | The mature pattern: 4-stage async pipeline, half-life decay memory. |
| [Cataclysm-AOL](https://github.com/josihosi/Cataclysm-AOL) | Cataclysm: DDA | **Intent → native AI executes.** Closest to this architecture. |
| [DwarfTalk](https://github.com/CarlosNahuelcoy/dwarftalk) | Dwarf Fortress | LLM actions mutating real game state (`adjust_mood` writes `personality.stress`). Fortress mode. |
| [df-llm](https://github.com/ma9o/df-llm) | Dwarf Fortress | The read layer, already solved. Adventure mode harness with structured state. |
| [it_was_inevitable](https://github.com/BenLubar/it_was_inevitable) | Dwarf Fortress | 2021: "Sentient Dwarf Fortress was inevitable." The idea has been waiting. |

## Documentation

- **[docs/RESEARCH.md](docs/RESEARCH.md)** — full evidence chain: every capability, every source
  file and line, every dead end, every community finding.
- **[docs/MVP.md](docs/MVP.md)** — what gets built first, with module interfaces and acceptance
  criteria.

## License

MIT. Dwarf Fortress and DFHack are separate projects with their own licenses and are not
distributed here.
