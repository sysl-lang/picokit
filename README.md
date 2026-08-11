# picokit

The **Pico Breadboard Kit** for sysl — a 3.5" ST7796 panel, a joystick, two buttons, two LEDs, an
addressable RGB LED and a buzzer, on a carrier board that an original Raspberry Pi Pico plugs into.

This is a **program, not a package**. Nothing imports it. `sh.sysl.st7796`'s README says that a pin
map and its bring-up "is board glue and belongs in the program, not here … letting it in is how this
would quietly become a package for one board" — and this is the program it means.

```
sysl-lang/rp2040   the chip's registers, generated from Raspberry Pi's own SVD
sysl-lang/pico     the board's host: USB stdio, the clock, the wireless chip, through the C SDK
sysl-lang/st7796   the panel: arithmetic on bytes and two callbacks this program supplies
sysl-lang/plutovg  the rasterizer, once the panel is up — see "What is not done yet"
```

## The board

52Pi sell it as **EP-0172**; GeeekPi sell the identical board under their own name. Both
[wikis](https://wiki.52pi.com/index.php?title=EP-0172) give the same pin map, and it is the only
documentation there is — the wiring behind the pins is not written down anywhere, which matters in
one place below.

| what | where |
|---|---|
| TFT — 3.5", 320x480, ST7796 over SPI0 | CLK `GP2`, DIN `GP3`, CS `GP5`, DC `GP6`, RST `GP7` |
| capacitive touch, over I2C0 | SDA `GP8`, SCL `GP9` |
| joystick | X `ADC0` (GP26), Y `ADC1` (GP27) |
| buttons | BTN1 `GP15`, BTN2 `GP14` |
| LEDs | D1 `GP16`, D2 `GP17` — D3 and D4 are wired to 3V3 and 5V and are not pins |
| RGB LED | `GP12` — one pin for a colour, so addressable rather than three channels |
| buzzer | `GP13` |

**The panel's controller is an ST7796 "may vary with production lot"**, which is 52Pi's own wording.
That is the one specification here worth confirming against the board in hand rather than trusting:
an ILI9488 has the same pinout, the same resolution and a different command set, and the failure is a
blank panel rather than a refusal.

## Building it

The pico-sdk owns the link — it generates a second-stage bootloader, selects a board header and
drives the final link with its own linker script, none of which fits inside something `sysl build`
could fetch. So the arrangement is the other way up: `sysl build-c` compiles this program into an
archive, and CMake links it.

```
export PICO_SDK_PATH=/path/to/pico-sdk
cmake -B build
cmake --build build
```

`build/picokit.uf2` is what goes on the board. Hold BOOTSEL, plug it in, copy the file across.

Everything else resolves on its own: all four dependencies are coordinates in
`picokit/package.hocon`, so this builds from a fresh clone.

**It is `thumbv6m-freestanding`.** The RP2040's Cortex-M0+ is Armv6-M, which is neither of the
targets a Pico 2 uses, and getting it wrong is not a link error — an Armv8-M image on an M0+ faults
at whatever executes first. `CMakeLists.txt` sets it, because `sysl build-c` is a program CMake knows
nothing about and nothing derives the target for us.

## What it does today

Alternates the two LEDs and reports the buttons over the USB serial port. That is small on purpose:
it proves the output path, the input path and the pads separately, before anything harder rests on
them.

The GPIO work is registers rather than `hardware/gpio.h` — `sh.sysl.pico` binds the parts of the SDK
that have no register equivalent, and the pins are not among them. A pin is three writes, they are
the writes the datasheet describes, and going through `sh.sysl.rp2040` keeps them readable against
it.

**One assumption in `main.sysl` wants checking against the board.** The buttons are configured with
the chip's own pull-ups, on the reading that they switch to ground; 52Pi document the pin numbers and
not the wiring. If they are to 3V3 instead, it is `PDE` rather than `PUE` and `pressed` loses its
`!`.

## A plain Pico

`sh.sysl.pico`'s `init` starts stdio and then the CYW43, and its `led` is a wireless-chip GPIO
because that is where a Pico W's LED is. So this program links `pico_cyw43_arch_none` and **wants a
Pico W today**. A plain Pico does not fail at CMake — it fails at the link, on `cyw43_arch_init`.

Nothing the *kit* does needs any of it: every part above is an RP2040 pin. The change is to reach
`sh.sysl.pico.externs` for `stdio_init_all` directly and never call `init`, which costs the package's
pleasanter surface for the one call that is Pico-W-only. It is not made here because which board is
under the kit is a fact about somebody's desk.

## What is not done yet

- **The panel.** SPI0 bring-up, and the two `*extern` callbacks `st7796.panel` takes — one that
  raises DC for a command, one that pushes pixels. The driver is written and tagged; what is missing
  is this board's half.
- **plutovg.** The two packages were built to pair: plutovg renders an ARGB32 band, st7796 converts
  it to RGB565 as it pushes, and the conversion lives in the driver so that plutovg stays a faithful
  binding. A 320x480 ARGB32 frame is 614 KB against this chip's 264 KB, so banding is not an
  optimization here — it is the only way it fits.

  **Its `requires { posix = true }` is not the obstacle it looks like.** Exactly two bound names in
  that package reach a file system — `Surface.write_png`, which is stb_image_write and `fopen`s, and
  the `font_face_load_from_file` extern — and a capability is a property of the module rather than of
  the call, so the whole package declares what those two need. Nothing on the drawing path touches
  an operating system.

  Probed rather than assumed: a program depending on plutovg and built with
  `sysl build-c --target thumbv6m-freestanding` gets past analysis, which is where a capability
  refusal would land, and stops in plutovg's C for want of the toolchain's own headers — the thing
  this project's CMake already supplies.
- **Touch**, over I2C0. The controller is not named in 52Pi's documentation.
- **The joystick**, which is the ADC and two pins.
- **The RGB LED**, which is one pin and a bit-banged or PIO waveform rather than a level.

## Licence

ISC.
