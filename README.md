# DisplayServices

`/System/Library/PrivateFrameworks/DisplayServices.framework` (private): brightness of built-in and external displays, ambient light, display notifications. ControlCenter links it (`otool -L`), together with `CoreBrightness` (Night Shift, True Tone), `MonitorPanel` and `CoreDisplay`.

**Status: planned. No code yet.**

## Layer
Layer 3 (UI and windowing side: it talks to the display hardware and to the window server). Depends on: `CoreFoundation`, `CoreGraphics` (display ids), `IOKit`, `SkyLight` (soft dimming through the window server). Used by: ControlCenter (Display slider), the keyboard brightness keys (HIToolbox / `d`), System Settings.

## API the Control Center uses
Called through `dlsym` in `apps/controlcenter-cocoa/CCControls.m` (prototypes from the common SPI knowledge, **not yet verified on the Mac VM: it has no display hardware that answers**):
- `int DisplayServicesGetBrightness(CGDirectDisplayID display, float *brightness)` (0..1, returns 0 on success)
- `int DisplayServicesSetBrightness(CGDirectDisplayID display, float brightness)`
Also expected (names to confirm with REA on the framework): `DisplayServicesCanChangeBrightness`, `DisplayServicesRegisterForBrightnessChangeNotifications`, `DisplayServicesGetLinearBrightness`, `DisplayServicesBrightnessChanged`.
Open: the real panel's Display slider and its page (Dark Mode, Night Shift, True Tone buttons): what each one calls (trace the real ControlCenter with `research/cc-trace/dark.d`-style scripts).

## Plan (FreeBSD backend)
1. Backends in order: `backlight(8)` (KMS drivers, `/dev/backlight/*`), the ACPI video sysctl `hw.acpi.video.lcd0.brightness` (percent), the vendor drivers (e.g. `clevo_wmi` for the keyboard), and when there is no hardware control the window server dims the picture: `SLS_MSG_SetDisplayTransferByFormula` (already used by the earlier Control Center, `SoftDim`, SkyLight repository).
2. A small daemon is not needed: the library calls the backends directly (root helper if `backlight(8)` needs it).
3. Notifications: a change by the keys or by another process is posted so the slider follows (kqueue on the device where possible, else polling).
4. Match Apple: call order and values observed on the Mac (dtrace of the real Control Center and of the F1/F2 keys), REA of the framework for the return codes.

## Existing code to reuse
`apps/controlcenter-cocoa/CCPlatform.c` (`ccp_brightness_*`, `ccp_kbd_*`), `apps/topbar/ControlCenter.m` (`HwBrightness`, `SoftDim`).

## Rules for this repository
- Our code: BSD-2-Clause. Apple files are never copied here: reference material is read from the user's own Mac (`macdeps`), notes and measured values are written in our own words (`docs/`).
- No GPL/LGPL code. Upstream code, if any, only under `upstream/` with a `PROVENANCE.md`.
- Before writing a function: search the code index and `where-is.sh` (rule 3); do exactly what Apple does, with evidence recorded here (rule 4).
- Test on the FreeBSD VM; CI is the shared `hygiene` workflow (`pearOS-bsd/.github`).
