# Melivoco — Data Model

Status: documentation only. No code yet. Legal groundwork comes before building — get the Amazon employment agreement and Career Choice terms reviewed, and the IP / voice-data privacy questions settled, before writing code.

This is the data model that sits behind the architecture in `melivoco-architecture.md`. It defines what a take, a session, and a stress point actually store. It is built in C++ only, with a central SQLite store of take statistics, as decided in the architecture.

## Principles

- **Stats-only, by design:** extract statistics, then drop the raw audio. The raw audio is never persisted. No audio files, no waveforms stored as audio, no audio blobs in the database.
- **Offline-first:** the store lives locally. A cloud version may come later, and would need its own privacy review first.
- **Security-first:** voice data at scale carries identity risk. Derived stats can still identify a voice, so treat the store as sensitive even though it holds no audio.
- **Invisible while making music:** Silent Witness writes stats in the background. Nothing in this model requires the artist to type anything during a session.
- **Analysis, not correction:** the model stores what happened, not how to fix it. It is not Auto-Tune or Melodyne.

## The store

One local SQLite database, owned by the Longitudinal Analysis Engine, which acts as a thin coordinator over it. Small, pure analysis services (fatigue, trends, sessions) read from the store; they do not own their own copies of the data.

All timestamps are stored as UTC, with the session's local timezone recorded once on the session so times can be displayed back in local time.

All scores in this model are 0–100 unless stated otherwise. That is a convention for display and comparison, not a claim that every future stat will fit it.

IDs are local integers, assigned by the store, starting at 1. They are not shared, synced, or used as an external identifier.

## Entities

### Session

A session is one sitting in the studio — one DAW project open, one stretch of recording. A session groups takes that belong together in time.

| Field | Type | Notes |
|---|---|---|
| `session_id` | integer, primary key | Local ID. |
| `started_at_utc` | timestamp | When the session started, from the DAW transport. |
| `ended_at_utc` | timestamp, nullable | When the session ended. Null while the session is open. |
| `local_timezone` | text | e.g. `America/Los_Angeles`. Recorded once, for display. |
| `daw_project_name` | text, nullable | Project name from the DAW, if available. |
| `notes` | text, nullable | Free text added later in Take Explorer, after the session. Never required during a session. |

A session has many takes. A take belongs to exactly one session.

### Take

A take is one detected stretch of singing on the vocal recording track — from threshold start to silence close, as defined in the architecture doc.

| Field | Type | Notes |
|---|---|---|
| `take_id` | integer, primary key | Local ID. |
| `session_id` | integer, foreign key → Session | Which session this take belongs to. |
| `take_number` | integer | Order within the session, starting at 1, in session order. |
| `take_name` | text, nullable | Moved to the Label table (see below). A take itself carries no name; names are labels with a source. |
| `track_name` | text, nullable | Vocal track name from the DAW, if available. |
| `started_at_utc` | timestamp | Take start, from threshold detection. |
| `ended_at_utc` | timestamp | Take close, on silence. |
| `duration_seconds` | real | `ended_at_utc - started_at_utc`, in seconds. Stored, not just computed, so later queries do not need to re-derive it. |
| `markers` | — | Removed from Take. Markers, when a host exposes them at all, are labels (see the Label table below), not take fields. |

A take has many window stats, many stress points, and many tags / notes / segments added later. A take has no audio. There is no audio field, no file path to audio, and no waveform blob.

### Label

A label is any name or marker attached to a take or a segment, kept separate from the detected segment itself because labels have sources and can be absent, wrong, or added later.

| Field | Type | Notes |
|---|---|---|
| `label_id` | integer, primary key | Local ID. |
| `take_id` | integer, foreign key → Take | Which take this label belongs to. |
| `segment_id` | integer, nullable, foreign key → Segment | Set when the label belongs to one segment rather than the whole take. |
| `kind` | text | `take_name`, `marker`, or `artist_tag`. |
| `value` | text | The label text, e.g. `verse`. |
| `source` | text | Provenance: `host_api`, `host_scripting`, or `artist`. Never assumed. |
| `created_at_utc` | timestamp | When the label was recorded. |

A take or segment with no label row is normal — in most hosts no label will be available at capture time — not an error.

### Window stat

Window stats are the per-slice numbers produced by extraction rule 1 in the architecture doc: window the stream, find amplitude per window slice, average, smooth outliers by replacing them with the local average.

Window stats are the finest-grained numbers in the store. They are what the engine aggregates into take-level stats, and what Fatigue Watch plots.

| Field | Type | Notes |
|---|---|---|
| `window_id` | integer, primary key | Local ID. |
| `take_id` | integer, foreign key → Take | Which take this window belongs to. |
| `window_index` | integer | Order within the take, starting at 0. |
| `started_at_utc` | timestamp | Start of this window slice. |
| `amplitude_raw` | real | Amplitude in this window before smoothing. Stored so smoothing can be audited and re-run. |
| `amplitude_smoothed` | real | Amplitude after the local-average smoothing rule. This is the value analysis uses. |
| `was_smoothed` | boolean | True if this window was replaced by the local average — i.e. it was an outlier. False otherwise. |

Future extraction rules (pitch, consistency, and others) add their own columns or their own per-window tables, specified the same way as rule 1 — mechanically first, then in audio terms. They do not change this table's shape for amplitude.

### Take stat (aggregated)

Take stats are the per-take summaries the engine and Take Explorer actually show. They are computed from the window stats of that take, and stored so the UI does not have to recompute them every time.

| Field | Type | Notes |
|---|---|---|
| `take_id` | integer, foreign key → Take, primary key | One row per take. |
| `mean_amplitude` | real, nullable | Mean of `amplitude_smoothed` across the take's windows. |
| `peak_amplitude` | real, nullable | Highest `amplitude_smoothed` in the take. |
| `smoothed_window_count` | integer | How many windows in this take had `was_smoothed = true`. A simple, honest count of how much smoothing happened. |
| `performance_score` | real, nullable, 0–100 | A single 0–100 score per take, for Fatigue Watch and longitudinal charts. The exact definition of this score is not yet specified — it is a placeholder field, not a finished formula. It must be defined before any chart treats it as meaningful. |

Null means "not yet computed", not zero. A take with no computed stats yet has nulls here, not 0s.

### Stress point

A stress point is a moment in a take where the voice was under strain, as flagged by analysis or by the artist later. Stress points are metadata about a moment — a time and a label — not audio.

| Field | Type | Notes |
|---|---|---|
| `stress_point_id` | integer, primary key | Local ID. |
| `take_id` | integer, foreign key → Take | Which take this stress point belongs to. |
| `at_utc` | timestamp | When in the take this point happened. |
| `window_index` | integer, nullable | Which window slice it falls in, if known. |
| `kind` | text | What kind of stress it is, e.g. `peak`, `strain`, `drop-off`. The full list of kinds is not yet specified. |
| `note` | text, nullable | Short note, added by the artist later in Take Explorer. |
| `source` | text | `analysis` or `artist`. Who flagged it. |

### Tag

Tags are artist-added labels for sifting takes later in Take Explorer. They are added after a session, never during one.

| Field | Type | Notes |
|---|---|---|
| `tag_id` | integer, primary key | Local ID. |
| `take_id` | integer, foreign key → Take | Which take this tag belongs to. |
| `label` | text | The tag itself, e.g. `keeper`, `warm-up`, `chorus`. |
| `created_at_utc` | timestamp | When the tag was added. |

### Take note

Free-text notes on a take, added after a session in Take Explorer.

| Field | Type | Notes |
|---|---|---|
| `note_id` | integer, primary key | Local ID. |
| `take_id` | integer, foreign key → Take | Which take this note belongs to. |
| `body` | text | The note. |
| `created_at_utc` | timestamp | When the note was added. |
| `updated_at_utc` | timestamp, nullable | When the note was last edited, if ever. |

### Segment

A segment is a labelled slice of a take — for example a verse, a chorus, or a phrase the artist wants to find again. Segments are added after a session in Take Explorer. They reference time, not audio.

| Field | Type | Notes |
|---|---|---|
| `segment_id` | integer, primary key | Local ID. |
| `take_id` | integer, foreign key → Take | Which take this segment belongs to. |
| `label` | text | e.g. `verse`, `chorus`, `bridge`. |
| `started_at_utc` | timestamp | Start of the segment. |
| `ended_at_utc` | timestamp | End of the segment. |

## Relationships, in one view

- Session → many Takes
- Take → many Window stats, one Take stat row, many Stress points, many Tags, many Take notes, many Segments, many Labels
- Nothing in this model points at an audio file. There is no audio table, no audio path, and no audio blob anywhere in the store.

## What is kept, what is dropped

Kept, per take:

- Extracted statistics: window stats (raw and smoothed amplitude, with a flag for smoothed windows) and the aggregated take stat row
- Metadata: start/stop times, session and timezone; track name only where the host exposes it; labels (take names, markers) stored separately with their source, only when actually obtained
- Stress points: time, kind, source, and an optional note
- Tags, notes, and segments added later in Take Explorer

Dropped, always:

- The raw audio files, once statistics have been extracted.
- No waveform is stored as audio. If a waveform is ever shown in Take Explorer, it must be drawn from the stored window stats, not from audio.

## Fatigue Watch, in data terms

Fatigue Watch reads the take stat rows for one session, in take order:

- **Warm-up:** the early takes, where the performance score is still rising.
- **Peak:** the take or takes with the highest performance score in the session.
- **Fall-off:** the takes after the peak where the score has dropped — in the R prototype, the first take 5+ points below the peak.

The R prototype in the goal files (`fatigue-watch-r-prototype.R`) uses fake data to show this curve: 24 takes, peak at take 10 with a score of 88, fall-off at take 15. That prototype is a lab step, not the app, and its numbers are fake. The real version reads `performance_score` from this model, once that score is defined.

## Open questions

- **Performance score definition:** `performance_score` is a placeholder. Its formula — what goes into a 0–100 score for a take — is not yet specified.
- **Stress point kinds:** the full list of `kind` values is not yet specified.
- **Threshold tuning for take recognition:** the threshold, hold time, and silence duration that close a take are not yet specified. They will need testing against real sessions.
- **Smoothing parameters:** the window size and the outlier threshold (how far above the local average a point must be to be smoothed) are starting guesses, not tuned values.
- **Further extraction rules:** rule 1 covers amplitude. Pitch, consistency, and other stats are future rules, to be specified the same way — mechanically first, then in audio terms. Each will add fields here.
- **Retention and deletion:** how long stats are kept, and how a take or session is deleted on request, is not yet specified. Deletion must remove the take's window stats, take stat row, stress points, tags, notes, and segments together.

## What comes next in the documentation

- The remaining stat extraction rules, specified the same way as rule 1, with their fields added to this model.
- The definition of `performance_score`.
- The Take Explorer workflow: how takes are sifted, tagged, and annotated after a session.
- The Fatigue Watch workflow in detail.
- The privacy and security guarantees, in the same concrete terms as this document.
