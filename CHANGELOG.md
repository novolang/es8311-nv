# Changelog

## 0.1.0

The first release: a driver for Everest Semiconductor's ES8311 mono audio
codec on an I2C control bus and a board's I2S handle.

- `Es8311`, the type `attach` builds from the board's I2C bus, the wiring and
  the I2S handle, implementing devices-nv's `AudioCodec` and `Identifiable`,
  with its own methods for the ADC volume, the digital microphone, the
  headphone driver, DSP/PCM mode, the DAC's slot and register access.
- The chip ID, 0x83 and 0x11, as `who` 0x8311 with the version register as
  `rev`.
- The clock manager: `clock_plan` finds the divider of 1 to 8 and the
  multiplier of 1, 2, 4 or 8 that bring a master clock, or the bit clock when
  the part has no master clock, to 256 times the rate, for 8 to 96 kHz, with
  the oversampling registers and the bit clock divider. Tests pin it at 8,
  16, 44.1 and 48 kHz and at the rows of the reference tables.
- The serial port in I2S and left-justified formats through `AudioCodec`, and
  DSP/PCM mode A and B through `set_dsp_format`; 16, 24 and 32 bits.
  `codec_set_format` configures and enables the board's I2S handle first.
- The DAC volume and the input gain in thousandths of a decibel, never above
  the request; the ADC volume; the DAC's two mute bits.
- The reset, the base state, the power-up and the power-down as named
  steps (`step_reset`, `step_open`, `step_source`, `step_clocks`,
  `step_serial_port`, `step_power_up`, `step_power_down`). A power-up leaves
  the DAC muted, as `AudioCodec` requires.
- The suite runs over a double of the part and of the I2S handle and covers
  every line of `src/`.
- The device program `examples/tone.nv`. Built for the nRF52840-DK it is
  15,020 B of flash and for the nRF54LM20-DK 16,340 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an ES8311, and no
  board package has an I2S block or, on the Nordic boards, an I2C driver. The
  device program is given a stand-in handle and finds an absent bus.
- The register map is from the reference drivers, not from Everest's data
  sheet, which could not be fetched: only the product brief, Revision 7.0,
  January 2020, is public. The reference drivers disagree on the DAC
  oversampling register and on the bit clock divider code above 18; the
  choices are named in `src/es8311.nv`.
- The manifest names devices-nv and embedded-hal-nv by the registry range
  `^0.3.3`, the first releases with `AudioCodec` and `I2s`.
- Not implemented: the equalisers, the DAC's dynamic range compression, the
  ALC and automute, a master clock the divider and multiplier cannot make
  (12 MHz, and 18.432 MHz at 8 and 16 kHz), the right-justified format,
  bit clock dividers above 18, and `AudioCodecAsync`.
- A local named `codec` breaks the embedded tier check
  (`a-local-named-like-a-stdlib-module-makes-the-embedded-tier-check-read-its-method-calls-as-module-calls`);
  the device program calls its codec `part`.
- The linter's `char-literal` rule reads register numbers such as 0x30 as
  characters. It is wrong for a register map, and those sites are left.
