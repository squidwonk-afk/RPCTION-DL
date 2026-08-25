# RPCTION

**A universal visual interaction layer for your screen.**

SEE → UNDERSTAND → TRACK → ACT → REPLICATE

If you can see it, you can interact with it.

Press a shortcut and point at something on screen. Your desktop keeps moving
and keeps responding underneath; hold `Ctrl` and click when you want RPCTION to
take the target. RPCTION works out what you pointed at — a URL, a phone
number, an address, a QR code, a table, a paragraph, an object inside a
photograph — and offers what you can do with it.
It does not care which application drew those pixels.

**Windows 10/11 · v0.1.1 · free · local · no account**

---

## Installing

Download `RPCTION-Setup-0.1.1.exe` and run it. The installer is per-user and does
not need administrator rights. A portable single-executable build is also
available.

**Windows will block this build, and how it blocks depends on your machine.**
This build is not code-signed, and Windows has two separate protections that
react to that very differently.

**SmartScreen** shows "Windows protected your PC". Choose **More info** →
**Run anyway** and it continues. That is a warning, not a virus detection.

**Smart App Control** refuses unsigned applications outright. There is no
"Run anyway", and downloading again or waiting does not help — it judges on
signing, not on reputation. It is on by default on many clean Windows 11
installs. Check under **Windows Security → App & browser control → Smart App
Control**. If it is on, this build will not run on your machine.

We are not going to tell you to switch it off. It cannot be switched back on
without resetting Windows, and trading a security feature for one small tool is
a bad deal. The fix is a signed build, and that is being worked on.

The SHA-256 of every published file is in [RELEASE_NOTES.md](RELEASE_NOTES.md).

To uninstall: Settings › Apps › Installed apps › RPCTION. Settings are left
behind on purpose so a reinstall keeps your hotkey; delete
`%APPDATA%RPCTIONsettings.json` for a completely clean removal.

### Building it yourself

```bash
npm install
npm start          # run from source
npm run pack       # build the installer into release/
```

`npm install` also vendors the OCR model and WASM core into `assets/` (a one-time
download of ~2 MB). After that the product never touches the network.

RPCTION starts into the system tray. First launch shows a three-step onboarding.

**Default shortcut: `CTRL + SHIFT + R`** — rebindable to any key combination in
Settings (letters, digits, function keys, punctuation, with any modifiers).

### Using it

| Input | Result |
| --- | --- |
| Move the cursor | The target under it is outlined and labelled |
| Hold `Ctrl` | Enters SELECT: RPCTION takes the mouse for as long as you hold it |
| `Ctrl` + left click | Performs the primary action — always a harmless copy/extract |
| `Ctrl` + right click / `Space` | Opens the action bar, including actions that leave RPCTION |
| Plain click, scroll, drag | Goes to the application underneath. RPCTION is watching, not blocking |
| `Tab` / `Shift+Tab` | Cycle modes (AUTO … MOTION, SOURCE, AUDIO) |
| `↑` / `↓` | Walk the hierarchy out and back in (word → line → paragraph → image) |
| `Esc` | Dismiss |

Actions marked `↗` hand the value to another application (browser, dialer,
maps). Those are never performed by a plain click — you pick them.

---

## What actually works

Everything below is covered by the test suites and was verified end to end, not
just wired up.

| Capability | Status |
| --- | --- |
| Global hotkey, tray, transparent overlay, cursor tracking | working |
| Local OCR with word/line/paragraph/block boxes | working, offline |
| URL, email, phone detection with normalization | working |
| Address detection | working, heuristic — confidence is reported honestly |
| Table detection and tab-separated extraction | working |
| QR detection and decoding | working, offline |
| Image-region detection (photographs and framed panels) | working |
| Object segmentation to a transparent PNG | working, with an honest confidence gate |
| Clipboard text and image output | working |
| Settings, onboarding, rebindable shortcut | working |
| **Motion Mode** — live buffer, tracking, temporal export | working |

### Deliberately not claimed

- **Object segmentation is a local, edge-aware region grower**, not a
  learned model. It is genuinely good on logos, icons, product shots and objects
  against separable backgrounds. On a cluttered photo it will sometimes decline —
  below a confidence threshold it shows no silhouette rather than a wrong one,
  and the plain box remains available.
- **Address detection is heuristic.** It requires a thoroughfare or country word
  plus supporting signals, so it under-reports rather than guessing.
- **Table extraction prioritises correct text in reading order** over perfect
  structural reconstruction, per the v0.1 brief.
- **No installer is built.** `npm start` runs it. Packaging is not wired up.
- **English OCR only.** More languages are a matter of vendoring more model files.

If a subsystem fails it degrades on its own and says so in the overlay status bar
(`TEXT DETECTION UNAVAILABLE`, `SCREEN ACCESS REQUIRED`), and the others keep
working. There is no fake success anywhere: the silhouette you see drawn *is* the
alpha channel that lands on your clipboard.

---

## One tool, thirteen modes

RPCTION has one hotkey and one overlay. Tab changes what it is looking for;
click tells it what to do. Everything else — capture, resolution, the action
system, the clipboard guarantees — is shared, and a mode is a way of asking a
different question of the same pixels.

```
AUTO  TEXT  OBJECT  PART  TEXTURE  IMAGE  LINK  CONTACT  TABLE  ACT  MOTION  SOURCE  AUDIO
```

Tab cycles forward, Shift+Tab back, and disabled modes are skipped rather than
shown greyed out. The overlay holds focus while it is up, so the application
underneath never receives the mode-switching Tab.

Adding a mode touches three places: the `Mode` union, the `MODES` array (which
is the Tab order), and `MODE_WEIGHTS` in the resolver — and the last of those is
an exhaustive `Record<Mode, …>`, so **the compiler refuses to build a mode that
has not been registered.** That is the whole extension mechanism, and it is why
adding two modes did not require touching the other eleven.

`MODES` is appended to and never reordered. It is both the default cycle order
and the migration's notion of what exists, so moving an entry would silently
change the order for every existing user.

### SOURCE

Point at code — in an editor, a screenshot, a slide, a paused video — and click.
RPCTION reads it back into text, rebuilds the indentation, and offers COPY or
SAVE with an extension matching what the code actually is.

**Every result carries its origin, and the origin is not optional.** A
`SourceResult` cannot be constructed without a provenance:

| provenance | meaning |
| --- | --- |
| `SOURCE AVAILABLE` | obtained from a real source, byte for byte |
| `SOURCE RECONSTRUCTED` | rebuilt from what was visible; may differ |
| `SOURCE PARTIAL` | real, but some lines were read with low confidence |
| `SOURCE UNAVAILABLE` | nothing legitimate could be obtained |

**Nothing in RPCTION can currently produce `AVAILABLE`**, and there is a live
assertion saying so. RPCTION captures pixels; a desktop application has no
legitimate route into another application's DOM, and fetching the page over the
network would break the guarantee that nothing leaves the machine. So the two
highest-fidelity tiers of the hierarchy have no provider, and the honest answer
is that they are out of reach — not an approximation wearing their label.

The providers are a registry ordered by fidelity:

```
50  direct resource access      [no provider - would need a browser extension]
40  accessible content tree     [no provider - would need the a11y tree]
30  visible text already recognised
20  OCR reconstruction
10  visual reconstruction       [no provider - would be fabrication]
```

A `BrowserProvider` or an `AccessibilityProvider` registers at a higher fidelity
and wins automatically. Nothing else in the mode moves.

**Text read off the screen is `RECONSTRUCTED`, never `AVAILABLE`** — even when
it comes from the graph and needed no extra work. It is a reading of rendered
glyphs, and a character OCR got wrong is wrong here too. Calling it `available`
because it was cheap to obtain is exactly the lie the mode exists to prevent.

#### Indentation, recovered rather than guessed

OCR strips leading whitespace — there are no glyphs to recognise. For prose that
costs nothing; for source it destroys the structure. But the indentation is
still on screen as horizontal offset, so it is measured: find the leftmost line,
estimate one character's width from the median glyph box, and express every
other line's offset in characters, snapped to a two-space grid. Blank lines come
back from vertical gaps larger than the median line pitch.

#### What it refuses

A photograph, a UI screenshot, a paragraph of prose: `SOURCE UNAVAILABLE`, with
a reason. Text that is readable but not code is not a partial success. Prose is
never assigned a language, because the language is what decides the file
extension — and code-shaped text whose language is not established saves as
`.txt` rather than acquiring a guessed `.js`.

### AUDIO

Captures the last few seconds of what the machine is playing. Cycle to AUDIO,
choose 2/5/10/30 seconds with the arrow keys, click. The clip is written as a
`.wav` and the confirmation says **SAVED** with a path — never COPIED, because
Windows has no pasteable clipboard form for audio.

**Nothing is buffered until the mode is entered.** The rolling buffer is
allocated on entry and released on exit, and that is structural rather than a
matter of remembering:

- the ring is **allocated once at a fixed size**, so a bug that forgets to stop
  writing physically cannot become a recording of the afternoon — 30 seconds of
  stereo at 48kHz is 11.5MB and cannot grow;
- `clear()` **drops the arrays** rather than resetting an index, so leaving the
  mode returns the memory and leaves nothing for a later bug to read;
- teardown releases the buffer **even when closing the device throws**, because
  a failed teardown still holding buffered audio is the worst outcome available.

Verified at the OS level: with RPCTION idle, Windows audio session modules are
not loaded into the process at all.

Capture is system audio via the display-media loopback stream. The video track
that necessarily comes with it is stopped the instant the stream arrives — it is
requested only because Chromium's display-media contract requires a display to
attach to, and keeping it would mean AUDIO mode was quietly capturing the
screen. **Per-application isolation is not claimed**, because Windows mixes
system audio before applications can see it and promising otherwise would be
lying about the mode's own output.

WAV encoding is written here rather than pulled in — RIFF/WAVE is a 44-byte
header in front of the samples, and a dependency for that would be larger than
the code and would need auditing for network access. The one real decision in it
is the clamp: floats beyond ±1 happen whenever two loud sources mix, and the
int16 conversion *wraps* rather than saturating, so an unclamped hot sample
becomes a large negative number and the loudest moment turns into a click.

**No transcription.** There is no local speech engine in the project, and adding
a cloud one is not on the table. Capture is the feature; transcription can be an
extension when a local engine is worth its size.

---

### The icon

One file: [`assets/icon.png`](assets/icon.png). Every other icon in the product
— the Windows executable, the installer, the tray, the window, the website
favicon — is derived from it mechanically by `npm run icon`, which runs as part
of every build. Nothing regenerates the canonical asset, and the test suite
compares the derived Windows icon against it byte for byte so a stale one fails
rather than ships. See [assets/README.md](assets/README.md).

---

## Privacy

Local first, and literally so:

- The OCR engine, model, QR decoder and segmenter are bundled and run on-device.
- No account, no API key, no backend, no telemetry.
- Screen contents are never transmitted. Pull the network cable and everything
  above still works.
- The only outbound actions are the ones you explicitly pick — OPEN, SEARCH,
  MAP, CALL, MESSAGE — which hand the selected value to another local app.

Capture buffers are released when a new activation supersedes them.

---

## Architecture

Four processes with hard boundaries, so perception, capability and presentation
never leak into each other.

```
main (Electron)         hotkey · tray · capture · clipboard · actions · settings
  │
  ├── engine (hidden)   OCR · QR · layout · segmentation · semantic graph
  │                     + motion: live buffer · tracking · temporal export
  │
  ├── overlay           target resolution · highlight · action bar
  │
  └── ui                onboarding · settings
```

The overlay can describe what is on offer but holds no capability to perform it —
every action is executed in `main`. "The user always decides" is structural, not
a matter of discipline.

### The pipeline

```
display → capture → layout regions ──┐
                    OCR near cursor ─┼→ semantic graph → target resolution → actions
                    OCR full screen ─┤
                    QR codes ────────┘
```

Each stage publishes a graph as soon as it has one, so the overlay sharpens
instead of blocking. OCR runs on the window around the cursor *first*: the user
is pointing at one place, and that place becomes actionable in a fraction of the
time a full-screen pass takes.

### Coordinate spaces

`src/shared/geometry.ts` owns the mapping between screen DIP, display DIP,
capture pixels and overlay CSS pixels. The capture scale is always derived from
the bitmap actually received rather than assumed from the OS scale factor —
which is what keeps target selection correct on high-DPI and scaled displays.

### Source map

| Path | Responsibility |
| --- | --- |
| `src/shared/types.ts` | The semantic graph vocabulary |
| `src/shared/geometry.ts` | Coordinate mapping |
| `src/shared/patterns.ts` | URL/email/phone/address recognizers |
| `src/shared/resolver.ts` | What is the user pointing at |
| `src/shared/actions.ts` | Action registry |
| `src/shared/hotkey.ts` | Accelerator capture and display |
| `src/engine/ocr.ts` | Local OCR, full-frame and regional |
| `src/engine/layout.ts` | Photographic and framed-panel region detection |
| `src/engine/segment.ts` | Cursor-seeded object segmentation |
| `src/engine/qr.ts` | QR sweep |
| `src/engine/graph.ts` | Detections → hierarchy |
| `src/motion/buffer.ts` | Bounded rolling frame buffer |
| `src/motion/capture.ts` | Live display stream → timestamped frames |
| `src/motion/tracker.ts` | Correlation + colour tracking |
| `src/motion/masks.ts` | Per-frame masks, temporally smoothed |
| `src/motion/gif.ts` | In-tree GIF89a encoder |
| `src/motion/encode.ts` | WebM / GIF / PNG-sequence output |
| `src/motion/engine.ts` | Motion lifecycle orchestration |
| `src/main/*` | OS surface |
| `src/overlay/*` | The interaction surface |

---

## Release quality vs. research suite

These are two different bars and conflating them would be dishonest in both
directions.

The **research suite** asks whether the perception engine has understood a
scene, on fixtures built specifically to defeat it: objects that touch at
identical tone, things behind glass, reflections, images inside images,
photographs degraded until a person would struggle. Its job is to keep finding
the edge of what the engine can do. A red check there is a research result.

The **release bar** asks something narrower: can this be handed to somebody. It
is the list in the next section, and every item on it is met.

A useful way to hold the difference: the adversarial suite is at 76 of 92, and
the ordinary-perception suite is at 59 of 59. The failures are concentrated
where they were designed to be.

### Was the refusal principle enforceable? No, and here is the measurement

RPCTION's product rule is **wrong is worse than empty**: if the engine cannot
tell what you pointed at, it should refuse rather than hand back something else.
OBJECT mode has a refusal gate for exactly this.

Across the entire adversarial corpus that gate fires **zero times**. Every one of
the sixteen failures is a wrong answer delivered confidently.

The obvious response is to raise the gate. That only works if wrong answers are
separable from right ones by something measurable, so every judged selection in
the corpus was swept and scored:

| OBJECT score | min | p10 | median | p90 | max |
| --- | --- | --- | --- | --- | --- |
| correct (49) | 0.52 | 0.81 | **0.88** | 0.95 | 0.95 |
| wrong (26) | 0.65 | 0.73 | **0.86** | 0.88 | 0.90 |

**The distributions sit on top of each other.** Nineteen of the twenty-six wrong
answers score at or above the tenth percentile of the correct ones. The best
threshold available — 0.80 — refuses 6 wrong answers and destroys 3 correct ones,
which is not a trade worth making, and every gate below it is strictly negative.

So the gate stays where it is, and the honest statement is that **RPCTION's
confidence number does not know when it is wrong**. Raising it would buy silence,
not trust. This is recorded as a live assertion so that nobody, including me,
quietly re-tunes it later on the strength of how sensible it sounds.

What carries the principle instead is the interaction, and it is load-bearing
rather than incidental: **the selection is drawn before anything happens to it.**
Hovering shows the mask; acting requires a click; `↑` and `↓` walk up and down
the hierarchy. Every failure below is a visible one the user can see and correct
before it reaches their clipboard — which is a materially different situation
from a tool that silently copies the wrong thing.

### The sixteen, classified

Categories: **A** release blocker · **B** serious quality issue · **C** edge case
· **D** fundamentally ambiguous · **E** intentionally unsupported.

| # | failure | category | real-world frequency | what the user gets | class |
| --- | --- | --- | --- | --- | --- |
| 1–3 | two figures touching at identical skin tone | BOUNDARY_ABSENCE | low | one mask over both, or a spill | **D** |
| 4 | a phone held against a figure | CONTAINMENT | low | the figure instead of the phone | **C** |
| 5–6 | a figure standing behind a table | OCCLUSION | medium | mask swallows the table | **B** |
| 7 | remote control with a dark keypad | TEXTURE | low | keypad omitted from the remote | **C** |
| 8–9 | an object resting on marble / carpet | TEXTURE | **high** | mask spills into the surface | **B** |
| 10 | an object above its own reflection | REFLECTION | low | reflection scores as a real object | **C** |
| 11 | a pane of glass | TRANSPARENCY | low | fabricates an object | **C** |
| 12 | a figure inside a photo inside the scene | IMAGE-IN-IMAGE | medium | wrong level of the nesting | **C** |
| 13–16 | jpeg-mid, jpeg-low, downscale, blur | DEGRADATION | medium | fragments, or a merged blob | **C** |

**No category A.** None of these can cause data loss, a wrong copy the user
cannot see, a crash, a privacy leak, a broken install, or a false clipboard
confirmation. Each produces a visibly wrong highlight, before any action, that
`↑`/`↓` corrects.

**Two category B**, and they are the two worth naming honestly rather than
burying:

- **An object on a strongly patterned surface** (marble, carpet) is the one that
  will actually be met, because desks have woodgrain and floors have carpet. The
  mask spills into the surface. The failure is visible and correctable, and it is
  the first thing to fix after v0.1.
- **An object partly behind another** merges the two. Occlusion is measurable in
  principle — the contour work in `contour.ts` was built and measured for exactly
  this — but it did not separate the cases well enough to ship, and that
  measurement is recorded rather than the code being kept because it sounded
  right.

**One category D.** Two objects of identical colour whose contours meet produce
no boundary at all. There is nothing in the frame that distinguishes them, and no
threshold can invent one. Motion resolves it — that is what the temporal identity
work established — but a single frame cannot.

**No category E.** Nothing here is deliberately unsupported; every one is
something RPCTION tries to do and does not always get right.

---

## Testing

```bash
npm test
```

Fourteen suites, 821 checks (805 passing; the 16 known failures are all in the
adversarial suite, are named and classified above, and are identical on every
run):

- **`src/shared/patterns.test.ts`** — pure logic. Recognizers and target
  resolution, including regressions for the false positives that were found and
  fixed (a table row reading as an address, a decimal reading as a URL, a word
  outranking the phone number it belongs to).
- **`src/motion/tracker.test.ts`** — tracking against seven synthetic sequences
  with known ground truth, in plain Node so the loop stays fast.
- **`src/engine/source/source.test.ts`** — SOURCE mode, in plain Node. Language
  detection with prose as the negative control, geometric reconstruction of
  indentation, and provenance — including the assertion that nothing can claim
  `AVAILABLE` while only pixels are reachable.
- **`src/shared/icon.test.ts`** — one icon, one source: that the canonical asset
  is never regenerated, that the derived Windows icon is current with it byte
  for byte, and that no second drawing of the mark has reappeared.
- **`src/shared/trust.test.ts`** — the trust invariant, asserted against the
  source: that the current target is never set without being drawn, and that an
  action only ever runs on the target that was drawn. The alternative to reading
  the source here is a full renderer harness; the alternative to that is
  trusting a comment.
- **`src/engine/audio/audio.test.ts`** — AUDIO mode, on generated PCM so it needs
  no sound device and answers the same on every machine. WAV headers read back
  from the encoded file rather than compared with the input, the ring keeping the
  LAST n seconds rather than the first, and the buffer actually being released.
- The Motion suite also carries the **temporal evidence audit** — whether time
  supplies information a single frame cannot. See
  [Does time give RPCTION information a single frame cannot?](#does-time-give-rpction-information-a-single-frame-cannot).
- **`scripts/motiontest.mjs`** — the whole Motion pipeline in a real renderer:
  lock, track, segment, encode, then **decode the output back** and check that
  the crop followed the target, that a motion crop is fully opaque, and that an
  extraction is transparent at the corners and opaque on the subject.
- **`scripts/selftest.mjs`** — the still pipeline in a real Electron renderer,
  same WASM and canvas the product uses, against synthetic screens that mirror
  the scenarios in the brief: a Discord message with a URL, a screenshot inside a
  screenshot, an address, an image region, object extraction, a QR code, a table,
  dark-mode small text, and the empty-screen fallback. It asserts on the semantic
  graph — type, position and extracted value — not on whether the UI looked busy.
- **`scripts/semantictest.mjs`** — the object hierarchy against a matrix of
  drawn scenes with pixel-exact ground truth, including four negatives that must
  produce no object at all. See [Semantic object hierarchy](#semantic-object-hierarchy).
- **`scripts/advtest.mjs`** — the same engine under deliberately hostile
  conditions: objects that touch, overlap, occlude, hide against their own
  background, carry internal edges harder than their silhouettes, and arrive
  compressed. Measured as hit RATES over sweeps of cursor positions rather than
  single probes. See [Adversarial robustness](#adversarial-robustness).

### Visual verification

```bash
npm run preview
```

Renders the real overlay over mock desktops driven by the real engine and writes
PNGs to `out/preview/`. Every highlight in those images came out of the actual
pipeline. This is how the overlay was checked without taking over a display.

---

## Motion Mode

**Replicate what you see, moving.**

Everything above operates on one frame. Motion Mode operates on an *entity
across time*: point at something moving — in a video, a game, a Discord or Zoom
screen share, anything the display is drawing — and RPCTION follows that thing
and exports it.

This is not screen recording followed by a crop. The thing being copied is the
target, not the screen it happened to be on.

```
CTRL + SHIFT + R  →  TAB to MOTION  →  point  →  click  →  2/5/10 SEC  →  output
```

| Input | Result |
| --- | --- |
| Move the cursor | The moving thing under it is outlined |
| Click | Locks the target; tracking starts and the temporal control opens |
| `1` `2` `3` | 2 / 5 / 10 seconds |
| `←` `→` | Choose the output |
| `Enter`, or click a selected output again | Extract |
| `Esc` | Cancel immediately — tracking stops, buffers are released |

### The two operations are different, and the UI says so

| Output | What you get |
| --- | --- |
| `VIDEO` · `GIF` | **MOTION CROP** — the rectangular region, following the target's path |
| `EXTRACT` · `OBJ GIF` · `PNG SEQ` | **TRANSPARENT** — the entity isolated, background removed |

A motion crop is never labelled an extraction. If per-frame segmentation
succeeds on fewer than half the frames, the export degrades to a motion crop and
the confirmation reads `SAVED MOTION CROP` rather than claiming an object was
isolated.

### How it works

```
display → live buffer → acquire → track → segment → smooth → encode
             (bounded)              ↑ live stage    ↑ export stage
```

- **Rolling buffer.** Frames are held as GPU-backed bitmaps in a ring sized by
  an explicit memory budget, not a guessed frame count. It arms when you enter
  Motion Mode and is released the moment you leave — nothing is buffered during
  ordinary desktop use, and nothing is ever written to disk until you export.
- **Tracking** is a local correlation tracker: normalized cross-correlation over
  a search window aimed by a constant-velocity motion model, with scale search,
  a pristine template kept for re-acquisition, and coasting through occlusion.
  A discriminative colour model runs alongside it — correlation needs internal
  structure, and a solid-coloured target has none, so the two cover each other.
- **Segmentation** is tracker-guided. Masks accumulate in target-relative space,
  which cancels the target's own motion and turns a plain temporal average into
  a motion-compensated smooth. That is what stops the edges crawling.
- **The export crop is the trajectory**, not the screen — the union of the boxes
  the target actually occupied, plus padding.
- **RPCTION's overlay is excluded from its own capture**, so the highlight and
  the temporal control never end up baked into your clip.

### Verified, not assumed

- VP8/VP9 WebM from `MediaRecorder` **does** preserve the canvas alpha channel on
  this build. That was probed before the feature was designed around it, and the
  test decodes the exported video back and checks that corners are transparent
  while the subject is opaque.
- The GIF encoder is written in-tree (median-cut palette, LZW, transparent
  index) rather than pulled in, keeping the "nothing is fetched" promise intact.
- Tracking is checked against seven synthetic sequences with known ground truth
  — person, ball on a busy background, occlusion behind a pillar, scale change,
  fast motion, low contrast, game character over scrolling scenery — asserting
  both accuracy *and* that the box travelled, because a tracker that reports the
  click position forever looks fine until you check displacement.

### Deliberately not claimed

- **AFTER only.** The buffer holds history, and the plumbing for "the last 5
  seconds" is in place, but the tracker runs *forward* from the lock — there is
  no tracking data behind that moment, so a BEFORE window would be guessing at
  positions it never observed. Frames before the lock reuse the locked box.
- **Segmentation quality is the still segmenter's.** Excellent on separable
  subjects, weaker on a subject that shares its background's colours. The
  confidence gate and the motion-crop fallback exist for exactly that case.
- **Export runs in real time.** `MediaRecorder` timestamps against the wall
  clock, so a 5-second clip takes about 5 seconds to encode. Progress is shown;
  the app does not block.
- **WebM files carry no Duration header.** `MediaRecorder` omits it on
  live-muxed output. The frames and their timing are correct and every player
  tested handles it, but some tools will show the duration as unknown until
  played through.
- **One target at a time.** Multi-object selection is architected for, not built.
- **Outputs are saved to disk, not copied.** Windows has no reliable way to put
  a video or GIF on the clipboard as a pasteable object, so RPCTION opens a save
  dialog and puts the resulting *path* on the clipboard as text. Claiming
  `COPIED` for something you could not then paste would be a lie.

---

## The website

```bash
npm run site        # build to site/dist
npm run site:dev    # build, watch, serve on :4321
```

Static HTML plus one 9 kB module. **No framework** — the site has exactly one
genuinely interactive component, and shipping a runtime to render the other 95%
would cost first paint and buy nothing. It reuses the esbuild pipeline that was
already here, so it adds zero dependencies and deploys to any static host.

| Route | |
| --- | --- |
| `/` | Hero, live demo, problem, SEE/UNDERSTAND/ACT, modes, object, text, links, motion, privacy, speed, download |
| `/download/` | Version, platform, and how to actually run it |
| `/docs/…` | Getting started, modes, shortcuts, permissions, privacy, troubleshooting, FAQ |

### The demo is the product, not a picture of it

`site/client/demo.ts` renders the same marks the desktop overlay renders — the
white-on-black box with corner ticks, the `TYPE | value | CLICK COPY` chip, the
action bar with `↗` on anything that leaves the app, the terse confirmation. It
measures the real DOM elements it points at, so a highlight lands exactly on the
text at any viewport size.

It also **yields to the visitor**: move your pointer over the simulated screen
and the scripted tour steps aside and starts responding to you. Targets are
declared as data (`{ type, value, actions, primary, result }`), so one component
covers every scenario on the site.

### What the site claims

Only what is true. The download page is generated from `site/release.json`,
which is written by the job that uploads the binaries, so it can only describe
files that exist: a platform with no artifact in the manifest gets no install
instructions, no button and no security note, and is listed as NOT BUILT. A
test enforces that. Built and tested are shown as separate facts, because an
artifact that has been launched once by CI is not one a person has used.
No user counts, no logos, no testimonials, no benchmarks.

### Verification

```bash
npm run site:shots   # renders 48 screenshots across desktop/laptop/mobile
npm run site:audit   # accessibility, metadata, no-JS, keyboard reachability
```

`site/shots.mjs` renders the built site offscreen at three viewport sizes and
flags horizontal overflow, dead internal links, console errors and missing
`<h1>`s. `site/audit.mjs` checks that every control has an accessible name and
can take focus, that headings descend without skipping, that the metadata is
present, and that the page's actual words are in the static HTML rather than
injected by script.

Both pass clean. Several real defects were found this way and fixed: a hero
headline that filled the viewport and pushed the demo below the fold, a demo
that sized itself off the viewport instead of its own container (so the same
markup overflowed in one place and floated in another), overlay marks wider than
the screen they annotated on mobile, and two heading-order violations.

---

## Semantic object hierarchy

**Anything can be copied. Only meaningful things are default objects.**

Point at someone's shirt and RPCTION selects **the whole figure**. Point at a
wheel and it selects **the car**. Point at a phone resting on a laptop and it
selects **the phone**, because that is a different thing that happens to be
touching another one. Point at a gradient, a shadow or a patch of noise and it
selects **nothing**, because there is nothing there.

### There is no object classifier, and that is the interesting part

Nothing in RPCTION knows what a person is. It has no model, no labels, no
network. The labels it produces are structural — `WHOLE_OBJECT`, `PART`,
`TEXTURE`, `BACKGROUND`, `REGION` — and never categorical. There is a registered
seam (`registerClassifier`) where a real local classifier can one day contribute
a name; with nothing registered, which is the shipping state, the engine works
exactly as it does now. A classifier is an extra signal, never a dependency.

### A merge tree is not an object tree

Weak boundaries say two regions *could* belong together. They say nothing about
whether the result is a thing. Two visually similar regions can be one object,
two objects, or an object and the wall behind it. Everything below exists to
make that second judgement.

Four hypotheses — whole object, part, texture, background — are scored from
named evidence and the strongest wins, so a wrong answer can always be traced to
the signal that produced it:

- **Self-containment.** What is on the other side of the outline: open
  background, or more foreground? A figure's contour has nothing but background
  beyond it. "Head plus shirt plus one arm" is bounded by a hard silhouette on
  five sides and runs through the middle of the subject on the sixth. A contour
  that stops inside the scene is evidence of a part. The exception is a phone on
  a laptop, which is surrounded by another object and is still an object — so a
  contour facing other foreground can earn its keep by being as hard as a real
  depth edge, and only then.
- **Boundary contrast.** External boundary strength against internal. A coherent
  object is bounded harder than it is divided; a texture patch is divided as
  hard as it is bounded. This one is treated as *necessary* — it scales the
  whole object score rather than adding to it.
- **Boundary uniformity.** Is the silhouette equally convincing on every side? A
  whole object is bounded by the same kind of edge all the way round. This is
  what a saturating "fraction of outline supported" measurement cannot see.
- **Completeness.** Of the outline that has no edge under it, how much has the
  *same material* on the far side? More of the same beyond the boundary means
  the region carries on and we cut it arbitrarily.
- **Texture.** Periodicity *or* micro-contrast, with a smooth gradient failing
  both. And a strong closed silhouette outranks a busy interior: a patterned
  tile is an object with a texture in it, not a texture.

### How the hierarchy is built

```
oversegment  →  merge weakest boundary first  →  merge tree = hypotheses
                                                        +
                    flood the background inward  →  whole-subject candidate
                                                        ↓
                        prune · deduplicate · measure · classify · score
                                                        ↓
                                              resolve for the active mode
```

Two independent candidate generators, because one was not enough. Agglomerative
merging assembles an object from its parts, and on a figure whose limbs meet the
torso across long weak boundaries it can stall halfway — leaving "upper body"
and "lower body" as the only levels that ever exist. No resolution logic can
recover a level that is not in the tree. So a second generator comes at it from
the other side: flood inward from the window border, stop at strong contours,
and whatever the flood never reaches is the subject. That one change took
whole-person selection from 0/5 to 5/5.

That flood also gives the engine an actual **background model**, which is what
lets it distinguish "surrounded by background" from "*is* background" — a
distinction that a scene made entirely of gradient depends on completely.

**Semantic compression.** A segmenter can produce a candidate at every pixel
count between one and the whole screen; almost none of them are choices anyone
wants. Candidates that are too small, too fragmented, or the same target at a
slightly different pixel count — compared by mask overlap, not by size — are
pruned, and the pruning reasons appear in the debug readout. Typical scenes
resolve to three to eight levels.

### Modes

| Mode | Gives you |
| --- | --- |
| `OBJECT` | The smallest independent coherent entity — not the smallest region, not the largest |
| `PART` | A component of that thing: a shirt, a wheel, a lens |
| `TEXTURE` | The material, at the largest extent still reading as that material |

`PART` is defined *relative to* `OBJECT`: find the level OBJECT would choose,
then take the best-scoring component strictly inside it. Scoring "part-ness"
independently kept returning either the whole object again or a crumb of
surface, because being a component is not a property a region has on its own.

The **mouse wheel walks the hierarchy** — figure → shirt → fabric — and the
status bar shows where you are (`PART 2/4`). That is a correction mechanism, not
a substitute for getting it right: sometimes the honest answer to "is this wheel
part of that car, or an object in its own right" is that geometry cannot tell,
and you always can.

Motion Mode inherits all of it. Lock onto a shirt and it tracks **the whole
subject** — tracking the shirt would follow a region constantly occluded by arms
and reshaped by movement. Switch to `PART` first and the component becomes the
tracking target instead; `TEXTURE` has to demonstrate temporal coherence against
the real tracker before it is offered at all, and both refuse rather than
quietly hand back the whole object.

### Verification

```bash
npm run test:semantic
```

Every scene renders twice: once as the picture, once as a label map giving
pixel-exact ground truth for which thing each pixel belongs to. A probe asserts
by intersection against the real figure, not by eyeballing a box. Visual
regression images land in `out/semantic/` — scene, cursor, and the chosen mask
with everything else dimmed — including the refusals, because declining to
answer is a result and has to be inspectable too.

The suite checks the fixtures before it checks the engine: that every declared
part is actually visible in the ground truth, and that the figure's silhouette
really is harder than its internal boundaries. Both of those caught real bugs
that had been reported as algorithm failures.

**59 semantic checks pass, and the 145 pre-existing checks still pass**, plus
nine Motion checks that re-run the hierarchy over moving frames and hold target
identity through a crossing. The matrix
covers a person-like figure, an animal, a product with a component and a logo, a
vehicle, an object on a surface, two adjacent objects, a patterned object, a
textured object, and four negatives.

### What is honestly not there

- **The scenes are synthetic.** They are drawn to reproduce the one photometric
  fact the approach rests on — an object's silhouette is a depth discontinuity
  and is sharp, a material change inside an object is soft — and the suite
  asserts that property of the fixture itself before trusting any result. Real
  photographs are harder: soft focus, shadow edges as strong as object edges,
  and subjects that genuinely match their background.
- **The premise can be violated, and then the engine is wrong.** A bright blonde
  head against a dark shirt on a mid-grey wall has an internal boundary harder
  than half its own silhouette, and the head will read as an object in its own
  right. That is not a bug to be tuned away; it is the limit of what geometry
  can establish without knowing what a head is.
- **Low-contrast subjects.** When a subject's luminance matches its background
  the silhouette evidence is not in the image, and the honest output is no
  target rather than a guess.
- **Lens-versus-phone is a judgement call.** Both are enclosed entirely by
  another object. What separates them is how hard the shared contour is against
  the container's own silhouette — which settles the clear cases and reports low
  confidence in the rest. Hierarchy cycling exists because sometimes only the
  user knows.
- **No class names.** Labels are `OBJECT`, `PART`, `TEXTURE`, `REGION` — never
  "PERSON" or "CAR", because RPCTION cannot justify those words. When a real
  classifier is registered it may add one, and it will be attributed to the
  classifier that said it.
## Adversarial robustness

```bash
npm run test:adversarial
```

The semantic suite asks whether the engine gets the right answer at the point
chosen for it. This one asks the question the product lives on: **would a person
consider RPCTION to have understood what they pointed at — from wherever they
happened to point?**

So the unit of measurement is a *sweep*, not a probe. Fourteen named places on a
figure, a hundred-odd interior grid points, thirteen positions of cursor jitter.
A single well-chosen probe hides a model that only works in the middle of
things.

**75 of 91 adversarial checks pass**, identically on every run - the fixtures
draw their randomness from a seeded generator, because a suite that scored 66
one run and 64 the next was measuring its own noise. The sixteen that do not
pass are listed below with a named ROOT CAUSE, which is the thing that says
whether two failures are one problem or two.

The suite prints those root causes itself, asserts that progressive refinement
never degrades an answer it replaces, and writes an image for every hard case.

### What holds up

| | |
| --- | --- |
| **Cursor placement** | 35/35 named positions across a figure, an animal, a car and a phone. Hair, ear, knee, shoe, bumper, wheel — all give the whole object. |
| **Hover stability** | 447 interior grid points across three scenes, **one distinct answer each**. The highlight does not flicker as the cursor moves inside an object. |
| **Jitter** | ±6px around a point never changes the answer. |
| **Edges** | 1–10px inside gives the object; 2px outside gives nothing. Selection stops within 1px of the *rendered* silhouette. |
| **Scale** | 0.1% to 50% of the frame, all correct. Size is evidence, never a rule. |
| **Off-frame** | A figure running off the bottom is selected, and the mask claims only visible pixels. |
| **Similar instances** | Two identical phones, and only the one under the cursor. |
| **Hard internals** | A black screen in a white case on a grey desk gives the whole phone. |
| **Surfaces** | Marble, carpet, grass, brick, water and screen moiré all read as material, never as objects. |
| **Shadows** | The object is selected; the cast shadow is not an object. |
| **Motion identity** | Two similar figures crossing, with and without motion blur: identity is kept, or the target is declared lost. It never silently switches. |

Every hard case above - passing or failing - writes an image to
`out/adversarial/`, refusals included: a failure named in a log is a claim, a
failure you can look at is evidence.

### The three fixes this pass produced

**Progressive refinement (§18/§35).** A lawn, a desk, a photo panel in an app
window: every one of them is strongly bounded, uniform, cleanly exposed to the
background, and beats the object sitting on it on every measurement the engine
has. They are not wrong to score well — they *are* coherent regions. No
rebalancing fixes that, because from the outside a lawn genuinely looks more
like a well-formed object than the ball on it does.

What changes the answer is changing the question. When the winning candidate is
essentially the frame the cursor is standing in, the analysis re-runs with that
frame as its window — and the frame's own interior becomes reachable from the
border, is recognised as background, and the thing on it stands alone. The
refinement is kept only if it finds something smaller that scores at least as
well and has a hard outline of its own, so it can improve an answer and never
degrade one. **Four surface scenes and the image-in-image case, fixed.**

**Mode gates ask the mode's own question.** `TEXTURE` was gated on "is this
region best described as a texture", which deliberately subtracts the
candidate's silhouette — so asking TEXTURE mode about a patterned bag refused
outright, the bag's own outline cancelling its own pattern. The mode asked what
material this is; texturiness answers that. (The same category error as PART,
fixed the same way.)

**A hard cap on hierarchy depth (§27/§40).** Growth-ratio pruning thins the tree
but does not bound it, and refinement could push a scene to twelve levels.
Twelve is not a hierarchy anyone can cycle through. The least informative level —
the one closest in size to its own neighbours — is dropped until seven remain.

### Nine changes that were tried, measured, and rejected

Both are recorded in the code so they are not attempted again blind.

**Colour-opponent gradients.** Luminance is blind to iso-luminant boundaries: a
red object on green grass differs by ~30 luma units and by an enormous amount of
colour. Adding chroma to the gradient field was measured at four settings:

| | semantic | adversarial |
| --- | --- | --- |
| luma only | 59 | 45 |
| + chroma (0.55) | 58 | 43 |
| + chroma (0.30) | 59 | 44 |
| + chroma, smoothed | 59 | 42 |

It rescued one case and broke compressed images across the board — every codec
subsamples colour, so at full resolution the engine reads 8×8 chroma blocks
inside a flat shirt as contours, and a mid-quality JPEG of a figure went from
four correct probes out of five to zero. Smoothing the chroma first is worse
still: blur reduces a hard one-pixel silhouette far more than an already-soft
internal transition, which inverts the exact asymmetry the model depends on.
Doing this properly needs chroma denoised in a way that preserves step edges —
real work, not a coefficient.

**An edge threshold that adapts to the image.** The constant `30` is a statement
about a crisp screenshot, and degraded images do have weaker boundaries. Tying
it to the window's mean gradient was worse either way round: rising with the
mean, one high-contrast object lifts the gate and genuine boundaries elsewhere
stop counting (49 → 44); allowed only to fall, it traded one set of scenes for
another and still lost (49 → 47). The mean is the wrong statistic — it is
dominated by whatever is loudest in the window. A percentile of the gradient
distribution would be the right one.

**A transmissive signal for glass** — the share of the outline where surface
texture carries straight through. It measured ~0.01 everywhere *including on
glass*, because the candidate a pane actually produces is one flat stripe of the
wall behind it: uniform inside, uniform outside, no texture on either side to
match. The idea is right and the measurement was aimed at the wrong thing. What
a pane really produces is a **tile of a field that repeats well beyond it**, and
that is what would need measuring.

**A divided-by signal for occlusion** — a contour running clean across a
candidate and continuing past its bounding box, meant to catch a figure merged
with the table occluding it. It never fired on that case, because the merged
mass's bounding box *is* the table's extent, so there is nothing outside it for
the contour to continue into. It cost a working check elsewhere (66 → 65). The
real evidence is the reverse relation — the **figure's** contour terminating
against a longer contour that carries on past it — which is the actual
T-junction and needs contour tracing rather than row-and-column scanning.

Removing both after measuring took median latency from 52–73ms to **35–56ms**:
the rejected signals were not free.

### Occlusion and transparency: two signals built, measured, and not shipped

Both were built properly — real implementations, positive *and* negative
controls, distributions measured before anything was allowed to depend on them.
Both failed to separate. The measurements are kept as live checks rather than
comments, so if a future change makes either one usable, the suite says so.

**T-junctions for occlusion.** Where one contour terminates against another that
carries on, one surface is passing behind the other:

```
        │                      │
    ────┼────  three arms   ───┘   two arms
        │      = a T             = a corner
```

Implemented as a junction count on a small ring around each boundary point —
peaks on the ring are contours crossing it; three arms with two of them opposite
is a T. Measured two ways against a figure behind a table (positive) versus a
shirt meeting pants and two blocks in contact (negatives):

| | positive | negative |
| --- | --- | --- |
| as a share of the contact contour | 0.11 | 0.08 |
| as distinct junction events | 5 | 5 |

No separation either way, and **the reason is worth more than the number**: an
articulated object is full of T-junctions between its own parts. A shirt's edge
terminating against an arm's edge is a textbook T. The topology that marks "one
surface passes behind another" also marks "two parts of me meet". The detector
works; the inference from it does not. Separating those needs depth ordering or
figural completion, and neither is in these pixels.

**Field continuation for transparency.** The earlier texture-matching attempt
was rejected for measuring the wrong thing — a pane produces a *flat slice* of
the wall behind it, with nothing on either side to match. The better question:
slide the candidate across the scene and see whether it lands on itself again,
counting only displaced samples that leave the candidate entirely. A slice of
wall seen through glass does; a phone on a shirt does not.

It works on most of the set — glass over a banded wall 0.99, a transparent
bottle 1.00, while a phone, a shirt, a patterned bag and a block on brick all
read 0.00. Two cases break it:

- **Glass over a random texture reads 0.06.** A stippled surface does not repeat
  at any displacement, so there is no continuation to find even though the pane
  is plainly transparent. The method can only see fields that *repeat*.
- **One opaque scene reaches 0.97**, which leaves no margin at all.

A signal whose weakest positive sits below its strongest negative cannot be
thresholded without suppressing genuine objects, which §18 forbids outright. So
it is measured and used nowhere.

**The half that does work is pinned as a test**: in one scene, a pane over a
repeating wall reads 0.99 while an opaque box standing behind that same pane
reads 0.00. Same wall, same pane — the only difference is whether something
solid is in the way. That is the measurement a future attempt should build on,
probably by detecting *repeating fields* directly rather than asking each
candidate about itself.

Neither signal runs during hover. `junctionEvidence` and `fieldContinuation`
live in `contour.ts` and are called only from the diagnostic path, so the
perception loop pays nothing for them.

### Where the remaining sixteen could be solved

Several apparently promising static signals have now been shown to be **valid
measurements that are not discriminative**. That is not an implementation
failure — it is an information limit, and continuing to search for static
heuristics that separate cases whose visible pixels are structurally equivalent
would be wasted effort. So the current pass classified every remaining failure
by *what evidence would be needed*, rather than attacking them one at a time.

The hard question for each is: could a **human** pick the right target from this
single frame — and if so, is the evidence they used visual, or is it knowledge
of what the thing *is*? The second is the only place a future local classifier
would genuinely earn its keep.

| category | n | static | temporal | classifier | ambiguous |
| --- | --- | --- | --- | --- | --- |
| BOUNDARY_ABSENCE | 3 | no | **yes** | yes | no |
| CONTAINMENT_AMBIGUITY | 2 | no | maybe | **yes** | no |
| OCCLUSION | 2 | maybe | **yes** | no | no |
| TEXTURE | 2 | **yes** | no | no | no |
| REFLECTION | 1 | **yes** | maybe | no | no |
| TRANSPARENCY | 1 | maybe | no | no | no |
| DEGRADATION | 4 | **yes** | no | no | no |
| AMBIGUITY | 1 | maybe | no | no | maybe |

- **BOUNDARY_ABSENCE** — two figures whose arms touch at identical tone. There
  is *zero* contrast at the seam; nothing in the frame marks it. Independent
  trajectories would settle it immediately.
- **CONTAINMENT_AMBIGUITY** — a dark rectangle inside a lighter object. Measured
  directly: the held phone scores *worse* than a face-in-a-person on every
  available axis, so no threshold takes one without the other. This is the
  clearest case in the whole suite for a classifier.
- **OCCLUSION** — the evidence exists in principle (the figure continues above a
  contour that spans the scene) but two static formulations failed. Watching the
  figure walk out from behind the table resolves it outright.
- **TEXTURE** — red on green: the boundary is in chroma and nowhere else.
  Measured and rejected globally because codecs destroy chroma.
- **DEGRADATION** — chosen for this pass as the largest category with an
  untested hypothesis. See below.

That table is printed by the suite itself, and a check asserts it accounts for
all sixteen.

### Three more signals measured and rejected

**Coarse-scale analysis for degradation.** The hypothesis: at a coarse enough
scale a head cannot form its own region while the whole figure still can.
Swept the analysis resolution:

| analysis edge | adversarial |
| --- | --- |
| 430 (shipping) | 75 / 16 |
| 300 | 68 / 23 |
| 220 | 66 / 25 |

The degradation failures do not resolve at coarse scale — they *relocate*. At
220 the head probe passes and the shirt and arm probes start failing at recall
0.50, i.e. the figure splits in half instead. And small-object selection breaks.
Refuted, and worth noting that the previous pass had dismissed this by argument
rather than measurement; the argument reached the right answer for the wrong
reason.

**A percentile edge gate.** The recorded rejection of an adaptive threshold said
the *mean* was the wrong statistic and a percentile would be the right one. A
specific diagnosis justified re-opening it: under blur a figure's boundary
support collapses from 1.00 to 0.62 while the head inside it stays at 1.00,
which looks exactly like a fixed floor cutting off a softened silhouette.
Gating on the 98th percentile of the window's gradients at four fractions —
0.08, 0.12, 0.16, 0.20 — produced **75 passing checks at every one of them**,
identical to the constant, for the cost of a histogram pass.

So the gate is not what is binding. Under blur a figure's silhouette and its
internal boundaries soften *together*, and no threshold on a quantity they both
fell through can separate them again. Degradation is static-solvable in
principle but not by any thresholding of edge strength.

**T-junctions and field continuation** were measured in the previous pass and
are documented above; both remain implemented in `contour.ts`, called only from
the diagnostic path, with their distributions asserted as live checks so that a
future change making either usable would be noticed.

### Identity-preserving tracking

The tracker used to answer one question — *does something here correlate?* — and
report the answer as a lock. A figure walking behind a pillar exposed what that
costs: it followed the figure to the pillar, stopped there, and reported
`locked` for the rest of the sequence while the figure walked out the far side
unaccompanied. Not a lost target, which a user sees and corrects, but a
confident one pointing at the wrong thing.

**The root cause was circular verification.** `verify()` was meant to be the
identity check, and it short-circuited on `ncc(frame, adaptive, box) >= 0.35` —
the same quantity the search had just finished maximising, against a template
the tracker itself rewrites. Of course the winning box matches the template that
chose it. And because `adapt()` ran on any frame scoring above 0.65, each frame
of a gradual occlusion taught the model slightly more pillar until the identity
*was* the pillar.

**The fix is to ask two questions instead of one.**

| | |
| --- | --- |
| `confidence` | is there something here that correlates? |
| `identity` | is that something the target the user picked? |

Identity is measured against the **pristine template**, which nothing ever
writes to, so it cannot drift. A high answer to the first and a low answer to
the second is not a lock — it is `OCCLUDED`: the prediction is still plausible,
but whatever is standing at the predicted position is not mine. The appearance
model only learns from frames that cleared the identity check, so an occluder
can no longer poison it.

The sequence now reads:

```
18:325 tracked  0.95    figure
20:344 OCCLUDED 0.16    pillar rejected
22:357 OCCLUDED 0.20
24:428 tracked  0.95    figure recovered on the far side
final centre 498 vs truth 496
```

Reacquisition searches **ahead along the motion model** before sweeping the
frame, and demands more identity evidence than ordinary tracking does — after a
target has been invisible there is nothing to contradict a good-looking match,
which is exactly when a tracker adopts the wrong thing.

The overlay says which is happening: `TRACKING 94%` versus `OCCLUDED 16%`, and
an occluded box is drawn stalled rather than locked, because it is a prediction
and not an observation.

### Three things measured and corrected on the way

**A fixed identity floor does not survive real content.** At an absolute 0.30
the tracker's own unit suite fell from 23 passing to 2 — it was refusing genuine
targets, coasting, and latching onto whatever it drifted into. A low-contrast
subject legitimately matches its own template at 0.25 forever. The bar is now
*relative* to what this target has been holding, which is scale-free and
calibrates itself.

**Correlation alone is too weak a basis for identity.** Using raw pristine NCC
made the tracker travel zero pixels of a 468px path. Colour does real work in
this tracker; identity has to use it too. What matters is that the *template*
never drifts, not which metric reads it.

**A template cut around a small subject is mostly background.** After a figure
left the frame entirely, the pristine template still matched the empty gradient
at 0.95 correlation and the tracker called it tracked. What distinguishes a
subject from the field it was cut out of is that its colours are concentrated
where it is and absent just outside — so distinctness now carries most of the
weight, and background can no longer masquerade as the target.

### Two identical adjacent objects

Appearance cannot separate these at all, and a pristine template does not help —
both templates match both blocks equally well, because the blocks *are* equal.
One intermediate version appeared to fix it by anchoring its identity baseline
on the first *update* rather than on the selection frame; that is the same hole
that let an empty gradient pass as a tracked target, so it was not solving this
case, it was failing to reject anything.

What does separate them is trajectory — see
[Trajectory identity](#trajectory-identity-when-appearance-has-no-opinion).

### Trajectory identity: when appearance has no opinion

Two identical objects side by side are the one case appearance cannot help with
at all. The correlation surface offers two equal peaks, and a strict maximum
resolves that tie by **scan order** — so the leftmost peak always won, and both
trackers of an identical pair ended up on the same block in every scenario
measured.

The audit ran five motions over pixel-identical blocks and recorded the
assignment matrix per frame: for each pairing, how well does the prediction fit
the observation, and how much better is that than the swapped pairing?

| case | evidence | worst margin | swaps | final error |
| --- | --- | --- | --- | --- |
| separating | resolvable | 128px | 0 | 11px |
| accelerating | resolvable | 128px | 0 | 14px |
| stationary | resolvable | 128px | 0 | 0px |
| together | resolvable | 120px | **19** | 55px |
| crossing | **ambiguous at coincidence** | **16px** | 11 | 113px |

**The margin is the answer.** It stays at 120–128px in every case where the two
objects are anywhere apart, and collapses to 16px at the instant a crossing pair
coincides. Trajectory carries identity information appearance does not, it is
decisive by a factor of 7.5, and where it fails it fails *visibly* rather than
silently.

**Two expectations I had were wrong, and the data corrected them.** Stationary
and co-moving pairs are not ambiguous: their predicted positions stay 64px
apart, so the assignment is determined even though the blocks are
indistinguishable to look at. Identity does not require telling two things
apart by appearance — it requires knowing which one you were following.

### The mechanism, and one that was rejected first

A **smooth prior** on distance-from-prediction was the obvious thing and it does
fix the identical-pair case. It also drags every ordinary match slightly toward
the prediction, and on a flat correlation surface that is enough to add real
positional lag: the tracker's own accuracy suite dropped from **23 passing to
20 at a prior of only 0.01**, on scale-change and low-contrast targets. Measured
at 0.01, 0.02, 0.03 and 0.06 — all of them cost accuracy.

What is needed is not a bias but a **choice between rivals**, made only when
appearance genuinely cannot choose. The search now keeps its coarse samples,
finds the best peak that is spatially distinct from the winner, and if that
rival scores within a hair of it, the motion model breaks the tie. Appearance
still decides whenever it can; it simply no longer decides by accident when it
cannot. Tracker accuracy: **23 of 23, unchanged.**

### What this does and does not settle

Separating, accelerating and stationary identical pairs now hold their
identities — 0 swaps, 0–14px error, where before both trackers converged on one
block with 211–254px error.

Two things remained, and the next pass took both apart.

### Comparing two hypotheses at two different precisions

A rigidly co-moving identical pair — two blocks 4px apart, moving in lockstep,
never separating — was the case the audit said should be solvable and was not.
The trace is the whole story:

```
 f | obs1 obs2 |    A: best   (rival)   marg tie | trackA
 1 |  272  336 |    291 0.839 ( 336 0.875) -0.035 . |    291
```

The first thing to notice is that the tie-break was working exactly as designed
and was never reached. At frame 1 the tracker's own block scored **1.0000** and
the twin scored **0.8203** — a margin of 0.17 against a tie epsilon of 0.05.
Appearance looked certain.

It was not reporting appearance. It was reporting **the sampling grid**. The
coarse search steps in strides of 6px, and one block's true position happened to
fall exactly on a sample while the other's fell 2px off one. On speckle texture
2px costs 0.17 of correlation. Only the leader was ever walked to its true
optimum; the rival was judged where the grid happened to land. **Two hypotheses
were being compared at two different precisions**, and the difference between
them was mistaken for evidence.

Refining both was not enough on its own, because the scale axis aliases the same
way: the best-scoring sample near the missed block was a *shrunken* box that fit
inside the target better than a correctly-sized box sitting 2px off, and
refining around that scale locked the peak out of the hypothesis that would have
matched exactly. Refinement now sweeps every scale the coarse pass used.

With both peaks measured properly, they score **1.0000 and 1.0000** — a real
tie, which is what two identical blocks should always have produced — and the
motion model breaks it correctly.

### Colour plateaus, and why correlation has the last word

That still left one target being stolen, and the cause was a different kind of
false peak. The match score is the better of correlation and colour likelihood,
and colour is a low-frequency cue: across two identical brown blocks 4px apart
it reads nearly the same *everywhere between them*. The coarse surface was not
two peaks but a **plateau**, whose maximum sat wherever sampling noise put it —
including halfway between the two objects.

Scoring peaks by correlation alone fixes the co-moving case outright, and breaks
a ball crossing a busy background: that target has almost no internal structure,
and colour is the only thing tracking it at all. Tracker accuracy 23 → 19.

So colour stays, and correlation decides *within* a plateau. When two positions
score the same, the one whose structure actually matches is the real reading —
which is what the score's own comment had always claimed and what the search was
not acting on. `score()` computes correlation on every call anyway, so carrying
it alongside costs nothing.

| case | swaps before | swaps after | error before | error after |
| --- | --- | --- | --- | --- |
| separating | 0 | 0 | 11px | 1px |
| accelerating | 0 | 0 | 14px | 0px |
| stationary | 0 | 0 | 0px | 0px |
| **co-moving** | **19** | **0** | 55px | **1px** |
| crossing | 11 | 3 | 113px | **10px** |

### Identity is history, not position

The original bug decided identity by scan order, so a fix that merely relocates
that accident is not a fix. Three tests attack the property rather than the
symptom:

- The same scene tracked from the left block and from the right block. Nothing
  about the pixels differs between the two runs — only which one was selected.
  Both end on the block they started on.
- The same pair rendered at four different offsets, putting the peaks at
  different phases of the sampling grid and in a different order relative to the
  search centre. **Spread across all four: 0px.**
- Two blocks in different colours, to confirm the assignment layer stays out of
  the way when appearance has a real opinion. It does, and it declares no
  ambiguity.

### `IDENTITY_UNRESOLVED`

Tracking states are not grades of one confidence number. `OCCLUDED` means the
prediction is plausible but the thing standing there is not the target.
`IDENTITY_UNRESOLVED` means the target is in view, so is something
indistinguishable from it, and **nothing measurable says which is which** —
appearance did not separate the readings and neither did trajectory.

It is a state rather than a threshold on purpose. Picking the leftmost, or the
first found, would be inventing an identity.

A crossing pair produces exactly the sequence you would want:

```
tracked -> identity_unresolved -> reacquired -> tracked -> identity_unresolved -> reacquired -> tracked
```

The unresolved frames are the frames nearest coincidence — separations of 8, 24,
40px — while the pair is tracked normally out to 184px apart. The engine stops
claiming to know at the moment it stops knowing, and says so.

**One thing that sounded right and measured wrong.** Freezing the appearance
model during unresolved frames is the obviously principled choice, and it is
worse: a co-moving pair is faintly ambiguous on many frames, so the model stops
learning almost entirely and the pair went from 0 swaps and 1px of error to 5
swaps and 68px. The poisoning it would prevent is already prevented twice over —
the pristine identity template is never written at all, and adaptation is
already gated on identity holding up against it. When two readings are genuinely
tied on appearance, learning from either teaches the same thing, because that is
what tied on appearance means. The identity *baseline* is still frozen, since it
is the scale occlusion is judged against.

### When appearance and trajectory disagree

A deliberate conflict: the target's colour and texture swap with its
neighbour's between two frames, so appearance points at the far block and
trajectory at the near one. The engine follows **appearance**, at identity 1.00.

That is the right way round, and the assertion originally written here demanded
the opposite before the measurement existed. Trajectory one frame after lock is
nearly empty evidence — velocity is still zero, so the "prediction" is only
"where it was", and preferring it would mean preferring stillness over every
visible thing in the frame. Trajectory earns its say when appearance has none,
which is precisely the co-moving and crossing cases and not this one. No
weighting change ships from it.

The honest residual: a 64px jump against a zero prediction is reported as an
ordinary lock. Penalising unexplained jumps would be a weighting change with no
measurement behind it — every case in the suite it would touch is already
correct without it.

### Cost

Refining every competing observation is not free, so it is only paid when
something is actually competing. A single target measures **1.00 observations
per frame** — nothing beyond the leader clears the tracking threshold, and the
extra refinements never run. An ambiguous crossing measures 2.05.

| case | median | observations/frame |
| --- | --- | --- |
| single target | 16.5ms | 1.00 |
| two identical targets | 14.6ms | 1.11 |
| crossing (ambiguous) | 16.8ms | 2.05 |

Static perception is untouched by this pass; median hover latency is unchanged.

### Does time give RPCTION information a single frame cannot?

**Yes — and both categories that needed it are blocked on the same thing.**

The static audit marked BOUNDARY_ABSENCE and OCCLUSION as temporally solvable.
This pass tested that claim directly rather than assuming it, with positive and
negative controls, and integrated nothing.

**Separation drift works.** The naive version of this signal — "do these regions
move differently" — is useless, because a walking figure's arms move differently
from each other in every frame and a figure is one object. So the measurement is
whether their *separation grows* or *oscillates around a constant*:

| case | drift | swing | directedness |
| --- | --- | --- | --- |
| two blocks separating | **2.75** | 2.75 | **1.00** |
| two blocks travelling together | 0.29 | 0.39 | 0.76 |
| one figure, both arms swinging | 1.01 | **3.30** | **0.31** |

The articulated figure has the *largest* swing in the set and the lowest
directedness — its arms move enormously relative to each other and go nowhere.
Two objects coming apart drift 2.7× further with perfectly directed separation.
This is real information, and no single frame contains it.

**But it needs identity, and the case it was meant to fix denies identity.** Run
the same motion with the two blocks made *identical* — which is precisely the
BOUNDARY_ABSENCE case — and both trackers collapse onto the same block within
five frames (separation 64 → 5). The correlation surface has two equal peaks
sixty pixels apart and the tracker takes whichever it likes. Drift cannot be
measured between two identities that cannot be held apart.

**That blocker has since been removed.** Identical targets are now held apart
whenever their trajectories differ at all — including the hardest case, two
objects moving in perfect lockstep — and where the trajectories genuinely do not
differ, at the instant of coincidence, the engine reports
`IDENTITY_UNRESOLVED` rather than guessing. So BOUNDARY_ABSENCE is temporally
resolvable when the assignment of observations to identities is unique, and
honestly unresolved when it is not. See
[Comparing two hypotheses at two different precisions](#comparing-two-hypotheses-at-two-different-precisions).

**Occlusion failed for a related reason, and worse — and has since been fixed.**
A figure walking behind a pillar: the tracker followed it cleanly to the
occluder and then *stopped there*, sitting on the pillar for the rest of the
sequence while the figure walked out the far side unaccompanied — reporting
`locked` throughout. See
[Identity-preserving tracking](#identity-preserving-tracking) for the fix.

That is the expensive kind of wrong. Not a lost target, which a user sees and
corrects, but a confident one pointing at the wrong thing. The existing
"vanished target is reported lost" check passes because there the subject
dissolves into flat background with nothing for correlation to hold; a textured
occluder gives it something, and it holds that instead. Recorded as a named
defect with an assertion, so it cannot quietly persist.

Both findings point at the same missing piece: **the tracker follows regions,
not identities.** Fixing that means re-acquisition that searches ahead along the
motion model after a target disappears, and verification strict enough to reject
an occluder — a specified piece of work, not a heuristic. Until it exists,
neither category is temporally solvable in practice, and the audit now says so.

### Updated solvability

| category | n | static | temporal | classifier |
| --- | --- | --- | --- | --- |
| BOUNDARY_ABSENCE | 3 | no | **yes, when trajectory assignment is unique** | yes |
| CONTAINMENT_AMBIGUITY | 2 | no | maybe | **yes** |
| OCCLUSION | 2 | maybe | **yes, now** | no |
| TEXTURE | 2 | **yes** | no | no |
| REFLECTION | 1 | **yes** | maybe | no |
| TRANSPARENCY | 1 | maybe | no | no |
| DEGRADATION | 4 | yes in principle | no | no |
| AMBIGUITY | 1 | maybe | no | no |

### The sixteen failures, and why

- **Two figures touching, identical skin tone (3 checks).** *BOUNDARY ABSENCE.* Their arms meet and
  both are the same colour, so there is no contour between them — not a weak one,
  none. Geometry cannot separate what has no boundary. A real photograph almost
  always has a contact shadow there; this fixture deliberately does not.
- **A phone held in a hand; a keypad in a remote (2 checks).** *CONTACT / CONTAINMENT.* Both are dark,
  strongly bounded rectangles lying entirely inside a lighter object. That is
  also an exact description of a phone on a laptop, which must resolve the other
  way. Hierarchy cycling exists for precisely this.
- **A figure behind a table (2 checks).** *OCCLUSION.* They touch, so the background-complement
  generator returns them as one component, and the merged blob outscores the
  figure alone. The correct level *is* in the hierarchy — it is reachable by
  cycling — it just does not win.
- **A red block on marble, and on carpet (2 checks).** *TEXTURE.* Refinement did
  not fire; the block's luminance boundary against the veined marble and the
  mottled carpet is too weak to clear the guard. This is the chroma limitation
  above, in two scenes.
- **Reflection and glass (2 checks).** *REFLECTION / TRANSPARENCY.* A reflection has a genuine contour and
  scores 0.88 as an object; a glass pane's specular streak reads as a bounded
  strip. Both are real contours belonging to no real object, and distinguishing
  them needs evidence this engine does not collect.
- **A degraded figure probed at the head (4 checks).** *DEGRADATION.* Under JPEG, downscale or
  blur, the figure's silhouette softens faster than the head's own outline, so
  pointing at the head returns the head. Four of five probes still resolve the
  whole figure at mid-quality JPEG; the head is the one that breaks first.

Every one of these is a *wrong object* rather than a refusal, which is the
outcome that costs the most trust — so they are the right things to be listed
first in any future pass.
