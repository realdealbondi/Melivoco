# Melivoco — Architecture

Status: documentation only. No code yet. Legal groundwork comes before building — get the Amazon employment agreement and Career Choice terms reviewed, and the IP / voice-data privacy questions settled, before writing code.

Melivoco is longitudinal vocal analysis software: a Strava for vocals, a logbook for vocal takes over a career. It is analysis, not correction. It is built in C++ only.

## Principles

- **Offline-first:** runs locally where possible, with a local model where possible. A cloud version may come later.
- **Stats-only, by design:** extract statistics, then drop the raw audio. Keep stats, metadata, and stress points only.
- **Security-first:** voice data at scale carries identity risk.
- **Invisible while making music:** the artist should not see anything to do with the app while performing.
- **Analysis, not correction:** Melivoco helps you understand your voice. It is not Auto-Tune or Melodyne.

## The three MVP features

1. **Longitudinal Analysis Engine** — the core, and the central figure in this architecture.
2. **Silent Witness** — the DAW plug-in that captures takes in a session and feeds the engine.
3. **Take Explorer** — sift, trim, annotate, segment, tag, and search every take.

Plus **Fatigue Watch**, a smaller feature inside the analysis view: chart warm-up end, peak performance, and fall-off across a session, so studio time can be optimised.

## Architecture verdict: thin coordinator over a central stats store

The Longitudinal Analysis Engine is the hub. It feeds the other parts and features.

The chosen shape, by a marginal amount — the trade-offs across the candidates are close, and no candidate was discarded just for having downsides:

- A **thin coordinator** over a **central store of take statistics** (SQLite), with **small, pure analysis services** behind it for fatigue, trends, and sessions.
- Centralise the data and the interface to analysis. Decentralise the actual computing, so the coordinator does not become a god object.

Close alternatives, kept in view:

- **Direct pipeline** — a close second. Simplest to ship first; harder to extend later when more consumers need the same stats.
- **Event bus / event-driven** — a close third. Best if many consumers are added later; more moving parts and harder to reason about ordering for a solo developer now.
- **Full CQRS, microkernel, and separate processes** — all viable, just more expensive for a solo developer right now. Not wrong, not chosen yet.

Invisibility and near-zero latency do not separate the candidates. They are decided by the plug-in's real-time core, which is the same in every option (see Silent Witness below). The choice therefore comes down to coherence and testability, where the differences are small.

## Silent Witness — placement

Silent Witness lives as the **last insert on the vocal recording track** — the track the audio is being recorded out of, routed to whatever vocal track — before it hits the master bus.

It does **not** live on the master bus. That was the earlier idea, and it was changed deliberately: the last insert on the master hears the full summed, post-everything mix, not an isolated vocal, so the stats would describe the whole mix, not the voice. Moving it to the vocal recording track keeps the capture invisible while giving clean, isolated vocal data.

### Real-time constraints

- Bit-exact passthrough: Silent Witness must not change the audio it hears.
- Zero reported latency.
- No allocation and no locks on the audio thread.
- Statistics are handed off to non-real-time analysis through a lock-free ring buffer.
- The artist sees nothing while performing. The plug-in works entirely in the background.

## Silent Witness — DAW sync and take recognition

Silent Witness syncs to the DAW's transport and buffer callbacks. It sees the same audio blocks the track is recording, but it only pulls amplitude and timing from them — it does not touch the recording itself.

### Take recognition

Accurate take recognition is required, so takes can be organised correctly:

- A take starts when the audio crosses a threshold.
- The take window is held while the signal stays above that threshold.
- The take closes on silence.
- The start time is saved with the take's stats.

### Segments, not labels

Silent Witness cannot assume labels. No plug-in format gives an insert a standard way to read take/clip names or timeline markers, so a take arrives labelled only when the host offers a host-specific path to that data. Silent Witness probes what the current host actually exposes at session start and degrades in tiers: transport data (playhead, tempo, sample rate) where available, audio-only detection where it is not, and labels only where a real path to them exists. Where no label is available, the take stays unlabelled until the artist names it later in Take Explorer — detection never depends on labels.

## Stat extraction — rule 1: window, amplitude, smooth

This is the first locked-in extraction rule.

1. **Window the stream:** chop the incoming audio stream into small window slices.
2. **Amplitude per window:** find the amplitude points in each window slice.
3. **Average:** find the averages across those points.
4. **Smooth outliers:** if a point spikes far above its neighbours — for example one tick 10 dB above the audio around it — replace it with the average of the numbers around it. Mechanically this is a moving-average / low-pass filter on the envelope. In audio terms it is like riding the fader so one loud spike does not jump out of the take.
5. **Keep the stat, drop the audio:** the averaged statistic is what is kept and passed on. The raw audio is dropped after extraction.

This rule keeps the data clean for the engine, protects privacy, and preserves the invisible workflow.

## What is kept, what is dropped

Kept, per take:

- Extracted statistics (starting with the smoothed amplitude stats above)
- Metadata: start/stop times from take recognition; take name and markers only when the host actually exposes them (nullable, with their source recorded)
- Stress points
- Notes, tags, and segments added later in Take Explorer

Dropped:

- The raw audio files, once statistics have been extracted.

## Data flow, end to end

1. The artist sings. Silent Witness sits, invisible, as the last insert on the vocal recording track.
2. Silent Witness syncs to the DAW transport, sees the same audio blocks being recorded, and detects take start/stop by threshold.
3. It windows the stream, extracts amplitude per slice, and smooths outlier peaks with a local average. If the host exposed a take name or markers through a real API path, they are attached as labels; otherwise the take is stored unlabelled.
4. Only stats, metadata, and stress points are handed off — via a lock-free ring buffer — to the Longitudinal Analysis Engine. The raw audio is dropped.
5. The engine, a thin coordinator over the central SQLite stats store, runs the small analysis services (fatigue, trends, sessions) over the stored stats.
6. Take Explorer reads from the engine/store so the artist can sift, trim, annotate, segment, tag, and search takes later — after the session, not during it.
7. Fatigue Watch charts that session from beginning to end: warm-up end, peak, fall-off.

## Code Complete — Architecture

### Program Organization

- The architecture should define the major building blocks in a program.
- In the architecture, you should find evidence that alternatives to the final organization were considered and find the reasons for choosing the final organization over its alternatives.
- Every feature listed in the requirements should be covered by at least one building block.
- If a function is claimed by two or more building blocks their claims should cooperate not conflict.
- What each building block is responsible for should be well defined.
- A building block should have one area of responsibility and it should know as little as possible about other building blocks areas of responsibility.
- The communication rules for each building block should be well defined.
- The architecture should describe which other building blocks the building block can use directly which it can use indirectly and which it shouldn't use at all.

### Major Classes

- The architecture should specify the major classes to be used.
- It should identify the responsibilities of each major class and how the class will interact with other classes.
- It should include descriptions of the class hierarchies of state transitions and of object persistence.
- If the system is large enough it should describe how classes are organized into subsystems.
- The architecture should describe other class designs that were considered and give reasons for preferring the organization that was chosen.
- The architecture doesn't need to specify every class in the system aim for the 80 20 rule. Specify the 20% of the classes that make up 80% of the system's behavior. (Jacobson, Booch and Rumbaugh 1999; Cruchten 2000)

### Data Design

- The architecture should describe the major files and table designs to be used.
- It should describe alternatives that were considered and justify the choices that were made.
- If the application maintains a list of customer IDs in the architects have chosen to represent the list of IDs using a sequential access list the document should explain why a sequential access list is better than a random access list stack or hash table.
- The architecture should specify the high level organization and contents of any databases used.
- The architecture should explain why a single database is preferable to multiple databases or vice versa explain why a database is preferable to flat files identify possible interactions with other programs that access the same data explain what views have been created on the data and so on.

### User Interface Design

- The user interface is often specified at requirements time if it isn't it should be specified in the software architecture.
- The architecture should specify major elements of web page formats GUIs command line interfaces and so on.
- The architecture should be modularized so that a new user interface can be substituted without affecting the business rules and output parts of the program.

### Resource Management

- The architecture should describe a plan for managing scarce resources such as database connections threads and handles.
- The architecture should estimate the resources used for nominal and extreme cases. In a simple case, the estimates should show that the resources needed are well within the capabilities of the intended implementation environment. In a more complex case, the application might be required to more actively manage its own resources. If it is the resource manager should be architected as carefully as any other part of the system.

### Security

- The architecture should describe the approach to design level and code level security.
- If a threat model has not previously been built it should be built at architecture time. Coding guidelines should be developed with security implications in mind including approaches to handling buffers. Rules for handling untrusted data encryption, level of detail contained in error messages, protecting secret data that's in memory and other issues.

### Performance

- Performance goals can include resource use in which case the goals should also specify priorities among resources including speed versus memory versus cost. If performance is a concern, performance goals should be specified in their requirements.
- The architecture should provide estimates and explain why the architects believe the goals are achievable. If certain areas are at risk of failing to meet their goals, the architecture should say so. If certain areas require the use of specific algorithms or data types to meet their performance goals the architecture should say that.
- The architecture can also include space and time budgets for each class or object.

### Scalability

- Scalability is the ability of a system to grow to meet future demands. The architecture should describe how the system will address growth in number of users, number of servers, number of network nodes, number of database records, size of database records, transaction volume and so on. If the system is not expected to grow in scalability is not an issue. The architecture should make that assumption explicit.

### Interoperability

- If the system is expected to share data or resources with other software or hardware the architecture should describe how that will be accomplished.

### Internationalization/Localization

- Internationalization is the technical activity of preparing a program to support multiple locales. Internationalization is often known as I18N because the first and last characters in internationalization are I and N and because there are 18 letters in the middle of the word. Localization known as L10N for the same reason is the activity of translating a program to support a specific local language.
- If the program is to be used commercially the architecture should show that the typical string and character set issues have been considered. Including character set used ASCII DBCS, EBCDIC, MBCS, Unicode, ISO 8859 and so on. Kinds of strings used like C strings, visual basic strings and so on. Maintaining the strings without changing the code and translating the strings into foreign languages with minimal impact on the code and the user interface.
- The architecture can decide to use strings in line in the code where they're needed keep the strings in a class and reference them through the class interface or store the strings in a resource file. The architecture should explain which option was chosen and why.

### Input/Output

- The architecture should specify a look ahead, look behind or just in time reading scheme. And it should describe the level at which IO errors are detected at the field, record, stream or file level.

## Open questions


- **Extraction on a vocal track vs. other tap points:** resolved for now by moving Silent Witness to the vocal recording track. Revisit if a DAW workflow routes vocals in a way that makes that placement impractical.
- **Threshold tuning for take recognition:** the threshold, hold time, and silence duration that close a take are not yet specified. They will need testing against real sessions.
- **Smoothing parameters:** the window size and the outlier threshold (how far above the local average a point must be to be smoothed) are starting guesses, not tuned values.
- **Further extraction rules:** rule 1 above covers amplitude. Pitch, consistency, and other stats are future rules, to be specified the same way — mechanically first, then in audio terms.

## What comes next in the documentation

- The remaining stat extraction rules, specified the same way as rule 1.
- What metadata and stress points get saved alongside each stat.
- The Take Explorer workflow: how takes are sifted, tagged, and annotated after a session.
- The Fatigue Watch workflow in detail.
- The privacy and security guarantees, in the same concrete terms as this document.
