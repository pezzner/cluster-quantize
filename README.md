[README.md](https://github.com/user-attachments/files/31969985/README.md)
# Cluster Quantize

An Ableton Live extension that quantizes chords as chords: it groups a MIDI
clip's notes into "clusters" by whether they actually **overlap in time**
(their `[start, start + duration]` spans intersect, transitively - not just
how close their start times happen to be), then snaps each cluster's overall
position to a grid — while every note inside the cluster shifts by the exact
same amount. A rolled chord, a deliberately spread pluck, or just normal
human timing between the notes of a chord survives byte-for-byte; only where
the chord *sits* on the timeline moves. That's the difference from Live's
own note quantize, which snaps every note independently and can turn a
chord into a smear if its notes weren't perfectly simultaneous to begin
with.

Clustering by overlap (rather than a start-time tolerance) matters because a
real played chord can easily have notes 20-30ms apart at the start while
still clearly being "one chord" - they're all held down over the same span.
A tolerance tight enough to avoid falsely merging unrelated notes is often
too tight to catch that, which would silently split a real chord into two
clusters that then snap to two different grid lines - changing how the
chord was actually played. That was the original bug in this extension's
first version; overlap detection is the fix.

Right-click a MIDI clip and you get two entries:

| Menu item | What it does |
| --- | --- |
| **Cluster Quantize...** | Opens a dialog: grid, strength, cluster anchor, cluster tolerance, and a before/after preview. One undo step. |
| **Cluster Quantize: Inspect Clip API** | Diagnostic. Prints the clip's real member names to the log — see [Note API](#read-this-first-the-note-api) below. |

## Scope: whole-clip only

Like [MIDI Align and Distribute](../MIDI%20Align%20and%20Distribute), this
extension operates on **every note in a clip**, not a piano-roll selection,
and on **one clip at a time**. The SDK doesn't expose which notes are
selected in the piano roll at runtime — see the Note API section below.

A checkbox to also group the resulting notes together the way Live's own
**Ctrl+G** does in the piano roll was considered and deliberately left out:
the SDK's `NoteDescription` type has no group-related field at all (its full
list is `duration`, `muted`, `pitch`, `probability`, `releaseVelocity`,
`selected`, `startTime`, `velocity`, `velocityDeviation`), and `MidiClip` has
no method for it either. There's currently no way for an extension to create
or read a piano-roll note group. If a future SDK beta adds one, that's a
clean addition to `clip-notes.ts` and the dialog.

## Read this first: the note API

Confirmed against `1.0.0-beta.0` (from the SDK's bundled TypeDoc reference,
matching what was already confirmed while building Distribute Notes and
MIDI Align and Distribute):

- **Reading**: `clip.notes` — a getter returning `NoteDescription[]`.
- **Writing**: `clip.notes = notes` — a setter taking `NoteDescription[]`.
  This **replaces the entire note list**. There's no partial/patch form, so
  writing back anything less than every note in the clip would silently
  delete the rest.
- **Timing fields**: `startTime` and `duration`.
- **Pitch field**: `pitch` (needed to read/normalise notes even though this
  extension never changes it).
- **Selection & grouping**: not exposed at runtime — see above.

`src/clip-notes.ts` is written against these confirmed names, tried first in
each candidate list. If a lookup ever misses (an SDK rename before 1.0
ships), you get an explicit error naming what it tried — run **Cluster
Quantize: Inspect Clip API** on any MIDI clip, read the member names it
logs, put the correct one at the front of the matching list, and the rest of
the extension works unchanged.

Because the writer replaces the whole note list, `writeClipNotes` always
rebuilds a full-length payload, patching in only the `start` field for the
notes that actually changed and leaving everything else - duration, pitch,
velocity, anything this extension doesn't touch - untouched.

## Install

1. Create a project with the SDK's project creator (see the Quick Start
   guide), or reuse this folder directly — it's a complete, runnable
   scaffold.
2. If scaffolding fresh: copy `src/` over the generated `src/`, plus this
   `README.md`; in the generated `build.ts` add the HTML loader
   (`loader: { ".html": "text" }`) so the dialog can be inlined; set
   `"name": "cluster-quantize"` in the generated `manifest.json`.
3. Create a `.env` file next to `package.json` with:

   ```
   EXTENSION_HOST_PATH=C:\ProgramData\Ableton\Live 12 Beta\Program\ExtensionHost\ExtensionHostNodeModule.node
   ```

   (adjust the path for your Live install; on Windows, `.env` has no
   basename so File Explorer can't create it directly — use
   `Set-Content -Path .env -Value '...' -Encoding ascii` from PowerShell.)
4. **Avoid `&` or other shell-special characters in the project's folder
   name.** npm runs scripts through `cmd.exe` on Windows, which treats a
   bare `&` as a command separator - a folder named with one in it breaks
   `npm start` in confusing ways that look like a build failure but aren't.
   `Cluster Quantize` is safe as-is (letters, spaces, no punctuation).
5. Run `npm install`, then enable **Settings → Extensions → Developer Mode**
   in the Live beta and run `npm start`. Only run one extension's `npm
   start` at a time unless you've specifically confirmed your Live beta
   build supports more than one simultaneous Developer Mode connection - see
   the project log for details.
6. If a freshly-connected extension logs its "ready" message but its menu
   items still don't appear: close every `npm start` terminal, fully quit
   and relaunch Live, re-confirm Developer Mode is on, then start just this
   one extension again. This cleared up an identical symptom while building
   MIDI Align and Distribute.

## Controls

**Grid** — the quantize grid, from 1/2 down to 1/32, including triplets
(1/4T, 1/8T, 1/16T, 1/32T). Values are independent of the clip's time
signature (they're all fractions of a beat), so there's no "1 bar" option.

**Strength** — 0% to 100%, same idea as Live's own built-in note quantize
"Amount". 100% snaps every cluster fully onto the grid; lower values pull
each cluster only partway there, keeping some of the original feel.

**Cluster anchor** — which point in a cluster decides where the whole
cluster snaps to:

- **Earliest note** — the first note to start is treated as "when the chord
  happens." Simple and predictable; matches how most people count a chord's
  timing.
- **Weighted center** — the average (centroid) of all the cluster's note
  start times. Useful for chords that are deliberately spread across a
  short span (e.g. a rolled or strummed chord) rather than just humanized,
  since it snaps based on the chord's overall position rather than
  whichever note happened to land first.

**Cluster tolerance** — notes that actually overlap in time are *always* one
cluster, regardless of this setting. This control only adds extra slack
*after* a note ends, for short, non-overlapping notes played close together
(e.g. a fast staccato chord where the notes don't sustain long enough to
literally overlap). Four presets: **Strict** (0 - must genuinely overlap,
the default), **Small gap allowed** (1/64 beat), **Medium gap allowed**
(1/32 beat), and **Large gap allowed** (1/16 beat).

All settings are a no-op (and say so, without touching the clip) rather than
an error when there's nothing meaningful to change - an empty clip, 0%
strength, or a clip that's already entirely on the grid.

## Project layout

```
src/
├── cluster-quantize.ts       Pure maths. No SDK imports, fully unit tested.
├── cluster-quantize.test.ts   node --test suite for the above.
├── clip-notes.ts              The only file that touches the SDK's clip API.
├── clip-notes.test.ts         node --test suite for the above, against a fake clip.
├── dialog.html                The modal webview: controls plus the preview.
├── extension.ts               Command registration and wiring.
└── html.d.ts                  Type declaration for the inlined HTML import.
```

Run the tests with `npm test` (Node 22+, no install required — it uses
native type stripping; it globs `src/*.test.ts`, so both suites run) and
typecheck with `npm run typecheck`.

## Notes on behaviour

- All edits run inside `context.withinTransaction`, so a cluster-quantize
  pass is a single undo step regardless of how many notes moved.
- The dialog's preview is a small mirror of `cluster-quantize.ts`, kept
  deliberately simple so it doesn't need the SDK. `cluster-quantize.test.ts`
  is the source of truth for the actual maths applied on Apply — if you
  change one, change both.
- The chord-grouping algorithm (chained from the note that opens each group,
  not a sliding window) is the same approach already proven in Distribute
  Notes and Align & Distribute, so a smoothly humanized run of single notes
  won't collapse into one giant cluster even at a loose tolerance setting.

## Ideas if you want to take it further

- A "tighten chords" toggle that collapses a cluster's internal spread to
  zero (distinct from the Ctrl+G note-grouping idea that got scoped out -
  this one only changes timing, no piano-roll grouping metadata involved,
  so it's fully buildable with today's SDK).
- Read the clip's own time signature (if a future SDK exposes it) to offer
  a "1 bar" grid option.
- Register the quick action on `ClipSlotSelection`/`ArrangementSelection` to
  cluster-quantize several clips at once.
