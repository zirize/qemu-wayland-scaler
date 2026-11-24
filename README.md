# QEMU on Wayland: mouse fixes and integer scaling

📄 Project page: **[QEMU on Wayland: mouse dead zones, slow Windows mouse, blurry scaling, and integer scaling](https://zirize.github.io/qemu-wayland-scaler/)** · More projects: **[zirize.github.io](https://zirize.github.io/)**

All three QEMU retro series (integer scaling, non-blocking LPT, CUE/BIN audio CDs) in one build: **[Building QEMU for retro guests](https://zirize.github.io/qemu-retro-patches/)**

Patches for QEMU that fix **relative mouse motion** and **blurry display output**
on a Wayland compositor (tested on KDE Plasma), add an **integer scaling**
option to the SDL display with a hotkey to switch it at run time, keep the
SDL window size when the guest changes resolution (**zoom-to-fit**), and make
the emulated PS/2 mouse honour the **sample rate the guest asked for**.

If you landed here from a search engine, one of these is probably your problem:

- The guest mouse pointer **drifts away from the host pointer**, or the guest
  cursor **stops at an invisible edge** inside the QEMU window (a "dead zone")
  — with `-display gtk` on Wayland/XWayland.
- Moving the mouse **slowly does nothing** and the guest pointer only starts
  moving once you move fast — it reads like a broken acceleration curve.
- The guest display is **blurry / bilinear-filtered** even though you are
  running at an exact 2x or 3x zoom, on a desktop using **fractional display
  scaling** (125%, 145%, 150%…).
- You want **pixel-perfect integer scaling** for an old low-resolution guest
  (Mac OS 9, Windows 98, DOS) and QEMU only offers a bilinear stretch.
- The mouse in a **Windows XP** (or other Windows) guest **crawls**, and the
  lower the guest resolution the worse it gets — while the same host mouse is
  fine in a Linux or Mac OS 9 guest.
- The `-display sdl` window **snaps back to a small 1:1 size** every time the
  guest changes resolution (BIOS → boot loader → desktop), throwing away the
  size you gave it.

There are nine patches. Six are bug fixes; three add to the SDL display — the
scaler option, a hotkey that cycles it, and zoom-to-fit.

## The problems, and what actually causes them

### 1. The GTK display cannot hold the pointer on Wayland

QEMU's GTK frontend emulates a relative-motion mouse: it grabs the pointer,
takes the difference between consecutive absolute positions, and warps the
pointer back to the middle when it gets close to an edge.

Two things go wrong on Wayland:

- `gdk_seat_grab()` has no `confine_to` argument, so the pointer is not
  actually confined to the window. Under XWayland, X11 pointer grabs do not
  deliver motion events once the pointer leaves the surface, so the guest
  cursor simply freezes — at a different place every time, because the motion
  is relative.
- The re-centering check in `gd_motion_event()` is written against the
  **monitor** geometry, not the widget. On a 3840x2160 screen with the guest
  in a window, it never fires. Instrumented here: 306 checks, 0 hits, 0 warps.

This is a known structural problem. The QEMU maintainer called the approach
["clunky at best"](https://lists.gnu.org/archive/html/qemu-devel/2022-01/msg05940.html)
and recommended using the low-level Wayland protocols instead. KDE filed the
same symptom as [bug 450818](https://bugs.kde.org/show_bug.cgi?id=450818).

**There is no patch for this here.** The fix is to use `-display sdl`, which
uses SDL2's native `pointer-constraints` and `relative-pointer` support and
works correctly. Patch 2 does fix a related real bug: `gd_grab_update()`
throws away the `GdkGrabStatus` and `gd_grab_pointer()` sets `ptr_owner`
regardless, so a failed grab leaves QEMU feeding motion to the guest while the
host pointer wanders over other windows.

### 2. The fractional part of scaled relative motion is thrown away

Both `ui/gtk.c` and `ui/sdl2.c` scale relative motion from window coordinates
into guest pixels and store the result in an `int`. When the window is larger
than the guest — zoom-to-fit, or any manual enlargement — that factor is below
one. A one-pixel movement of the mouse becomes zero, every single event.

The guest pointer then stands still while you move the mouse slowly, and falls
short of the distance travelled when you move it fast. Truncation is toward
zero, so left/up lose the fraction differently from right/down and the pointer
drifts. Patches 1 and 4 keep the remainder and fold it into the next event.

### 3. Fractional display scaling silently undoes your filter choice

QEMU's SDL window is created without `SDL_WINDOW_ALLOW_HIGHDPI`, so SDL only
ever draws it at its *logical* size. On a compositor running at a fractional
scale, the compositor then resamples that to physical pixels with its own
filter — which destroys whatever scaling mode QEMU picked.

On a display at 1.45x here, measured against the guest framebuffer: the window
content matches a **bilinear** upscale before the patch and a **nearest** one
after it (fraction of adjacent identical pixels: 88.9% → 97.5%). Patch 3 asks
for the real pixel density so the compositor has nothing left to resample.

Asking for it has a consequence: the drawable and the window are no longer the
same size, and the GL paths in `ui/sdl2-gl.c` set the viewport from the window
size. Since GL counts from the bottom left, that draws the guest small in that
corner. Patch 3 takes the drawable size there instead.

### 4. No integer scaling option

Patch 5 adds `scaler=linear|nearest|integer|integerplus` to the SDL display:

| mode | behaviour |
|---|---|
| `linear` | the existing bilinear stretch (default) |
| `nearest` | nearest-neighbour; sharp, but some guest pixels end up wider than others |
| `integer` | scale by the largest whole number that fits, letterbox the rest — every guest pixel the same size |
| `integerplus` | scale by a whole number into a render target with no interpolation, then stretch that to fill the window |

`integer` and `integerplus` tolerate the window falling up to two pixels short
of a whole multiple. That is not arbitrary slack: a fractional-scale compositor
rounds the surface to whole physical pixels, so a window sized to be exactly N
times the guest comes back a pixel short, and taking the `floor()` there drops
a whole step — 2x becomes 1x and half the window turns into border. Measured
here: a 3x request of 2400x1800 became 1655x1241 logical and came back as
2400x**1799**.

The option is defined on `DisplayOptions` rather than `DisplaySDL` so other
display backends can implement it too.

**The GL path (`gl=on`) does not implement the scaler.** It renders a plain
bilinear stretch over the whole window. Use the default 2D path for the scaler.

`scaler=` is read once at start-up, so comparing the modes on a guest used to
mean restarting it. Patch 8 lets the grab modifier plus **S** (Ctrl+Alt+S by
default) step through `linear → nearest → integer → integerplus` while the
guest runs. While the mode differs from the one QEMU was started with, the
window title names it. `S` is used by neither the SDL nor the GTK hotkeys, and
unlike F1–F12 or Backspace a guest has no common use for it together with
Ctrl+Alt. With `gl=on` the key is passed to the guest, since there is no scaler
to cycle.

The scaler option had also been added to the GTK display's usage line, where
nothing reads it. Patch 9 takes it out of the GTK help.

### 5. The PS/2 mouse ignores the sample rate the guest selected

A PS/2 mouse is sampled at a fixed rate and the guest picks that rate with
`AUX_SET_SAMPLE`. QEMU records the value and then ignores it: every host
pointer event is turned into a packet and sent straight away.

Host pointing devices report far faster than any PS/2 mouse — 1000Hz is common
on a gaming mouse, and on one here 84% of the events carry a total movement of
two counts or less. A guest that only looks at the deltas does not care, because
the counts still add up. A guest that derives the pointer *speed* from the counts
in a single packet does. Windows' "enhance pointer precision" is exactly that: a
guest that asked for 100 samples a second and instead gets 1000 packets of one
count each reads that as the slowest possible movement and keeps the pointer at
the bottom of its acceleration curve.

Patch 6 accumulates the motion and releases it at the interval the guest asked
for, which is what the hardware would have done. Button changes are not delayed,
a sample rate of zero still passes everything through, and remote (polled) mode
is untouched. Measured on a Windows XP guest: packets drop from ~1000/s to a
96/s ceiling, and the mean counts per packet go from ~1 to 7.55.

Mac OS 9 is unaffected because ADB is polled by the guest, so QEMU already
accumulates motion between polls.

### 6. The SDL window shrinks back on every guest mode change

Whenever the guest switches resolution, the SDL display calls
`sdl2_window_resize()` and sets the window to the new surface size, undoing
whatever size you gave it. With a scaler that fits the guest into the window
that is rarely wanted: a guest that boots through several modes ends up in a
small 1:1 window each time. No SDL option turned it off.

GTK and Cocoa already have `zoom-to-fit` for this, so patch 7 adds the same
option to the SDL display. When it is on, the new resolution is scaled into the
current window instead. Ctrl+Alt+0 still resizes the window to 1:1, since that
is an explicit request.

Measured with a boot sector that switches between text mode (720x400) and mode
13h (320x200) every three seconds, the window set to 1440x800 physical pixels:
with `zoom-to-fit=on` it stayed 1440x800 at the same position throughout; without
it, it shrank to 1x on every switch.

## Usage

```
qemu-system-ppc  -display sdl,scaler=integerplus  ...
qemu-system-i386 -display sdl,scaler=integer,zoom-to-fit=on  ...
```

Ctrl+Alt+S cycles the scaler while the guest runs.

## The patches

Both series are the same nine commits, `Signed-off-by`, each one building on
its own. `checkpatch.pl` reports no style problems; the one error it raises on
every commit is QEMU's policy against AI-assisted contributions (see Status).

- [`patches/qemu-master/`](patches/qemu-master/) — against QEMU master (11.1.50),
  with a cover letter. Branch:
  [`zirize/qemu@ui-scaler-and-motion-fixes`](https://github.com/zirize/qemu/tree/ui-scaler-and-motion-fixes)
- [`patches/qemu-screamer-10.2.0/`](patches/qemu-screamer-10.2.0/) — against
  [mcayland/qemu](https://github.com/mcayland/qemu) `screamer-v10.2.0` (the
  Mac OS 9 / PowerPC "screamer" audio fork rebased on QEMU 10.2.0). Branch:
  [`zirize/qemu@screamer-ui-scaler-and-motion-fixes`](https://github.com/zirize/qemu/tree/screamer-ui-scaler-and-motion-fixes)

The `zirize/qemu` branches also carry one unrelated commit each, which is not in
these patch series: a non-blocking parallel port on the master branch (see
[qemu-parallel-port-freeze](https://github.com/zirize/qemu-parallel-port-freeze))
and a `sungem` short-frame fix on the screamer branch.

```sh
git am /path/to/patches/qemu-master/00*.patch
```

The commits:

```
ui/gtk: keep the fraction when scaling relative motion
ui/gtk: do not take ownership of the pointer if the grab failed
ui/sdl: render at the output's pixel density
ui/sdl: do not scale relative pointer motion
ui: add a scaler option for the SDL display
hw/input/ps2: send packets at the sample rate the guest asked for
ui/sdl: add a zoom-to-fit option
ui/sdl: cycle the scaler with Ctrl+Alt+S
qemu-options: do not list scaler= for the GTK display
```

The PS/2 one is independent of the display work and touches `hw/input` rather
than `ui/`; it is kept in the same series only because it came out of chasing
the same "the mouse feels wrong" report.

## Tested on

Ubuntu 26.04, KDE Plasma on Wayland at 1.45 display scale, SDL 2.32.10,
with a `mac99` Mac OS 9.2.2 guest and `q35` Windows XP / Windows 98 SE guests.
Zoom-to-fit was measured on `qemu-system-i386` as above; on the screamer build
only start-up with the option was checked (the resize path is the same code).
The scaler hotkey was tried on the Windows 98 SE guest.

## Status

Not submitted upstream: QEMU does not accept contributions made with AI tools
(`docs/devel/code-provenance.rst`), and these were. These are offered as-is for anyone hitting the same
problems. The patches are against QEMU and carry QEMU's licence (GPLv2).
