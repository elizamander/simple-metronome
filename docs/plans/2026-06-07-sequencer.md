# Single-Voice Pattern Sequencer Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement issue #5 — a 16-step single-voice pattern sequencer mode alongside the existing metronome.

**Architecture:** Two sibling DOM trees in one HTML file, toggled via `data-mode` attribute on `.container`. The scheduler refactor (already merged) handles step-normalized timing for both modes. Pattern is a flat 16-boolean array in `S.pattern`. Step toggles take effect immediately during playback; all other params remain deferred to measure boundary.

**Tech Stack:** Vanilla HTML/CSS/JS in a single `index.html` file. No build step, no test runner. Web Audio API for sound. `localStorage` for profiles.

**Reference:** Design doc at `docs/plans/2026-06-07-sequencer-design.md`.

**Testing approach:** No automated test infrastructure exists in this codebase. Each task has a browser smoke check — load `index.html` in a browser and verify the described behavior by hand. After every task, commit before moving on.

---

## Task 1: Add `S.pattern` to state

**Files:**
- Modify: `index.html` (the `S` object, around line 674–687)

**Step 1: Add the field**

In the `S` object, after `pending: null,` add:

```js
    pattern: [true, false, false, false, true, false, false, false,
              true, false, false, false, true, false, false, false],
```

**Step 2: Smoke check**

Open `index.html` in a browser. Open devtools console. Type `S.pattern.length` — expect `16`. Type `S.pattern` — expect an array with `true` at indices 0, 4, 8, 12 and `false` elsewhere.

**Step 3: Commit**

```bash
git add index.html
git commit -m "Add S.pattern to state for sequencer mode (refs #5)"
```

---

## Task 2: Add mode-switch markup and styling (visual only, no behavior)

**Files:**
- Modify: `index.html` — add CSS in the `<style>` block, add markup at the top of `.container`.

**Step 1: Add the markup**

Inside `<div class="container">`, BEFORE the existing `<div class="profiles-wrap" ...>`, add:

```html
    <div class="mode-switch" id="modeSwitch">
      <button class="mode-btn on" data-mode="metronome">Metronome</button>
      <button class="mode-btn"    data-mode="sequencer">Sequencer</button>
    </div>
```

**Step 2: Add CSS for the segmented control**

In `<style>`, near the other component blocks (after `.profiles-btn` rules, before `</style>`), add:

```css
    .mode-switch {
      display: flex;
      gap: 2px;
      padding: 3px;
      border-radius: 999px;
      background: var(--panel-bg);
      border: 1px solid var(--panel-bdr);
      margin-bottom: 1.5rem;
    }
    .mode-btn {
      flex: 1;
      padding: 0.4rem 1.1rem;
      border: none;
      background: transparent;
      color: var(--text-dim);
      font: inherit;
      font-size: 0.85rem;
      border-radius: 999px;
      cursor: pointer;
      transition: background 0.15s, color 0.15s;
    }
    .mode-btn:hover { color: var(--text); }
    .mode-btn.on {
      background: var(--earth);
      color: #fff;
    }
```

**Step 3: Smoke check**

Reload the page. The segmented control appears at the top with "Metronome" filled in earth tone and "Sequencer" dim. Click "Sequencer" — nothing happens yet (no JS wired). That's expected.

**Step 4: Commit**

```bash
git add index.html
git commit -m "Add mode-switch segmented control markup and styles (refs #5)"
```

---

## Task 3: Wire mode-switch click handler (no functional swap yet)

**Files:**
- Modify: `index.html` — DOM cache, add `commitMode` function, add event listener.

**Step 1: Add DOM cache entry**

In the DOM CACHE section (around line 715), add:

```js
  const elModeSwitch = document.getElementById('modeSwitch');
  const elContainer  = document.querySelector('.container');
```

**Step 2: Add the mode-commit function**

After the existing `commitParam` function (around line 829), add:

```js
  function commitMode(mode) {
    if (mode === S.mode) return;
    clearActiveProfile();
    if (S.playing) {
      applyPending();
    }
    S.mode = mode;
    elContainer.setAttribute('data-mode', mode);
    elModeSwitch.querySelectorAll('.mode-btn').forEach(b => {
      b.classList.toggle('on', b.dataset.mode === mode);
    });
    if (S.playing) {
      schedStep = 0;
      nextTime  = ctx.currentTime + 0.05;
    }
  }
```

**Step 3: Add event listener**

Near the other event wirings (e.g., after `elProfilesBtn.addEventListener` around line 1296), add:

```js
  elModeSwitch.addEventListener('click', e => {
    const b = e.target.closest('.mode-btn');
    if (b) commitMode(b.dataset.mode);
  });
```

**Step 4: Set initial attribute**

In the INIT section (around line 1337), before `buildDots();`, add:

```js
  elContainer.setAttribute('data-mode', S.mode);
```

**Step 5: Smoke check**

Reload. Verify `data-mode="metronome"` is set on `.container` (use devtools). Click "Sequencer" — the active pill should switch to Sequencer and `data-mode` becomes `"sequencer"`. Click "Metronome" — switches back. Start the metronome, click "Sequencer" while playing — verify `S.pending` is null in console and `schedStep` resets. No visible UI change for the panels yet (CSS not added).

**Step 6: Commit**

```bash
git add index.html
git commit -m "Wire mode-switch click handler (refs #5)"
```

---

## Task 4: Hide/show mode-specific panels via CSS

**Files:**
- Modify: `index.html` — CSS additions in `<style>`.

**Step 1: Add the rules**

After the `.mode-btn.on` rule from Task 2, add:

```css
    .container[data-mode="metronome"] .seq-grid { display: none; }
    .container[data-mode="sequencer"] .panels,
    .container[data-mode="sequencer"] .beat-dots { display: none; }
```

**Step 2: Smoke check**

Reload. In metronome mode, the existing panels (Time, Rhythm) and beat-dots are visible. Click "Sequencer" — panels and beat-dots disappear, leaving only the BPM display, tempo slider, and play button. (The `.seq-grid` doesn't exist yet, so nothing replaces them — that's the next task.) Click "Metronome" — panels return.

**Step 3: Commit**

```bash
git add index.html
git commit -m "Hide mode-specific panels via data-mode attribute (refs #5)"
```

---

## Task 5: Add sequencer grid markup and base CSS

**Files:**
- Modify: `index.html` — add markup after `.beat-dots`, add CSS.

**Step 1: Add the grid markup**

In the HTML body, find the `<div class="beat-dots" id="beatDots"></div>` line (around line 586). On the line after it, add:

```html
    <div class="seq-grid" id="seqGrid"></div>
```

We render cells from JS (Task 6) rather than hardcoding 16 buttons in HTML.

**Step 2: Add CSS**

In `<style>`, after the existing `.dot` rules (around line 318), add:

```css
    .seq-grid {
      display: flex;
      gap: 4px;
      margin: 1.4rem 0 1.8rem;
      width: 100%;
      max-width: 560px;
      justify-content: center;
    }
    .seq-cell {
      flex: 1;
      aspect-ratio: 1 / 1;
      max-width: 30px;
      min-width: 0;
      border: 1.5px solid var(--panel-bdr);
      border-radius: 6px;
      background: var(--panel-bg);
      cursor: pointer;
      padding: 0;
      position: relative;
      transition: background 0.08s, border-color 0.08s;
    }
    .seq-cell:hover { border-color: var(--text-dim); }
    .seq-cell.on::after {
      content: '';
      position: absolute;
      top: 50%; left: 50%;
      width: 38%; height: 38%;
      transform: translate(-50%, -50%);
      border-radius: 50%;
      background: var(--sage);
    }
    .seq-cell.lit {
      background: var(--earth);
      border-color: var(--earth);
    }
    .seq-cell.on.lit::after {
      background: var(--cream);
    }
    /* Subtle gap every 4 cells to mark beat boundaries (desktop) */
    .seq-cell:nth-child(4n):not(:last-child) {
      margin-right: 8px;
    }
```

**Step 3: Smoke check**

Reload. Switch to Sequencer mode. The `.seq-grid` div exists but is empty — no cells visible yet (rendering is the next task). Switch back to Metronome — panels return. Verify in devtools that the div is there.

**Step 4: Commit**

```bash
git add index.html
git commit -m "Add sequencer grid container and cell styles (refs #5)"
```

---

## Task 6: Render the 16 cells and sync them to `S.pattern`

**Files:**
- Modify: `index.html` — DOM cache, add `buildSeqGrid()` and `litSeqCells()`, call at init.

**Step 1: Add DOM cache entry**

In the DOM CACHE section, add:

```js
  const elSeqGrid = document.getElementById('seqGrid');
```

**Step 2: Add the build/sync functions**

After the `litDots()` function (around line 948), add:

```js
  function buildSeqGrid() {
    if (elSeqGrid.children.length !== S.stepCount) {
      elSeqGrid.innerHTML = '';
      for (let i = 0; i < S.stepCount; i++) {
        const c = document.createElement('button');
        c.className = 'seq-cell';
        c.dataset.step = i;
        elSeqGrid.appendChild(c);
      }
    }
    litSeqCells();
  }

  function litSeqCells() {
    for (let i = 0; i < elSeqGrid.children.length; i++) {
      elSeqGrid.children[i].classList.toggle('on', !!S.pattern[i]);
    }
  }
```

**Step 3: Call at init**

In the INIT section (around line 1337), after `buildDots();`, add:

```js
  buildSeqGrid();
```

**Step 4: Smoke check**

Reload. Switch to Sequencer mode. 16 cells should be visible in a single row. Steps 0, 4, 8, 12 have dots (from the default pattern). Beat boundaries (after cells 4, 8, 12) have a small extra gap. Click around — cells highlight on hover but clicking doesn't toggle yet (no handler).

**Step 5: Commit**

```bash
git add index.html
git commit -m "Render sequencer grid cells with toggled-state dots (refs #5)"
```

---

## Task 7: Wire cell-click to toggle steps immediately

**Files:**
- Modify: `index.html` — add event listener.

**Step 1: Add event listener**

Near the `elModeSwitch.addEventListener` from Task 3, add:

```js
  elSeqGrid.addEventListener('click', e => {
    const c = e.target.closest('.seq-cell');
    if (!c) return;
    const step = parseInt(c.dataset.step);
    S.pattern[step] = !S.pattern[step];
    c.classList.toggle('on', S.pattern[step]);
    clearActiveProfile();
  });
```

**Step 2: Smoke check**

Reload. Switch to Sequencer mode. Click cell 1 — dot appears. Click again — dot disappears. Click cell 0 — its dot disappears. Confirm `S.pattern[1]` toggled in console. Load a metronome profile, switch to sequencer, toggle a cell — the active-profile-name should clear.

**Step 3: Commit**

```bash
git add index.html
git commit -m "Toggle sequencer steps on cell click (refs #5)"
```

---

## Task 8: Implement audio playback in sequencer mode

**Files:**
- Modify: `index.html` — replace the sequencer placeholder in `scheduleNote` (around line 803–807).

**Step 1: Replace the placeholder**

Find:

```js
  function scheduleNote(step, t) {
    if (S.mode === 'sequencer') {
      // placeholder for pattern grid lookup
      return;
    }
```

Replace those lines with:

```js
  function scheduleNote(step, t) {
    if (S.mode === 'sequencer') {
      if (!S.pattern[step]) return;
      playThud(t, 'beat');
      const delay = Math.max(0, (t - ctx.currentTime) * 1000);
      setTimeout(() => triggerSeqVisual(step), delay);
      return;
    }
```

`triggerSeqVisual` doesn't exist yet — we'll add it in the next task. For now the call will silently no-op (and throw a ReferenceError when actually invoked). To avoid the error during this task's smoke check, comment out the `setTimeout` line temporarily, OR move directly to Task 9 and combine the verification. **Recommended: combine smoke check with Task 9** — don't reload between tasks.

**Step 2: Commit (no smoke check yet)**

```bash
git add index.html
git commit -m "Add pattern lookup to sequencer scheduleNote branch (refs #5)"
```

---

## Task 9: Add the playhead visual function

**Files:**
- Modify: `index.html` — add `triggerSeqVisual()`, add `litSeqCellIdx` tracker, clear on stop, clear on mode switch.

**Step 1: Add the tracker and function**

After the `triggerVisual` function (around line 928), add:

```js
  let litSeqCellIdx = -1;
  function triggerSeqVisual(step) {
    if (litSeqCellIdx !== -1 && elSeqGrid.children[litSeqCellIdx]) {
      elSeqGrid.children[litSeqCellIdx].classList.remove('lit');
    }
    elSeqGrid.children[step].classList.add('lit');
    litSeqCellIdx = step;
  }

  function clearSeqLit() {
    if (litSeqCellIdx !== -1 && elSeqGrid.children[litSeqCellIdx]) {
      elSeqGrid.children[litSeqCellIdx].classList.remove('lit');
    }
    litSeqCellIdx = -1;
  }
```

**Step 2: Call `clearSeqLit()` on stop**

In the `stop()` function (around line 887), after `buildDots();`, add:

```js
    clearSeqLit();
```

**Step 3: Call `clearSeqLit()` on mode switch**

In `commitMode` (added in Task 3), after the line `elContainer.setAttribute('data-mode', mode);`, add:

```js
    clearSeqLit();
```

**Step 4: Smoke check**

Reload. Switch to Sequencer mode. Press Space to start. You should hear the default pattern (every 4th step), at the current BPM (120). Each cell lights up briefly as the playhead passes through. Cells 0, 4, 8, 12 have a dot AND light up (combined state); intermediate cells light up but have no dot. Press Space to stop — no cells should remain lit. Click a few cells to toggle them while stopped, press Space — the new pattern plays immediately. Press Space to stop. Press Space again to start. Click a cell while playing — the change takes effect within the same loop (next time the playhead reaches it). Switch to Metronome while playing — the metronome takes over, no lit sequencer cells remain.

**Step 5: Commit**

```bash
git add index.html
git commit -m "Add sequencer playhead visual and clear on stop/mode-switch (refs #5)"
```

---

## Task 10: Add mobile 4×4 grid layout

**Files:**
- Modify: `index.html` — add media query in `<style>`.

**Step 1: Add the rule**

After the `.seq-cell:nth-child(4n)` rule from Task 5, add:

```css
    @media (max-width: 600px) {
      .seq-grid {
        display: grid;
        grid-template-columns: repeat(4, 1fr);
        gap: 6px;
        max-width: 280px;
      }
      .seq-cell {
        max-width: none;
      }
      .seq-cell:nth-child(4n):not(:last-child) {
        margin-right: 0;
      }
    }
```

**Step 2: Smoke check**

Reload. Open devtools, switch to mobile viewport (e.g., iPhone SE, 375px width). Switch to Sequencer mode. The grid should now be 4 rows of 4 cells. Each row corresponds to one beat. Click cells — toggles work. Start playback — playhead progresses across row 1, then row 2, then 3, then 4. Switch back to desktop viewport — grid returns to single row.

**Step 3: Commit**

```bash
git add index.html
git commit -m "Add 4x4 mobile grid layout for sequencer (refs #5)"
```

---

## Task 11: Save sequencer state with profiles

**Files:**
- Modify: `index.html` — update `confirmProfileSave` (around line 1257) and the profile update handler (around line 1197–1208).

**Step 1: Update `confirmProfileSave`**

Find:

```js
    const newProfile = { id: Date.now(), name, tempo: S.tempo, tsTop: S.tsTop,
                         tsBot: S.tsBot, rhythm: S.rhythm, swing: S.swing };
```

Replace with:

```js
    const newProfile = { id: Date.now(), name, tempo: S.tempo, tsTop: S.tsTop,
                         tsBot: S.tsBot, rhythm: S.rhythm, swing: S.swing,
                         mode: S.mode, stepCount: S.stepCount,
                         pattern: S.pattern.slice() };
```

**Step 2: Update the per-profile update button handler**

Find (inside the `updateBtn.addEventListener('click', ...)` handler):

```js
          profiles[idx] = { ...profiles[idx], tempo: S.tempo, tsTop: S.tsTop,
                            tsBot: S.tsBot, rhythm: S.rhythm, swing: S.swing };
```

Replace with:

```js
          profiles[idx] = { ...profiles[idx], tempo: S.tempo, tsTop: S.tsTop,
                            tsBot: S.tsBot, rhythm: S.rhythm, swing: S.swing,
                            mode: S.mode, stepCount: S.stepCount,
                            pattern: S.pattern.slice() };
```

**Step 3: Smoke check**

Reload. Clear existing profiles via devtools (`localStorage.removeItem('metronome-profiles')`) and reload. Switch to Sequencer mode. Toggle a custom pattern (e.g., turn off step 0, turn on steps 2 and 6). Click "+ Save current" in the profiles panel — accept the default name. Open devtools, inspect localStorage `metronome-profiles` — the saved profile should have `mode: "sequencer"`, `stepCount: 16`, and a `pattern` array matching what you toggled. Verify a sibling metronome profile (save one too) also has `mode: "metronome"` and the same fields.

**Step 4: Commit**

```bash
git add index.html
git commit -m "Save mode, stepCount, and pattern with profiles (refs #5)"
```

---

## Task 12: Load sequencer profiles correctly

**Files:**
- Modify: `index.html` — update `applyProfile` (around line 1235–1246).

**Step 1: Update `applyProfile`**

Find:

```js
  function applyProfile(profile) {
    commitTempo(profile.tempo);
    commitTimeSig(profile.tsTop, profile.tsBot);
    commitRhythm(profile.rhythm);
    commitParam('swing', profile.swing);
    elSwingChip.classList.toggle('on', profile.swing);
    elSwingChip.classList.toggle('faded', profile.rhythm === 'quarter');
    S.activeProfileId = profile.id;
    syncActiveProfileName();
    closeProfilesPanel();
    if (!S.playing) start();
  }
```

Replace the body with:

```js
  function applyProfile(profile) {
    const targetMode = profile.mode || 'metronome';
    commitMode(targetMode);
    commitTempo(profile.tempo);
    commitTimeSig(profile.tsTop, profile.tsBot);
    commitRhythm(profile.rhythm);
    commitParam('swing', profile.swing);
    elSwingChip.classList.toggle('on', profile.swing);
    elSwingChip.classList.toggle('faded', profile.rhythm === 'quarter');
    if (targetMode === 'sequencer' && Array.isArray(profile.pattern)) {
      S.pattern = profile.pattern.slice();
      litSeqCells();
    }
    S.activeProfileId = profile.id;
    syncActiveProfileName();
    closeProfilesPanel();
    if (!S.playing) start();
  }
```

Note: `commitMode` and `commitTempo` both call `clearActiveProfile()` internally, but the assignment to `S.activeProfileId` happens after them, so the active-profile pointer is set correctly at the end.

**Step 2: Smoke check**

Reload. Open profiles panel. Click the metronome profile from Task 11 — it should switch to metronome mode and load tempo/time-sig/rhythm. Open profiles panel again. Click the sequencer profile — it should switch to sequencer mode, restore the saved pattern (dots appear on the right cells), and start playing. Toggle off a step in the loaded sequencer profile — the active-profile-name clears (because cell-click calls `clearActiveProfile`).

**Step 3: Commit**

```bash
git add index.html
git commit -m "Load sequencer profiles with mode switch and pattern restore (refs #5)"
```

---

## Task 13: Update default profile name for sequencer mode

**Files:**
- Modify: `index.html` — update `defaultProfileName` (around line 1140–1144).

**Step 1: Update the function**

Replace:

```js
  function defaultProfileName() {
    const sym = { quarter: '♩', eighth: '♪', sixteenth: '♬' };
    const swing = S.swing ? ' swing' : '';
    return `${S.tempo} · ${S.tsTop}/${S.tsBot} · ${sym[S.rhythm]}${swing}`;
  }
```

With:

```js
  function defaultProfileName() {
    if (S.mode === 'sequencer') {
      return `${S.tempo} · seq`;
    }
    const sym = { quarter: '♩', eighth: '♪', sixteenth: '♬' };
    const swing = S.swing ? ' swing' : '';
    return `${S.tempo} · ${S.tsTop}/${S.tsBot} · ${sym[S.rhythm]}${swing}`;
  }
```

**Step 2: Smoke check**

Reload. Switch to Sequencer mode. Click "+ Save current" — the suggested name in the input should be e.g. `120 · seq`. Cancel. Switch to Metronome. Click "+ Save current" — suggested name reverts to e.g. `120 · 4/4 · ♩`.

**Step 3: Commit**

```bash
git add index.html
git commit -m "Use distinct default profile name for sequencer mode (refs #5)"
```

---

## Task 14: Add M/S badges to profile list

**Files:**
- Modify: `index.html` — update `renderProfileList` (around line 1146–1233) and add CSS.

**Step 1: Add CSS for the badge**

In `<style>`, after the `.profile-load-btn` rules (around line 463), add:

```css
    .profile-mode-badge {
      display: inline-block;
      width: 14px;
      height: 14px;
      line-height: 14px;
      text-align: center;
      font-size: 9px;
      font-weight: 600;
      color: var(--text-dim);
      border: 1px solid var(--panel-bdr);
      border-radius: 3px;
      margin-right: 8px;
      vertical-align: 1px;
    }
```

**Step 2: Render the badge inside the load button**

Find this line inside `renderProfileList`:

```js
      loadBtn.textContent = p.name;
```

Replace with:

```js
      const badge = document.createElement('span');
      badge.className = 'profile-mode-badge';
      badge.textContent = p.mode === 'sequencer' ? 'S' : 'M';
      loadBtn.append(badge, p.name);
```

**Step 3: Smoke check**

Reload. Open profiles panel. Each saved profile should have a small `M` or `S` badge before its name. Older metronome profiles (without a `mode` field) show `M`. Sequencer profiles show `S`. Hover the badge — tooltip still works (it's on the parent button).

**Step 4: Commit**

```bash
git add index.html
git commit -m "Add M/S badges to profile list entries (refs #5)"
```

---

## Task 15: End-to-end smoke test and close issue

**Step 1: Full smoke test**

Reload `index.html` fresh in a browser (cleared localStorage if you want a clean slate):

1. **Default state.** Page loads in Metronome mode. Default panels visible. Play — quarter-note metronome at 120 BPM. Stop.
2. **Mode switch.** Click Sequencer. Panels disappear, grid appears with dots on steps 0, 4, 8, 12. Play — quarter pulse, cells light up in sequence. Stop.
3. **Pattern edit.** Toggle off step 4, toggle on steps 2 and 6. Play — new rhythm plays. Toggle step 10 while playing — change happens within the loop.
4. **Mode switch while playing.** While playing in sequencer mode, click Metronome. Metronome resumes from beat 1.
5. **Save sequencer profile.** Switch back to Sequencer, edit a pattern, save profile as "Test seq".
6. **Save metronome profile.** Switch to Metronome, change tempo to 140, save as "Test met".
7. **Profile list.** Open profiles — both profiles show with `S` and `M` badges respectively.
8. **Load profiles.** Click "Test met" — switches to metronome, 140 BPM, starts playing. Click "Test seq" — switches to sequencer, restores pattern, starts playing.
9. **Mobile.** Devtools mobile viewport. Switch to sequencer. Grid is 4×4. Edit pattern, play — works.
10. **Persistence.** Reload page. Open profiles — both profiles still there with correct badges. Load each — restores correctly.

**Step 2: Close issue #5**

```bash
gh issue close 5 --comment "Closed by sequencer implementation. Pattern grid, mode toggle, profile integration, and mobile layout all working. Follow-ups tracked: #7 (post-load profile update prompt), #8 (background visuals for sequencer mode)."
```

**Step 3: No commit needed for the smoke test itself.**

---

## What this plan does NOT do

- **Issue #7** — "update loaded profile after changes" prompt. Filed separately.
- **Issue #8** — background pulse/flash visuals in sequencer mode. Filed separately.
- **Step-count toggle (8 vs 16).** Out of scope per design.
- **Per-step velocity, pitch, or accents.** Out of scope for single-voice v1.
- **Tests.** No test infrastructure in the codebase. Each task includes a manual browser smoke check; the final task is an end-to-end run-through.
