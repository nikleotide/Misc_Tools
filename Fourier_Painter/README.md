
# Fourier Plotter

A single self-contained HTML page that decomposes a shape, a sound, or the outline of a
photograph into a sum of sine waves, animates the decomposition, and exports the series as
runnable Python or R.

Everything happens in the browser. There is no build step, no server, and no data leaves
the page.

---

## Running it

Open `fourier-plotter.html` in a browser. That's all.

- **Size:** ~122 KB, one file.
- **Dependencies:** none, apart from two webfonts (Archivo and IBM Plex Mono) pulled from
  Google Fonts. If you're offline the page falls back to system fonts and works normally.
- **Needs:** a browser with `<dialog>`, Pointer Events, Web Audio and Canvas 2D — anything
  from roughly 2023 onward. Recording needs microphone permission; nothing else asks for any.
- Respects `prefers-reduced-motion` by starting paused.

---

## The idea

Walk once around a closed shape and each coordinate traces out a repeating signal. Feed that
signal through a discrete Fourier transform and you get an amplitude and a phase for every
harmonic. Add the sines back up and the shape reappears. Keep only the first few and you keep
only its broadest strokes.

Every tab is that same operation with a different number of axes:

| Tab | Axes | Input |
| --- | --- | --- |
| **Plane** | x, y | a shape you draw |
| **Space** | x, y, z | a plan view plus a height profile |
| **Sound** | one | a slice of a recording |
| **Image** | x, y | the traced outline of a subject in a photo |

Each axis gets its own stack of rotating vectors. One vector is one sine term; its shadow on
that axis is that term's contribution. Where the stacks agree is where the pen is.

---

## The tabs

### Plane — draw a shape

Draw a closed shape on the gridded bed. The ends are joined automatically, so an open stroke
still becomes a loop. Draw again to add another stroke; strokes are concatenated into one path,
and the connecting jumps will show up in the result.

The stack along the top drives the pen sideways (x); the one down the left drives it up and
down (y). Presets: **Star**, **Heart**, **Square**, **Flower**.

Square is the one to try if you want to see ringing — approximating a corner with smooth sines
never quite settles. Flower shows six-fold symmetry landing on isolated spikes in the spectrum.

### Space — draw in three dimensions

You can't freehand a 3D curve on a flat screen, so input is split the way a drafter would:

- **Plan** panel — draw the loop seen from directly above. This sets x(t) and y(t).
- **Height** panel — drag left to right to set z(t) as the pen travels the loop.

The viewport above shows the result. **Drag to orbit**, **scroll to zoom**. Three stacks now,
one per axis, each in its own coordinate plane; two dashed axis-aligned legs connect each stack
tip to the pen so you can read off which stack contributes which coordinate.

*Match the height at the seam* subtracts a linear ramp so a profile that starts and ends at
different heights still closes into a loop. Turn it off to see the ringing you'd otherwise get.

Presets: **Trefoil knot**, **Crown**, **Lissajous**, **Saddle**.

### Sound — one slice of a recording

**Record** from the microphone or **Upload** an audio file (anything the browser can decode;
long files are trimmed to the first 12 seconds). Or start from a test tone: **Square**,
**Sawtooth**, **Vowel**, **Bell**.

A recording is a wave, and one slice of it is one cycle. The page finds the repeating period by
autocorrelation, resamples it to 512 points, and decomposes it exactly like a drawing — except
there's only one axis, so there's only one stack of vectors, and its vertical position traces
the wave.

- **Drag the waveform strip** to move the window.
- **Window** slider runs from 16 samples up to the entire clip. The line beneath reports the
  sample count, the cycle frequency, and how high 256 harmonics actually reach.
- **One cycle** snaps back to the detected period; **Whole clip** takes everything.
- **Play window** / **Play rebuilt** is the point of the tab. Both play the same window for the
  same duration at the same volume, so the only difference you hear is the missing harmonics.
  A square wave at 4 harmonics is nearly a sine; at 20 it buzzes.

Windows shorter than half a second repeat so you can hear them — one cycle is about 4 ms, which
is a click, not a sound. Longer windows play once, complete, never truncated.

Because a window can only ever hold 256 harmonics, a long window's rebuild is a low-frequency
ghost of the original. It's normalised so you can still hear it, and the line under the buttons
tells you what fraction of the real level it actually carries.

### Image — trace a photograph

**Upload image**. The background colour is estimated from the picture's border, everything far
enough from it becomes the subject, the largest connected region is kept, and its outer edge is
walked pixel by pixel into a closed loop. That loop is then a drawing like any other — same
stacks, same scan mode, same export.

Works best on a clear subject against a plain background. **Edge sensitivity** tunes the cut;
**Flip subject** handles a light subject on a dark background.

When the automatic trace gets it wrong, four tools under *Fix the edge*:

| Tool | What it does |
| --- | --- |
| **Pan** | drag moves the picture |
| **Add** | brush over anything the trace missed |
| **Cut** | brush over anything that shouldn't be included |
| **Draw** | trace the edge yourself; your strokes replace the automatic outline |

Painted areas show as a green or red tint. **Undo my edits** returns to the plain automatic
trace. In Draw mode, *Undo stroke* works per stroke.

Images are downscaled so the longest side is 320 px before segmentation.

---

## Shared controls

**Harmonics k ≤** (1–256) is the cutoff. *Keep the loudest harmonics first* switches from
"first n harmonics" to "n largest terms", which usually looks far better for the same count.

**Trace / Scan** changes the order of arrival, not the maths:

- **Trace** draws in parameter order, the way the pen actually moves.
- **Scan** sweeps a line across the shape's own x axis and marks every y the curve passes at
  that column — two wherever it doubles back, more where it folds again. A circle gives two per
  column; a star reaches four; a flower reaches six. The x stack still drives the line.
  *Scan columns* (12–400) sets the dot density.

**Pen speed** (0–3×) and **Position t** scrub the animation. Space toggles play/pause, `C`
clears the current tab.

**Zoom** (0.4–6×) plus **Reset view**. On any main panel: scroll to zoom about the cursor, drag
to pan — **shift**-drag on the drawing bed, since plain drag draws there.

The channel strips (CH1, CH2, CH3) and the amplitude spectrum sit directly beneath each pane.
Each channel shows the target signal, the reconstruction, the strongest individual sine terms,
and a playhead. Clicking the spectrum sets the harmonic cutoff.

---

## The maths

Every axis is one sine series:

```
value(t) = mean + Σ  A_k · sin(2π·k·t + φ_k),        t ∈ [0, 1)
                k
```

- The path is resampled to **512 points evenly spaced along its length**, so one lap takes t
  from 0 to 1. Even spacing by arc length is what makes t well behaved regardless of how fast
  you drew.
- A light periodic smoothing pass removes pointer jitter before the transform.
- Amplitudes come from the DFT as `A_k = 2·|X_k|`, phases as `arg(X_k) + π/2` (the +π/2 converts
  the natural cosine form to sine), wrapped to (−π, π]. The Nyquist term (k = 256) appears once
  and is handled separately.
- **256 harmonics** is the ceiling — half of 512, the most a 512-point signal can carry.
- **Fit error** in the console is the RMS distance between the reconstruction and the input,
  in model units and as a percentage of the shape's diagonal.

### Units

Analysis happens in a fixed model space, not in screen pixels, so exported numbers don't change
when you resize the window.

- **Plane, Image:** the shape is scaled so its longest side spans 200, centred on the origin.
  x right, y up.
- **Space:** same, with x right, y back, z up.
- **Sound:** amplitude centred on zero and scaled so the loudest point of the window is 100;
  t is one cycle.

---

## Exporting

*Full series and export code* — or the clickable `+N more` in any formula — opens five views:

| View | Contents |
| --- | --- |
| **Table** | every k with its amplitude and phase per axis; copies as TSV |
| **Python** | runnable script, numpy + matplotlib (plotly for 3D) |
| **R** | runnable script, base R (plotly for 3D) |
| **JSON** | structured, with the convention and units recorded |
| **CSV** | `axis,k,amplitude,phase`, one row per term |

A checkbox switches between the terms currently on screen and all 256.

The code exports write the **formula out in full** rather than dumping coordinates:

```python
t = np.linspace(0, 1, 2000)

x = (0.000000
     +  30.564557 * np.sin(2*np.pi*1*t + 0.000000)
     +  71.147320 * np.sin(2*np.pi*2*t + 0.000000)
     ...)
```

So you can edit coefficients, drop harmonics, or paste the expression anywhere. Harmonics whose
amplitude rounds to zero are omitted with a note; an axis with no live terms emits `+ 0*t` so it
stays a vector rather than collapsing to a scalar.

**Requirements for the exported code:**

- 2D and 1D: `numpy` + `matplotlib` (Python); base R only.
- 3D: `plotly` for a rotatable figure — `pip install plotly` / `install.packages("plotly")`.
  Both scripts degrade gracefully if it's missing: Python falls back to matplotlib's 3D axes
  (also drag-rotatable when run as a script), R falls back to a static projection and tells you
  the install command. Neither errors out.

`rgl` is deliberately **not** used for R. It needs OpenGL, which means XQuartz on macOS, and it
fails at library-load time in a way no guard can catch. plotly renders in the browser instead.

---

## Known limits

- **256 harmonics, always.** Fine for shapes; it means a long audio window can only reach
  `256 / window duration` Hz, so its rebuild will be a rumble. The window readout tells you the
  number.
- **Multiple strokes are concatenated**, not treated as separate loops, so the joins appear in
  the result.
- **Image segmentation is colour-distance based**, not machine learning. Busy or low-contrast
  backgrounds need the sensitivity slider, or the Add/Cut brushes, or Draw.
- **Scan mode needs an x axis**, so it isn't offered on the Sound tab.
- **Recording** produces whatever format the browser hands back; it's decoded through Web Audio,
  so anything the browser can play, it can analyse.

---

## Modifying it

The file is one HTML document: styles in `<style>`, everything else in a single `<script>`.
Useful constants sit at the top of the script:

| Constant | Meaning |
| --- | --- |
| `N = 512` | samples per loop; `HALF` (256) is the harmonic ceiling |
| `TRACE_RES = 900` | resolution of the drawn reconstruction |
| `MODEL_R = 100` | shape half-extent in model units |
| `GAP2D`, `GAP3D` | how far the vector stacks sit from the shape |
| `HN = 257` | bins in the 3D height profile |
| `ZR = 150` | height-panel range in model units |
| `LEN_MIN = 16` | shortest audio window, in samples |

The script is organised as: geometry helpers → DFT → rigs (one per tab) → renderers → analysis
pipelines → export codegen → input handlers → controls → render loop. A "rig" holds one path,
one series per axis, the active terms, and the reconstruction; the renderers are the only
tab-specific part, and the planar renderer is shared between the Plane and Image tabs.

Palette: iron-red `#B4322A` for x, prussian blue `#1E4F8A` for y, sap green `#3F6B33` for z,
violet `#6B3FA0` for sound, on oyster paper `#E6E7DE`.
