# Simple Metronome

## What
A simple web-based metronome app. Open `index.html` in any browser — no install, no internet connection required, no dependencies.

## Why
A simple metronome with sounds that aren't annoying, for practicing piano and staying focused. The goal was a tool that doesn't rise above the piano keys but is discernible when you need it.

Also: an experiment in building with Claude Code.

## Design
- Tempo is the most obvious thing on screen at all times
- When the metronome is running, there is movement — the background gradient pulses with the beat
- Pleasing and calming, like walking through a meadow but abstract, not graphical
- Not like walking into an Apple Store. Not like a lava lamp.
- Beat 1 always sounds and looks different — you always know where you are in the measure

## Features

### Tempo
- The current tempo is always displayed prominently
- **Click** the tempo number to type a new value, then press **Enter** to confirm
- **↑ / ↓ arrow keys** shift the tempo one BPM at a time
- **Slider** for large changes
- All changes while the metronome is running take effect at the end of the current measure — if you're typing, the metronome waits for Enter so it doesn't jump to intermediate values mid-thought

### Time Signature
- Common signatures are one click away: 2/4, 3/4, 4/4, 5/4, 6/8
- Custom entry: type any numerator and denominator directly
- Displayed in stacked musical notation style, without a fraction bar

### Rhythm
- **Quarter notes** — one click per beat
- **Eighth notes** — two clicks per beat; the on-beat click is louder
- **Sixteenth notes** — four clicks per beat; on-beat clicks are louder, subdivisions are quieter
- **Swing** — applies a 2:1 (triplet) swing feel to eighth and sixteenth subdivisions; toggle by clicking the Swing button or pressing **S**

### Sound
- Synthesized low-frequency thud: a sine wave that pitches down on impact, layered with a brief burst of lowpass-filtered noise
- Beat 1 (the downbeat) carries more weight — slightly louder and at a higher starting frequency
- Subdivisions are noticeably quieter so the beat structure stays clear

### Visual
- A row of dots shows your position in the measure; beat 1 is a distinct color
- The background pulses on each beat with a keyframe animation: rapid peak on impact, brief bloom, then a slow gather and anticipation glow before the next beat

## Keyboard Shortcuts
| Key | Action |
|---|---|
| `Space` | Start / stop |
| `↑` / `↓` | Tempo +1 / -1 BPM |
| `S` | Toggle swing |
| Click tempo | Edit tempo by typing |
| `Enter` | Confirm typed tempo |
| `Escape` | Cancel tempo edit |

## Architecture
A single `index.html` file with embedded CSS and JavaScript. No frameworks, no build step, no network requests. The audio is generated entirely by the browser's Web Audio API.
