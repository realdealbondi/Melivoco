# Melivoco — Evolution

This directory is the record of how Melivoco changes over time: what was decided, when, why, and what it replaced. The code shows what Melivoco *is*. This shows how it *got here*.

New entries go at the top, newest first. Each entry names the date, what changed, why, and what trade-off was accepted.

---

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
