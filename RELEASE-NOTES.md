# RPCTION 0.1.1

Windows 10/11, macOS and Linux, all 64-bit. One real bug fixed, one interaction
model corrected, text recognition made meaningfully more exact, and the first
builds for macOS and Linux.

RPCTION is a local visual interaction layer for your screen. Press a hotkey,
point at something you can see — a URL inside a screenshot, a phone number in a
chat, an object in a photograph, something moving in a video — and copy,
extract, track or act on it.

**Everything runs on your device.** No account, no API key, no subscription, no
cloud inference, no telemetry. After installation RPCTION never needs the
network, and this build contains no code that would contact it.

---

## What changed since 0.1.0

**Activating RPCTION no longer makes a playing video disappear.** This was a
real bug and it is the main reason this release exists. Opening RPCTION over
YouTube left the page and the player on screen, the sound playing, and the
picture gone. The cause was not the overlay covering anything: it was the
browser underneath deciding it was hidden and switching its compositor off, so
what stayed on the glass was the last frame it drew — a frame that still had the
page in it and no longer had the video. RPCTION is now very slightly
transparent, which is what tells Windows it is not an opaque window, and the
browser keeps drawing.

**Your desktop stays live and stays clickable.** Previously the overlay took
every click on the display for as long as it was open, so the machine felt
frozen even though nothing was. Now RPCTION observes by default and the desktop
keeps receiving your mouse: scroll the page, press play, drag a window. Hold
**Ctrl** when you want RPCTION to take the click instead. The status line always
says which of the two it is doing.

**Text recognition is more exact where exactness matters.** A URL, an email
address, a phone number, a version string and a commit hash are now read with
the character set each of those can actually contain, so a hash does not come
back as English words and a filename keeps its underscores. Pasting the wrong
character is worse than pasting nothing, and this is aimed squarely at that.

**The first result arrives about seven times sooner.** Text under the cursor is
now read from a wide, short band rather than a large square, which is the shape
text actually occupies. First actionable target went from roughly 2.5 seconds to
roughly 0.3.

**LIVE and FREEZE.** LIVE is the default and is what the product is for: the
screen keeps moving and RPCTION re-reads it when you have been away. FREEZE pins
what RPCTION is reasoning about to the moment you opened it, for a screen that
will not hold still. It freezes only RPCTION's own view — never your desktop,
and never Motion, whose whole subject is change.

---

## Installing

**Nothing here is code-signed.** That is a deliberate choice for this release
rather than an oversight: certificates cost money and take time, and holding a
working build back for them helps nobody. Every operating system reacts to an
unsigned application, each in its own way, and all three are described plainly
below. **We are not going to tell you to switch a security feature off.**

The SHA-256 of every file is published, so you can check that what you
downloaded is what was built.

### Windows

Run `RPCTION-Setup-0.1.1.exe`. Per-user, no administrator rights. Installing
over 0.1.0 keeps your hotkey and your modes. A **portable** build is also
available: one executable, installs nothing.

Windows has two separate protections and they behave very differently.

**SmartScreen** shows "Windows protected your PC". Choose **More info** →
**Run anyway** and it continues. That is a warning, not a virus detection.

**Smart App Control** refuses unsigned applications outright. There is no "Run
anyway", and downloading again or waiting does not help — it judges on signing,
not on reputation. It is on by default on many clean Windows 11 installs. Check
under **Windows Security → App & browser control → Smart App Control**. **If it
is on, this build will not run on your machine.** Switching it off cannot be
undone without resetting Windows, so the honest answer is to wait for a signed
build rather than to weaken your system for one tool.

### macOS

Open `RPCTION-0.1.1-macOS-arm64.dmg` on Apple Silicon, or the `-x64` one on
Intel. Drag RPCTION to Applications.

**RPCTION is not notarized, so Gatekeeper may warn or refuse it.** A plain
double-click will usually be refused; right-click the app and choose **Open**,
then **Open** again. macOS will ask for **Screen Recording** permission the
first time you activate — RPCTION needs it to see what you are pointing at, and
what it sees stays on your machine.

macOS cannot give an application the system audio mix without a driver RPCTION
will not install, so AUDIO mode reports CAPTURE UNAVAILABLE there rather than
recording silence.

### Linux

Download `RPCTION-0.1.1-Linux-x86_64.AppImage`, make it executable, run it.
Nothing is installed and nothing else is required — no Node, no npm, no
package manager.

**The AppImage is unsigned.** That is normal for the format and no desktop will
block it for that reason.

An AppImage runs on most modern x86_64 distributions, and how much of RPCTION
works depends on your session rather than your distribution. Under **X11**
everything is available. Under **Wayland** the compositor owns global
shortcuts, so RPCTION's hotkey will not fire — bind it in your desktop settings
instead — and screen capture goes through the desktop portal, which asks
permission each session. Linux has no equivalent of the capture-exclusion flag
Windows and macOS provide, so RPCTION's own overlay can appear in Motion
exports.

---

## What is in it

**Thirteen modes**, cycled with Tab: AUTO, TEXT, OBJECT, PART, TEXTURE, IMAGE,
LINK, CONTACT, TABLE, ACT, MOTION, SOURCE, AUDIO. One hotkey activates RPCTION;
Tab changes what it is looking for.

**Local recognition.** Text, URLs, email addresses, phone numbers, addresses,
tables and QR codes, all on device via bundled Tesseract and a local QR decoder.

**Object selection without a classifier.** RPCTION has no object model and never
invents a category name. It works structurally: whether a region is bounded more
strongly on the outside than it is divided on the inside, whether its outline is
supported the whole way round, whether something continues it. That is enough to
tell a whole object from one of its parts, and it is honest about what it does
not know.

**Object extraction** to a transparent PNG.

**Motion Mode.** Select something moving and follow it; export as video, GIF, PNG
sequence, or a transparent extraction. This is not screen recording — the buffer
only exists while Motion Mode is open.

**SOURCE.** Point at code and RPCTION reads it back into text, rebuilding the
indentation from where the characters sit on screen. Every result states where it
came from: text rebuilt from pixels is labelled RECONSTRUCTED, never passed off
as an original file, and anything RPCTION cannot legitimately read says
UNAVAILABLE rather than fabricating plausible code.

**AUDIO.** Captures the last 2, 5, 10 or 30 seconds of what your machine is
playing, as a .wav. The rolling buffer only exists while AUDIO mode is open.

**ACT** is user-directed throughout. RPCTION never clicks anything on its own.

---

## Known limitations

These are stated rather than hidden, because a tool asking for screen access has
to be straight about what it cannot do.

- **No object classifier.** RPCTION does not know what things *are*. Measured
  over 1,574 probes across 23 scenes, it returns the right object 93.5% of the
  time. The failures are specific and named below rather than averaged away.
- **Objects on textured ground.** A figure standing on grass, gravel or carpet
  is the weakest case by a distance: the boundary dissolves into the texture and
  RPCTION selects a part of the figure instead of the figure. Use `↑` to widen
  the selection, or PART mode if a component is what you wanted.
- **Shapes that are mostly holes** — a bicycle, a wire frame — select poorly.
- **The selection is always shown before you act on it**, and `↑` / `↓` cycles up
  and down the hierarchy to correct it. That is the intended remedy, not an
  afterthought.
- **English only** for text recognition.
- **SOURCE cannot reach real source files.** RPCTION reads pixels; nothing in
  this build can report SOURCE AVAILABLE.
- **AUDIO captures the whole mix.** Windows combines system audio before
  applications can see it, so per-application isolation is not offered.
- **No transcription.** Capture is the feature; there is no local speech engine
  in this build and no cloud one is being added.
- **Windows, macOS and Linux builds exist; none has been through a full manual
  test.** Every artifact is built on its own operating system in CI, and each is
  automatically checked - mounted or extracted, metadata read, icon compared,
  and started once to confirm it does not fall over. That is a smoke test. It is
  not the same as somebody sitting down and using it, and that has not happened
  for 0.1.1 on any platform.
- **Nothing is code-signed or notarized.** Deliberate for this release. See
  the platform notes above for what each system will do about it.
- **No automatic updates.** Which is also why it never contacts the network.
- **442 MB of memory while running**, across five processes, measured on the
  packaged Windows build twenty seconds after launch while idle. **321 MB
  installed.** Most of both is the Electron runtime and the bundled
  text-recognition model.

---

## Verifying your download

Nothing here is signed, so the checksum is how you confirm that the file you
have is the file that was built. `SHA256SUMS` ships beside the binaries.

Windows, in PowerShell:

```
Get-FileHash .\RPCTION-Setup-0.1.1.exe -Algorithm SHA256
```

macOS:

```
shasum -a 256 RPCTION-0.1.1-macOS-arm64.dmg
```

Linux, against the published list:

```
sha256sum -c SHA256SUMS
```

<!-- BEGIN GENERATED CHECKSUMS -->
| file | size | SHA-256 |
| --- | --- | --- |
| `RPCTION-Setup-0.1.1.exe` | 91.8 MB | `45a06e1c2737d9eba5300c3b766187cdcd623212e4085705fff036a740474558` |
| `RPCTION-Portable-0.1.1.exe` | 91.6 MB | `1e548476869c8f6ae270e3aeeb97ad5f8bccf5dabeeb1ee80afdf81f75346c90` |
<!-- END GENERATED CHECKSUMS -->

---

## Uninstalling

On **Windows**: Settings › Apps › Installed apps › RPCTION › Uninstall, or the
uninstaller in the install folder. It removes the application, both shortcuts
and the registry entry. On **macOS**, drag RPCTION out of Applications. On
**Linux**, delete the AppImage.

Your settings are deliberately left behind so a reinstall keeps your hotkey and
modes. Delete the file below for a completely clean removal.

| platform | settings |
| --- | --- |
| Windows | `%APPDATA%\RPCTION\settings.json` |
| macOS | `~/Library/Application Support/RPCTION/settings.json` |
| Linux | `~/.config/RPCTION/settings.json` |

---

## Verified for this release

- **808 automated checks, 792 passing.** The 16 failures are all in the
  adversarial perception suite, are individually named and categorised in the
  README, and none is a release blocker. Counted from a single complete run of
  the full suite at the commit this release was built from, not carried over
  from a previous one.
- **Every package is opened and started by the machine that built it.** CI
  mounts the disk images, extracts the AppImage, compares each icon against the
  one canonical source, and launches the result from a temporary directory —
  which is what demonstrates that nothing resolves out of a source tree. A
  launch only counts if the application reaches its own startup report; a
  process that merely survives a timer does not pass, and neither does one
  killed by anything other than the harness.
- The video-visibility fix is measured against a real browser playing a real
  video, and the test asserts the bug still reproduces without it.
- Object perception measured over 1,574 probes: **93.5% identity, 92.4% mask
  IoU**. Those two are load-independent and came out identical on every run.
- **34 ms median, 67 ms p95** to resolve a target, measured on an otherwise idle
  machine. The same suite on a machine at ~70% load from other applications
  reads 73 ms median and 369 ms p95, so treat the first pair as the figure and
  the second as what a busy desktop costs.
- Text recognition: 6 of 10 paste-critical fixtures exact, up from 5.
- **No network connections** from the packaged application, verified at the
  socket level.
- The signing state of every artifact is **read off the bytes**, not assumed
  from whether signing was attempted. The Windows builds are confirmed
  unsigned, the macOS bundles confirmed neither signed nor notarized, and the
  AppImage unsigned — reported as such rather than quietly rounded up.
- **Only the architectures actually started are counted as started.** The
  runner that builds the macOS disk images is Apple Silicon, so the Intel build
  is checked but not launched, and nothing here claims otherwise.

**Not verified for this release, and not claimed:** the manual checklist in
`CHECKLIST.md` needs a person, a real desktop and real applications. It has not
been walked for 0.1.1.
