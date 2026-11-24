# es8311-nv

A driver for Everest Semiconductor's ES8311, a low-power mono audio codec
with an I2C control port and an I2S or PCM audio port, specified in the
product brief "ES8311 Low Power Mono Audio CODEC", Revision 7.0, January
2020 (ES8311PB). The driver implements the `AudioCodec` and `Identifiable`
traits of [devices-nv](https://novo-lang.org/packages/devices-nv) and
configures the board's `I2s` handle of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) for the
format it sets.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
chip, and no board package has an I2S block yet. The driver has run only
against a double of the part and a double of the I2S handle, in the host
suite. The device program below builds for the nRF52840-DK and the
nRF54LM20-DK against a stand-in handle and has not been flashed.

**The register map is not checked against Everest's data sheet.** The
public document is the product brief, which gives the pins, the clock
modes, the I2C protocol and the serial formats and does not give the
registers. The registers, the clock coefficients and the power sequences
come from the drivers listed under Tests, and the places where those drivers
disagree are named in `src/es8311.nv` and below.

## Install

```sh
novo pkg add es8311-nv
```

The device program is `examples/tone.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/tone.nv` brings the codec up and plays a square wave of about 444
Hz at 16 kHz, printing one line per step over the board's UART:

```
codec found
```

```novo norun:needs-pkg
use embedded_hal.{ I2s, I2sConfig, BoardI2c }
use dev_audio.{ AudioCodec, AudioFormat, AudioFraming, CodecPath, MilliDb }
use dev_bus
use es8311

// The board's I2S block goes here; this stand-in refuses to be configured.
@value
struct NoI2s
    id: Int

impl I2s for NoI2s
    fn configure(self, c: I2sConfig) -> Bool [hw]
        false
    fn sample_rate(self) -> Int [hw]
        0
    fn enable(self) -> Bool [hw]
        false
    fn disable(self) [hw]
        pass
    fn transmit(self, var buf: Vec[i32; 256]) -> Int [hw]
        -1
    fn receive(self, var buf: Vec[i32; 256], n: Int) -> Int [hw]
        -1
    fn xruns(self) -> Int [hw]
        0

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The codec answers at 0x18 with CE low, on the first I2C bus.
    let part = es8311.attach(BoardI2c { id: 0 }, dev_bus.wiring_i2c(0, 0x18),
                             NoI2s { id: 0 }, es8311.config_default())
    if not part.begin()
        hal.uart.write("no answer from the codec at 0x18\n")
        return
    let f = AudioFormat { sample_rate: 16000, bit_depth: 16, channels: 2,
                          framing: AudioFraming.AudioPhilips, codec_is_master: false }
    if part.codec_set_format(f) and part.codec_set_power(true)
        let set = part.codec_set_gain(CodecPath.CodecOutput, MilliDb { mdb: -6000 })
        if set != None
            // After the first buffer is queued the output is opened.
            let _ = part.codec_set_mute(false)
```

The full program, which also fills and sends the buffers, is
`examples/tone.nv`. A board package that depends on this package builds it
as its `main`. `tests/device_build.sh` does so from a scratch package for
the two boards below and for the emulated board, where it runs the image and
checks that the program reports that the codec does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 20,176 B | 4,908 B | 3,236 B |
| `nrf54lm20-dk` | 21,624 B | 5,356 B | 3,236 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables. The
arena is the runtime's heap and is reserved whether or not the program uses
it.

**The I2S block is absent hardware on both boards; the I2C bus is not.**
embedded-hal-nv has a trait for an I2S interface and no board handle for
one, and neither Nordic board package drives its I2S block.  The Nordic DKs
drive the control port's I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA
P0.26 and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04),
on the pins `docs/hal/README.md` names under "The Nordic DKs' buses", so
both images carry the driver's reading path.  Before the Nordic buses landed
the two rows were 15,020 and 16,340 bytes of flash, of which the program was
2,750 and 3,070 bytes, because the board packages answered for I2C as absent
hardware and the build removed the code that follows the first bus read.  The
program is written against the `I2s` trait and is given `NoI2s`, so on both
images `begin` answers `false` and the program prints `no answer from the
codec at 0x18`.  A reply from the codec over a DK's bus is NOT VERIFIED until
the driver rig carries one.  The one argument of `attach` is what changes when
a board package gives a handle.

### The effects

Every function of the driver that reaches the bus or the I2S handle
declares `[hw]`, and nothing else. The driver performs no input or output of
its own. `attach` allocates one cell of the arena, because a struct with a
generic field cannot be a `@value`; the sequences build only the one-byte
frames of devices-nv's register functions, in the caller's frame. The module
checks at every tier, `embedded` included.

## What it is

The ES8311 has a 24-bit sigma-delta ADC with a low-noise preamplifier and
an analogue or PDM microphone input, a 24-bit DAC with a headphone driver
and a differential output, dynamic range compression, and an I2S or PCM
serial port that runs from 8 to 96 kHz. It takes a master clock of 256 or
384 times the sample rate, a USB clock of 12 or 24 MHz, or can take its bit
clock as the clock. The I2C address is 0x18, or 0x19 with CE high. The
chip ID is 0x83 at 0xFD and 0x11 at 0xFE.

The part is mono. A stereo frame carries the DAC's sample in one slot,
chosen by `dac_slot_set`, and the ADC's in both.

The codec makes everything from a digital clock of 256 times the sample
rate. The LRCK divider is 256 and the ADC and DAC dividers are 1. The clock
the board gives is brought to 256 times the rate by a divider of 1 to 8 and a
multiplier of 1, 2, 4 or 8, which `clock_plan` finds:

| Master clock | Rate | Divider | Multiplier |
|---|---|---|---|
| 2.048 MHz | 8 kHz | 1 | 1 |
| 12.288 MHz | 8 kHz | 6 | 1 |
| 4.096 MHz | 16 kHz | 1 | 1 |
| 12.288 MHz | 16 kHz | 3 | 1 |
| 12.288 MHz | 32 kHz | 3 | 2 |
| 11.2896 MHz | 44.1 kHz | 1 | 1 |
| 12.288 MHz | 48 kHz | 1 | 1 |
| 18.432 MHz | 48 kHz | 3 | 2 |
| 1.536 MHz | 48 kHz | 1 | 8 |

A clock that no divider and multiplier make into 256 times the rate is
refused. That includes a 12 MHz USB clock, which the brief lists, and an
18.432 MHz clock at 8 and 16 kHz, whose rows in the reference tables use an
ADC and DAC divider of 3 and a different LRCK divider.

With no master clock, `config_sclk()` takes the bit clock as the clock. The
bit clock is rate times bit depth times channels, so 16-bit stereo is 32
times the rate, multiplied by 8, and 32-bit stereo is 64 times, multiplied
by 4. A 24-bit stereo frame makes 48 times, and a 16-bit mono frame 16
times, and neither can be multiplied to 256.

## What the package contains

| Module | What is in it |
|---|---|
| `es8311` | `Es8311Config`, `Es8311Port` over devices-nv's `RegPort`, the I2C framing, the clock plan, the named steps, the gain functions, the type `Es8311` and its traits |

## How to choose an entry point

A program on a board builds an `Es8311Config` with `config_default()` (a
master clock of 256 times the rate) or `config_sclk()`, passes it with the
board's I2C bus, the wiring and the I2S handle to `attach`, and uses the
traits on the result.

| Method | What it does |
|---|---|
| `attach(bus, w, i2s, cfg)` | the codec at `w.addr` and the handle that carries its samples; nothing is sent |
| `begin()` | the bus ready, the chip ID read, the reset and the base state |
| `codec_set_format(f)` | `AudioCodec`: configures and enables the I2S handle, then sets the clock source and role, the dividers and the serial port |
| `codec_set_power(on)` | powers the converters up, with the DAC muted, or down and disables the handle |
| `codec_set_mute(on)`, `codec_is_muted()` | the DAC's two mute bits |
| `codec_set_gain(path, g)`, `codec_gain(path)` | the DAC volume or the input gain, to the largest step at or below `g` |
| `device_id()` | `Identifiable`: 0x8311 and the version register |
| `set_adc_volume(g)`, `adc_volume()` | the ADC's digital volume |
| `set_dmic(on)`, `set_headphone(on)` | the PDM input and the headphone driver |
| `set_dsp_format(depth, b)`, `set_dac_slot(right)` | DSP/PCM mode A or B, and the slot the DAC plays |
| `reg(r)`, `set_reg(r, v)`, `i2s_stop()` | one register, a write, and the handle's clocks stopped |

The steps are functions over `Es8311Port`, a trait a type holding an I2C bus
implements with devices-nv's `RegPort` (`read_regs` and `write_reg`; the
register reads and writes are `port.read_reg(reg)`, `port.write_reg(reg, v)` and
`port.update_reg(reg, mask, bits)`), so a test runs them over a double of the part: `step_reset`,
`step_open`, `step_source`, `step_clocks`, `step_serial_port`,
`step_power_up` and `step_power_down`. Their order is the one place to read
the sequences:

| Step | Writes |
|---|---|
| reset | RESET 0x1F, wait 20 ms, 0x00, 0x80 |
| base state | analogue power to 0xFA, the noise-immunity bit of GPIO 0x44, the master and bit clocks on, the dividers at their reset values, the system registers at minimum power, the headphone driver enabled, the ADC filters |
| source | RESET with the master bit, CLKMGR1 with the source and inversion |
| clocks | the divider and multiplier, the oversampling, the ADC and DAC dividers, the bit clock divider and inversion, the LRCK divider of 256 |
| serial port | word length, format and LRP of both ports |
| power up | the ADC and DAC clocks on, the ports unmuted, the DAC muted, the ADC volume at 0 dB, PGA and modulator, DAC, microphone input, analogue power |
| power down | the DAC muted, volumes to their lowest, PGA, modulator and DAC off, analogue power to its suspend word, the state machine held in reset, the clocks stopped |

## The gains

The DAC volume and the ADC volume are 0.5 dB steps from -95.5 dB at code 0
to +32 dB at code 255; code 191 is 0 dB. Code 0 is not silence: the output
is silenced by `codec_set_mute`. The input gain of `AudioCodec` is the
analogue gain: the PGA from 0 to 30 dB in 3 dB steps, then the ADC scale from
6 to 42 dB in 6 dB steps, so the total runs from 0 to 72 dB. The ADC digital
volume is separate, with `set_adc_volume`. The gain a call sets is never
above the request, and a request below the range sets nothing.

## The rules a user needs

1. Wait for the supply, then call `begin`. The part answers its ID only
   when it is powered and its I2C pins are wired.
2. Set the format before powering up. `codec_set_format` configures the I2S
   handle and enables it first, because the part needs its clocks running
   before its registers are final.
3. `codec_set_power(true)` leaves the DAC muted. Open it with
   `codec_set_mute(false)` once the first buffer is queued.
4. `codec_set_power(false)` loses the clock registers. Set the format
   again before the next power-up.
5. A master codec needs a bit clock that divides the digital clock by 1 to
   18. 24-bit slots make 48 times the rate and are refused as a master.
6. `AudioFraming` names the I2S and left-justified formats only. DSP mode is
   `set_dsp_format`. The brief lists a right-justified format too, and none
   of the reference drivers has a register code for it, so it is not offered.

## What is not included

- The DAC equaliser and the ADC equaliser; both are bypassed.
- The dynamic range compression of the DAC and the ALC and automute of the
  ADC.
- The ADC to DAC test path and the ADCDAT source select of the GPIO
  register, other than the noise-immunity bit the base state sets.
- The settling wait of `AudioCodecAsync.codec_power_up`: the reference
  drivers wait for none, and no timer-parking body is written.
- A master clock that the divider and multiplier cannot make, the
  right-justified format, and the bit clock dividers above 18, where the
  reference drivers disagree.
- The power-amplifier enable pin of a module: it is the board's GPIO.

## Where the traits fall short

- `AudioFraming` has two framings; the part also has DSP/PCM modes A and B.
  The driver's own `set_dsp_format` covers them.
- `AudioCodec` has no place for the codec's mono DAC slot, the microphone
  input choice, the headphone or line output, or a digital microphone.
  The driver's own functions cover them.
- `I2sConfig` has one `bit_depth`, which is both the slot width and the
  sample width, so a codec that takes 24-bit data in a 32-bit slot cannot say
  so; the package follows the bit depth as the slot.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `AudioCodec` and `Identifiable` traits this driver implements and the
  wiring record and register helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `I2s` traits and the board handles.
- [wm8960-nv](https://novo-lang.org/packages/wm8960-nv) drives a stereo
  codec with the same trait.
- [inmp441-nv](https://novo-lang.org/packages/inmp441-nv) reads an I2S
  microphone over the same handle.

## Tests

`novo test` runs the module over a double of the part, a register file on an
I2C bus whose log keeps every transaction, and a double of the I2S handle:
the framing of reads and writes, the chip ID, the clock plan at the table's
rows and at every refusal, the plan for each format, every step as an exact
register log (the reset, the base state with the retried write, the format
at 48, 8 and 32 kHz, as master, from SCLK and with inversion, the power-up
with the DAC muted before it is powered, and the power-down), the bits each
write keeps, the mute, both gains and the ADC volume at their steps and
limits, and the type over absent hardware. Every refusal of a write at each
position of the format and the power-up is checked. The suite covers every
line of `src/`.

**Hardware: NOT VERIFIED.** See the example section.

The sources read for this driver, for their facts and not ported, are the
product brief above, Espressif's `esp_codec_dev` ES8311 driver (the successor
of the esp-adf one) and its `es8311` component in esp-bsp, and the Linux
kernel's `sound/soc/codecs/es8311.c`. The fixes and differences kept are
named in `src/es8311.nv` where they are kept:

- The word-length field is written whole. esp_codec_dev only ORs the bits
  for 16 bits, which leaves a stale bit when a port goes from 32 bits to 16.
- The DAC is muted before it is powered, and stays muted until the program
  unmutes it. The esp-bsp component starts with the output open.
- The first write to the GPIO register after the reset is retried once, as
  esp_codec_dev notes the part drops it.
- The DAC oversampling register is 0x20 up to 16 kHz as in esp_codec_dev;
  esp-bsp and Linux write 0x10 throughout.
- The bit clock divider code is the divider less one up to 18. Above that
  esp_codec_dev and Linux give different codes for 19 and 20, and the driver
  refuses them.
- esp_codec_dev, without an MCLK pin, uses a fixed multiplier of 8 (4 at 8
  kHz), which is right only for a bit clock of 32 times the rate; the driver
  finds the multiplier from the bit clock of the format.

## Licence

Apache-2.0.
