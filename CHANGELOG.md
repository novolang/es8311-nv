# Changelog

## 0.2.1 — 2026-11-27

The README example calls `part.codec_set_mute` without a binding.  `examples/tone.nv` keeps `let _ =` before the two `push` calls that fill the buffer and says in a comment that the frames fit it.  No signature of the package changed.

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Es8311Port` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Es8311Port` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Es8311Port` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Es8311Port for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `reg_read`, `reg_write`, `reg_update`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Es8311Port` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 15,632 bytes before and 15,552 after on the read-out (nrf52-qemu: 10,440 and 10,280).

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.3) and devices-nv ^0.4.0 (was ^0.3.3), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed.  The enums and the record of devices-nv 0.4.0 give the codec three things to answer for:

- `i2s_config_of` passes the format's `slot_bits` to the board's `I2sConfig`; a format of 0 is a slot as wide as the sample, as before.
- `fmt_code` answers -1 for the framings the format register has no code for, `AudioRightJustified`, `AudioDspA`, `AudioDspB` and `AudioPdm`, and `plan_of` and `codec_set_format` refuse them.  DSP mode stays `set_dsp_format`.
- `gain_set` and `gain_get` answer `None` for the paths the part does not have: only `CodecOutput` and `CodecInput` are its paths.

`examples/tone.nv` and the suite's format helper add `slot_bits: 0` to their `AudioFormat`.

The device program builds for the boards its README names, flash nrf52840-dk 15,764 B before and 15,676 B after; nrf54lm20-dk 17,212 B before and 17,140 B after; nrf52-qemu 10,444 B before and 10,388 B after.

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
  20,176 B of flash and for the nRF54LM20-DK 21,624 B; before the Nordic buses
  landed, with the reading path left out, it was 15,020 B and 16,340 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an ES8311, and no
  board package has an I2S block. The Nordic DKs drive I2C with TWIM0 and
  TWIM22, so the device program's images carry the reading path; a reply from
  the codec over either bus is NOT VERIFIED. The device program is given a
  stand-in handle.
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
