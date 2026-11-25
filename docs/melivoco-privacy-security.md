# Melivoco — Privacy and Security Guarantees

Status: documentation only. No code yet. This is a design document, not legal advice. The legal questions it flags — especially voice data as biometric data, and Seth's Amazon employment terms — go to legal counsel before building, per the legal groundwork Seth asked for first.

This document states, in the same concrete terms as the architecture doc, what Melivoco promises about a vocalist's data, and what it does not promise yet.

## The one-sentence promise

Melivoco learns from your voice without keeping your voice. It extracts statistics from a take, keeps only those statistics plus the metadata that labels them, and drops the raw audio.

## Guarantees

### 1. Stats-only, by design

- Silent Witness extracts statistics from the audio stream (starting with the locked-in rule 1: window, amplitude per slice, smooth outliers with a local average) and passes only the resulting statistics onward.
- The raw audio is dropped after extraction. It is never written to disk by Melivoco, never uploaded, and never kept "just in case."
- What is kept per take: extracted statistics, metadata (start/stop times from take recognition, plus take name and markers only when the host actually exposes them), stress points, and — later, in Take Explorer — notes, tags, and segments the artist adds.
- Nothing else from the audio is kept. There is no hidden copy, no cache of the waveform, and no background recording.

### 2. Offline-first, local-only in the MVP

- Melivoco runs locally. The Longitudinal Analysis Engine reads from a central SQLite store of take statistics on the artist's own machine.
- In the MVP there is no cloud version, no account, and no upload. A cloud version is possible later; it is not part of these guarantees and would need its own privacy review before it exists.
- No take, statistic, or piece of metadata leaves the machine unless the artist deliberately exports or shares it.

### 3. Invisible capture, isolated vocal only

- Silent Witness lives as the last insert on the vocal recording track — the track the audio is being recorded out of — before it hits the master bus. It does not live on the master bus.
- That placement is a privacy decision as well as a sound one: on the master bus it would hear the full summed mix, not an isolated vocal. On the vocal recording track it hears only the vocal it is meant to analyse.
- The artist sees nothing to do with the app while performing. Capture happens entirely in the background.

### 4. DAW sync reads the minimum, touches nothing

Silent Witness syncs to the DAW's transport and buffer callbacks so it sees the same audio blocks the track is recording. What it reads, and what it does not do:

- It reads: the audio blocks on that vocal track and the transport state. Take names and track markers are read only where the host exposes a real path to them — most hosts do not — and their absence changes nothing about capture.
- It does not touch the recording itself. It does not modify, re-route, or re-record any audio.
- It does not read other tracks, the master bus, or any audio the artist is not recording through that vocal track.
- It does not send anything to the DAW, to a server, or to any third party. The only handoff is statistics and metadata to the local engine, through a lock-free ring buffer, on the same machine.
- Bit-exact passthrough: Silent Witness must not change the audio it hears. Zero reported latency, no allocation and no locks on the audio thread.

### 5. Your data stays yours, and deletable

- Takes, statistics, and metadata live in the local store on the artist's machine. There is no Melivoco account holding a copy.
- Deleting a take in Take Explorer deletes its statistics and metadata from the store. There is no server copy to chase and no backup that keeps it after deletion. (Exact deletion behaviour for any future export or backup the artist makes themselves is still to be specified — see open questions.)
- Notes, tags, and segments added in Take Explorer are the artist's own words, stored locally alongside the take, and deleted with it.

### 6. Voice data is treated as sensitive even when it is only statistics

- A voice carries identity risk at scale. That is why Melivoco is stats-only and offline-first in the first place.
- Keeping only statistics does not automatically settle the legal question. Voice can count as biometric data under laws such as CCPA, BIPA, and Texas law, and the obligations that follow — notice, consent, and deletion protections — still need a read from legal counsel, even with no raw audio retained. This document records the design intent; it does not claim the legal question is closed.
- If Melivoco is ever released open source (one of the paths Seth is weighing, alongside selling it), that changes the code, not the data: the source being public would not make any artist's takes or statistics public. Each artist's data would still stay on their own machine.

## What is NOT guaranteed yet

Honest limits, so nobody reads this as more than it is:

- **Encryption at rest:** not yet decided. The local SQLite store is local-only, but whether its contents are encrypted on disk is an open design decision, not a guarantee in this draft.
- **Threshold and smoothing parameters:** the take-detection threshold, hold time, silence duration, window size, and outlier threshold are starting guesses, not tuned values. They affect what counts as a take, not what leaves the machine — but they are not settled.
- **Future stats:** rule 1 covers amplitude. Pitch, consistency, and other statistics are future rules, to be specified the same way — mechanically first, then in audio terms — and each new rule needs the same privacy check: what is extracted, what is kept, what is dropped.
- **Any cloud version:** none exists in the MVP, so none of these guarantees covers one. A cloud version would need new guarantees of its own before it ships.

## Open questions for legal counsel

- Does stats-only retention change the biometric-data obligations, or do notice / consent / deletion duties apply in full anyway?
- What does deletion need to cover if an artist has exported statistics or shared them?
- Does label data supplied by an external metadata source (take names, track markers), where it exists, count as personal data in any jurisdiction where Melivoco might be used?
- Anything in Seth's Amazon employment agreement or Career Choice terms that touches work done on his own time, on his own gear, outside Amazon? (Already flagged in the legal checklist; still open until counsel reviews the actual documents.)

## How this document relates to the architecture doc

- The architecture doc (`melivoco-architecture.md`) says what Melivoco is and how it is built: thin coordinator over a central SQLite stats store, Silent Witness on the vocal recording track, DAW sync with transport data (labels only where a host actually exposes them), threshold take detection, and extraction rule 1.
- This document says what that architecture promises about the artist's data, and where the promise stops. If a future architecture change would break a guarantee here, the guarantee has to be revisited explicitly — the change does not silently override it.
