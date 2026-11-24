# MELIVOCO

Longitudinal vocal analysis software — a Strava for your voice.

Melivoco tracks a vocalist's takes over the span of their entire career, so singers can see how their voice changes, warms up, peaks, and tires over time. Vocals are complex and invisible: you cannot see your throat while you sing, and there is no comfortable visual gauge for what your voice is doing. Melivoco is built to make that invisible work visible.

The idea came from watching baseball pitchers practice with a high-speed camera on their release. Pitchers can study every point in a motion. Singers deserve the same kind of record for their voice.

Even takes that never make a record still count. A session where you worked your voice, even if nothing was release-ready, is still proof of the work — and still part of your journey as an artist.

## The problem

- Vocal progress is hard to track because the voice is a muscle you cannot see.
- Studio takes are scattered, lost, or only judged in the moment.
- There is no easy way to compare a take from today with one from a year ago, or to see when you were at your best in a session and when you started to fall off.
- Existing tools mostly focus on correcting a single take, not on understanding a voice over time.

## What Melivoco does

Melivoco is analysis-first, not correction-first. It takes in vocal takes, extracts useful statistics from them, and keeps those statistics so you can follow trends across sessions, months, and years.

## MVP features

### 1. Longitudinal Analysis Engine
The core of Melivoco. It takes in vocal takes and runs analysis on them over time, building a long-term picture of a vocalist's range, consistency, warm-up, peak, and fatigue.

### 2. Silent Witness plug-in
A plug-in that artists put in their DAW. It processes takes throughout a session and sends them into the standalone Melivoco app.

Once the statistics have been extracted from the audio, the audio files are dropped. Melivoco keeps the stats, metadata, and stress points — not the heavy audio — so storage stays light and the voice itself stays private.

### 3. Take Explorer
A way to sift through every take in the database. Trim parts, add notes to specific parts, segment takes, tag parts, and search through your vocal history. That makes analysis easier on both a granular and an emotional level.

### Fatigue Watch
A smaller feature inside the analysis view. It charts a session from beginning to end: where warm-up should end, where you hit your best performances, and where you started to fall off — so you can optimize studio time.

## Principles

- **Offline-first:** Run locally where possible, with a local model where possible. No connection required unless you choose it. A cloud version may come later.
- **Stats-only, by design:** Keep the analysis, drop the audio. The goal is insight, not a growing library of raw vocal recordings.
- **Security-first:** Voice data at scale carries real identity risk, so Melivoco is designed to protect it from the ground up.
- **Analysis, not correction:** Melivoco helps you understand your voice. It is not an auto-tune or pitch-correction tool.

## Status

Early development / proof of concept. The three MVP features above are the starting point.

## Building

Melivoco is being built in C++. Development tools include Visual Studio Community and CLion.

Documentation is being written alongside the code, following the approach in *Code Complete* and *The Pragmatic Programmer*, including test-driven development and design by contract.

## About

Melivoco is being built by a vocal artist with almost two decades making music, for vocalists who want a logbook for their voice — a record of the work, not just the final record.
