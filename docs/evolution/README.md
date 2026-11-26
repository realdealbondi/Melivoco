# Melivoco — Evolution

This directory is the record of how Melivoco changes over time: what was decided, when, why, and what it replaced. The code shows what Melivoco *is*. This shows how it *got here*.

New entries go at the top, newest first. Each entry names the date, what changed, why, and what trade-off was accepted.

---

## 2026-11-25 — DAW host-capability correction applied; more Code Complete Architecture rules added

- **What changed:** (1) The DAW host-capability correction was applied to all three docs and pushed (commit 0bd44e6). The architecture doc's take-labelling section is now "Segments, not labels": Silent Witness detects segments by threshold and transport, probes what each host actually exposes, and degrades in tiers — Tier 0 audio-only, Tier 1 transport (the normal v1), Tier 2 labelled only where a real path exists — and detection never depends on labels. The data model removed `Take.markers`, and labels now live in a separate, nullable, provenance-tagged Label table (kind: take_name / marker / artist_tag; source: host_api / host_scripting / artist), where a take or segment with no label row is normal, not an error. The privacy doc now states take names and markers are read only where the host exposes a real path (most hosts do not), and its DAW-sync wording was reduced to the transport/playhead data JUCE actually provides. (2) Seth dictated nine further Code Complete — Architecture subsections verbatim into the architecture doc, after Resource Management — Security (2 rules), Performance (3), Scalability (1), Interoperability (1), Internationalization/Localization (3), Input/Output (1), Error Processing (8), Architectural Feasibility (1), and Over-Engineering (1) — 21 rules total, pushed in commits 71a79c6 and ee11ee9.
- **Why:** DAW host-capability research reported on 2026-11-24 showed take/clip names, boundaries, and timeline markers have no standard insert API in any plug-in format (VST3, AU, AAX, CLAP, or LV2, through JUCE or natively), so the docs had to stop assuming them; Seth approved applying the fix to all three docs on 2026-11-25. The Code Complete additions continue his verbatim dictation of the book's architecture rules into the architecture doc, as written, so the full standard sits in the document that describes how the system is built.
- **What it replaced / what was considered:** This reverses part of the 2026-11-24 "Architecture locked in" entry below — the line that Silent Witness would "automatically grab take name and track markers" from the DAW. That old entry is kept as written, per this log's rule; this entry is the correction. Transport/playhead, tempo/time signature, and sample rate remain available in every format through JUCE's AudioPlayHead PositionInfo (fields treated as optional). The alternative of assuming host features and failing silently was rejected in favour of probing per session and degrading in tiers.
- **Trade-off accepted:** Most takes in v1 will be unlabelled, and labelling quality will vary by host — accepted in exchange for a design that works across DAWs without depending on APIs that do not exist. Separating segments from labels adds a table and a provenance field to track, accepted so labels can arrive later from host scripting or the artist without changing how segments are detected or stored.

## 2026-11-24 — Code Complete Architecture rules added to the architecture doc

- **What changed:** A Code Complete — Architecture section was added to the architecture doc, with verbatim rules under five subsections: Program Organization (8 rules), Major Classes (6 rules, including the 80/20 rule), Data Design (5 rules), User Interface Design (3 rules), and Resource Management (2 rules). The additions were pushed to the repo in three steps (commits ee8b4ae, 660a634, and e84fbb3).
- **Why:** Seth directed that further Code Complete rules go into the architecture doc rather than the requirements doc, kept as the book's own rules, as written — like the periodic table — so a later builder handoff has everything it needs in the document that describes how the system is built.
- **What it replaced / what was considered:** This follows the requirements split earlier the same day, where the 26 prerequisite rules were moved out into their own requirements doc. The alternative was to keep adding all Code Complete material to that requirements doc; Seth chose the architecture doc for these architecture rules instead.
- **Trade-off accepted:** The architecture doc now carries verbatim reference rules alongside Melivoco's own design decisions, so it is longer and mixes the book's checklist with the project's choices — accepted in exchange for having the full architecture standard in one place.

## 2026-11-24 — Requirements split out; Code Complete prerequisites completed

- **What changed:** The 26 Code Complete prerequisites Seth dictated verbatim were finished and moved out of the architecture doc into their own requirements doc — 7 Specific Functional Requirements, 7 Specific Non-Functional Requirements (Quality), 9 Requirements Quality, and 3 Requirements Completeness. The architecture doc is architecture only again. The repo was also renamed by Seth himself to Melivoco (single l), matching the brand spelling.
- **Why:** The rules are the book's own, as written — like the periodic table — and they belong as requirements, not mixed into how the system is built. Keeping them separate means each doc has one job, and the architecture can be handed to a builder without the requirements getting lost in it.
- **What it replaced / what was considered:** The requirements had been added directly into the architecture doc as they were dictated. The alternative was leaving them there; splitting them keeps the architecture doc focused on the design.
- **Trade-off accepted:** Two docs to keep in sync instead of one — the requirements doc points to the architecture doc, and new Code Complete rules Seth adds next go into the architecture doc at his request.

## 2026-11-24 — Architecture locked in (documentation phase)

- **Shape chosen:** a thin coordinator over a central SQLite store of take stats, with small pure analysis services behind it (fatigue, trends, sessions). Centralise the data and the interface; decentralise the computing. It won by a marginal amount over a direct pipeline and an event bus — the differences were small, and all options were weighed with full pros and cons.
- **Silent Witness placement changed:** from last insert on the master bus to last insert on the vocal recording track itself. The master hears the summed, post-everything mix, not an isolated vocal — so the old placement was wrong for the job.
- **Stat extraction rule #1 locked in:** window the audio stream into slices, find amplitude per window, smooth harsh peaks by replacing an outlier with the local average of surrounding points, keep the averaged stat, drop the raw audio.
- **Take recognition locked in:** Silent Witness detects take start/stop (threshold-based) and uses that to organise takes.
- **DAW sync locked in:** sync to the DAW transport/buffer callbacks, see the same audio blocks the track is recording, and automatically grab take name and track markers — no artist action needed.
- **main() agreed:** a lean entry point only — parse startup arguments, create the coordinator, initialise the local stats store, start the loop listening for the plug-in. It does no analysis itself.
- **Principles reaffirmed:** offline-first, security-first, stats-only (drop the audio), analysis not correction, invisible to the artist while making music, near-zero latency.

## 2026-11-24 — PHILOSOPHY.md added

- The statement that Melivoco is built to be understood, to stay maintainable, and to be secure by design — built greenfield on principles learned from older monolithic systems mistakes, instead of repeating them. Design by Contract thinking from *The Pragmatic Programmer* is the clearest influence here: modules make promises to each other, and a failure is a broken promise, not a mystery.

## 2026-11-24 — Spelling corrected: Mellivoco -> Melivoco

- The app name is **Melivoco** (single l). The README was corrected; the repository name still reads Mellivoco (double l) and is due to be renamed to match. The domain mellivoco.com (double l, GoDaddy, registered 2026-03-27, expires 2027-03-27) does not match the brand spelling — the single-l melivoco.com was available as of 2026-11-24 and is the intended home once it can be bought.

## 2026-11-24 — README expanded

- README rewritten with the fuller overview, the problem, the three MVP features (Longitudinal Analysis Engine, Silent Witness DAW plug-in, Take Explorer), Fatigue Watch, and the offline-first / security-first principles.

## 2026-11-24 — Legal groundwork comes first

- Decision: settle the legal groundwork before building further — Amazon employment terms, IP and patenting the audio-processing method, voice-data privacy and security, and the finances of running it. Documentation continues meanwhile; code waits for counsel.

## 2026-11-02 — Repository created

- Initial commit: README only. The idea already existed in full — the repo started as the place to build it in public, properly documented.

## Before the repo — the idea

- Melivoco comes from almost two decades as a vocal artist: the voice is a muscle you cannot see, and there is no visual gauge for what it is doing. The spark was watching baseball pitchers study their release with high-speed cameras — singers deserve the same kind of record for their voice. Even takes that never make a record still count; a session where you worked your voice is still proof of the work.
- Business model undecided — sell it vs. open source, leaning open source supported via Patreon. Melivoco is a love letter to music, an ideal and a movement, not just a plug-in.

---

## How to add an entry

Add a dated section at the top, newest first, in this shape:

```
## YYYY-MM-DD — Short title

- What changed:
- Why:
- What it replaced / what was considered:
- Trade-off accepted:
```

If a decision is reversed later, do not delete the old entry — add a new one that says what changed and why. The history is the point.
