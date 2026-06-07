# Single-Voice Pattern Sequencer Design

Issue: #5 — Single-voice pattern sequencer grid.

Builds on the scheduler refactor (2026-03-08-scheduler-refactor-design.md), which already added `S.mode`, `S.stepCount`, a unified `S.pending`, and a step-normalized clock with mode-aware `totalSteps()` / `stepDuration()` / `scheduleNote()` stubs.

## Decisions

- **Mode switch UX.** Sequencer is a distinct mode, not an overlay. A segmented control at the top of the container swaps between Metronome and Sequencer views. Each mode owns its own panel tree; only one is visible at a time.
- **Fixed 16 steps.** No 8/16 toggle in v1. If a user wants 8 steps, they leave the second half off.
- **Layout: single row of 16 on desktop, 4×4 grid on mobile.** Subtle gaps every 4 cells on desktop to group beat boundaries; on mobile each row of 4 is its own beat.
- **All steps play the same sound.** No downbeat emphasis — this is a single-voice sequencer. Emphasis would be a second voice and is deferred to multi-voice work.
- **Default pattern: every 4th step on** (indices 0, 4, 8, 12). Mirrors the default 4/4 quarter-note metronome on first entry — easiest mental bridge from metronome to sequencer.
- **Cell visuals: outline / dot-inside / lit / lit-with-dot.** Toggled state is a dot inside the cell; currently-playing state is a background fill (lit). Both states combine independently.
- **Step toggles are immediate, not deferred.** Unlike tempo/time-sig changes, toggling a step takes effect on the next time the playhead reaches it within the same loop. Step sequencers are interactive instruments; deferring would feel sluggish. This diverges from the scheduler refactor's blanket "all changes deferred" — that statement was about params, not pattern editing.
- **Unified profile list with M/S badges.** One profile list; each profile remembers its mode. Small `M`/`S` indicator distinguishes types in the list view.
- **No keyboard shortcut for mode switch in v1.** Avoid shortcut clutter.

## State

Add to `S`:

```js
pattern: [true, false, false, false, true, false, false, false,
          true, false, false, false, true, false, false, false]
```

A flat array of 16 booleans. Index is step number. Persists across mode switches.

Already present from the refactor: `S.mode` (`'metronome'` or `'sequencer'`), `S.stepCount` (16), `S.pending`.

## Mode toggle

Markup at the top of `.container`:

```html
<div class="mode-switch">
  <button class="mode-btn on" data-mode="metronome">Metronome</button>
  <button class="mode-btn"    data-mode="sequencer">Sequencer</button>
</div>
```

Click handler:
1. Update `S.mode`.
2. Toggle `.on` class on the two buttons.
3. Set `data-mode` attribute on `.container`.
4. If playing: call `applyPending()`, then reset `schedStep = 0` and `nextTime = ctx.currentTime + 0.05` (same as `start()`). This avoids trying to map step indices between two clocks with different `totalSteps()`.
5. Clear any lit cell from the previous mode.

CSS rules hide/show the mode-specific subtrees:

```css
.container[data-mode="metronome"] .seq-grid,
.container[data-mode="metronome"] .seq-controls { display: none; }

.container[data-mode="sequencer"] .panels,
.container[data-mode="sequencer"] .beat-dots { display: none; }
```

Visual style: pill-shaped container, two buttons inside. Active button gets the earth-tone fill, matching `.chip.on`.

## Sequencer grid UI

Markup rendered once at startup:

```html
<div class="seq-grid" id="seqGrid">
  <button class="seq-cell" data-step="0"></button>
  ...
  <button class="seq-cell" data-step="15"></button>
</div>
```

Cell visual states (combinable):
- Default: outlined cell, empty.
- `.on` — has a dot inside (toggled active). Dot is a `::after` pseudo-element.
- `.lit` — cell background fills (currently playing).
- `.on.lit` — lit cell with dot inside (dot color shifts for contrast).

Layout:
- Desktop: single flex row, 16 cells, small gap every 4 cells for beat grouping.
- Mobile (`@media (max-width: 600px)`): `display: grid; grid-template-columns: repeat(4, 1fr);` — 4 rows of 4.

Interaction:
- Click a cell: toggle `S.pattern[step]`, toggle `.on` class on the cell. Immediate (no measure-boundary deferral).
- Toggling a step also calls `clearActiveProfile()` — the loaded profile is no longer current.

## Audio: pattern lookup

`scheduleNote(step, t)` — replace the sequencer placeholder:

```js
if (S.mode === 'sequencer') {
  if (!S.pattern[step]) return;
  playThud(t, 'beat');
  const delay = Math.max(0, (t - ctx.currentTime) * 1000);
  setTimeout(() => triggerSeqVisual(step), delay);
  return;
}
```

Active steps play the `'beat'` thud. Inactive steps schedule nothing.

Background pulse-layer animations and downbeat flash do NOT fire in sequencer mode. The grid cell highlight is the only visual feedback. Background visuals for sequencer mode are stubbed as issue #8.

## Playhead visual

New function:

```js
let litCellIdx = -1;
function triggerSeqVisual(step) {
  if (litCellIdx !== -1) elSeqGrid.children[litCellIdx].classList.remove('lit');
  elSeqGrid.children[step].classList.add('lit');
  litCellIdx = step;
}
```

`stop()` clears the lit cell. Mode switch also clears it.

DOM cache adds `elSeqGrid = document.getElementById('seqGrid')`.

## Profile integration

Profile shape gains three optional fields:

```js
{ id, name, tempo, tsTop, tsBot, rhythm, swing,    // existing
  mode, stepCount, pattern }                        // new
```

Existing profiles without these fields are implicitly metronome profiles. No migration required.

**Save** (`confirmProfileSave`): always writes `S.mode`, `S.stepCount`, and `S.pattern.slice()` alongside existing fields.

**Update** (per-profile ↻ button): same — overwrites all fields.

**Load** (`applyProfile`): if `profile.mode === 'sequencer'`, set sequencer mode, apply `pattern` to `S.pattern`, re-render cell `.on` classes. Otherwise (missing or `'metronome'`), follow existing metronome-load path.

**Default name** (`defaultProfileName`):
- Metronome: `120 · 4/4 · ♩` (unchanged).
- Sequencer: `120 · seq`.

**List display:** small badge before the profile name — `M` or `S`, in `text-dim` color.

## What stays the same

- `playThud`, `firePulse`, `buildDots`, `litDots` — unchanged.
- The setTimeout/lookahead scheduler architecture.
- All existing keyboard shortcuts and event handlers.
- `commitParam`, `applyPending` — unchanged. Params still defer to measure boundary in both modes; only step toggles are immediate.
- Single-file architecture. No new files in `index.html`'s vicinity.

## Out of scope (filed as follow-ups)

- **Issue #7** — prompt to update loaded profile after divergence.
- **Issue #8** — background pulse/flash visuals for sequencer mode.
