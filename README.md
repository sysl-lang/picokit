# picokit

The **Pico Breadboard Kit** for sysl — a 3.5" ST7796 panel, a joystick, two buttons, two LEDs, an
addressable RGB LED and a buzzer, on a carrier board that an original Raspberry Pi Pico plugs into.

This is a **program, not a package**. Nothing imports it. `sh.sysl.st7796`'s README says that a pin
map and its bring-up "is board glue and belongs in the program, not here … letting it in is how this
would quietly become a package for one board" — and this is the program it means.

```
sysl-lang/rp2040blocks  the chip's GPIO and SPI, over a register map generated from Raspberry Pi's SVD
sysl-lang/pico          the board's host: USB stdio, the clock, the wireless chip, through the C SDK
sysl-lang/st7796        the panel: arithmetic on bytes and three callbacks this program supplies
sysl-lang/plutovg       the rasterizer, once the panel is up — see "What is not done yet"
```

**`sysl-lang/rp2040blocks` came out of this repository.** The SPI bring-up and the GPIO helpers were
written here and ran here first, then moved — a PL022 reset cycle and a bit-rate divisor search are
facts about the chip, and would have been copied into the next board's program unchanged. What is
left here is pin numbers, which is the one thing that is genuinely about this carrier board.

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
`picokit/sysl.hocon`, so this builds from a fresh clone.

**It is `thumbv6m-freestanding`.** The RP2040's Cortex-M0+ is Armv6-M, which is neither of the
targets a Pico 2 uses, and getting it wrong is not a link error — an Armv8-M image on an M0+ faults
at whatever executes first. `CMakeLists.txt` sets it, because `sysl build-c` is a program CMake knows
nothing about and nothing derives the target for us.

## What it does today

Alternates the two LEDs and reports the buttons over the USB serial port. That is small on purpose:
it proves the output path, the input path and the pads separately, before anything harder rests on
them.

**Run on the board, 2026-08-10**, on a Pico W: the banner arrives over USB, D1 and D2 alternate, and
both buttons report themselves independently.

```
the Pico Breadboard Kit, in sysl
BTN1 down
BTN1 up
BTN2 down
BTN2 up
```

The GPIO work is registers rather than `hardware/gpio.h` — `sh.sysl.pico` binds the parts of the SDK
that have no register equivalent, and the pins are not among them. A pin is three writes, they are
the writes the datasheet describes, and going through `sh.sysl.rp2040` keeps them readable against
it.

**The buttons switch to ground**, which the board in hand confirms and 52Pi do not document either
way. So they get the chip's own pull-ups and `pressed` is `read` inverted. Had they gone to 3V3 it
would be `PDE` rather than `PUE`, and the symptom would have been a button reporting itself held
while nobody touched it.

## A plain Pico

`sh.sysl.pico`'s `init` starts stdio and then the CYW43, and its `led` is a wireless-chip GPIO
because that is where a Pico W's LED is. So this program links `pico_cyw43_arch_none` and **wants a
Pico W today**, which is what it has been run on. A plain Pico does not fail at CMake — it fails at
the link, on `cyw43_arch_init`.

Nothing the *kit* does needs any of it: every part above is an RP2040 pin. The change is to reach
`sh.sysl.pico.externs` for `stdio_init_all` directly and never call `init`, which costs the package's
pleasanter surface for the one call that is Pico-W-only. It is not made here because which board is
under the kit is a fact about somebody's desk.

## The panel

**Run on the board, 2026-08-10, and it took two goes.** The first was right in every respect except
polarity — bars across the top in the correct order, square below them, correct rotation, correct
channel order, and every colour its own complement: red reading cyan, green magenta, blue yellow, and
a black background coming up white.

**This glass wants display inversion on.** The ST7796's init sequence sends neither `INVON` nor
`INVOFF`, so the controller is left at its post-reset default, and this panel is wired the other way
up — common on the IPS glass these carrier boards use. One command fixes it, and **BTN1 toggles it**,
which is how the claim was settled: the two builds print identically and differ only on the glass, so
a rebuild could not tell them apart and a button press could.

That also answers what 52Pi's *"may vary with production lot"* left open. The full init sequence —
the `CSCON`-gated extension registers and both gamma curves included — produced a coherent picture,
so **this is an ST7796** and not the ILI9488 that shares its pinout.

**`CS`, `DC` and `RST` are ordinary GPIOs rather than the SPI block's**, which is the one wiring
decision this program makes rather than inherits. The PL022 has a chip-select of its own and it is
useless here: it drops between frames, and this panel needs one assertion held across a command and
every parameter that follows it.

Everything else about the bus — the reset cycle, the divisors, the balanced push loop, and why
`drain` waits on `BSY` rather than on an empty FIFO — is `sh.sysl.rp2040blocks` and documented there.

The demo clears to black and paints three bars and a white square, chosen so that a wrong colour
order, a wrong rotation and a wrong window look like different mistakes rather than like each other.

## What is not done yet

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
