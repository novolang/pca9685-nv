# Changelog

## 0.1.0

The first release: a driver for NXP's PCA9685 sixteen-channel, 12-bit PWM
controller on an I2C bus.

- `Pca9685`, the type a board's device table builds, implementing
  embedded-hal-nv's `Pwm` per channel, with its own methods for the
  frequency, an LED's duty, a servo's pulse in microseconds, a pulse with a
  phase, sleep and wake, and the OE pin.
- The prescale from the oscillator and the wanted frequency by the data
  sheet's formula, rounded to the nearest and held to 3 to 255, for the
  internal 25 MHz oscillator and for an external clock up to 50 MHz, and
  the frequency and the period a prescale gives.
- The per-channel ON and OFF counts with the full-on and full-off bits, the
  wrap of the OFF count past the end of the period, and the ALL_LED
  registers for all sixteen channels at once.
- The start sequence: auto-increment on, the external clock switched in
  while asleep, MODE2 (output driver, invert, change on acknowledge,
  OUTNE), the prescale checked by a read, every output off, the 500
  microsecond oscillator wait, and the OE pin driven low last.
- The sleep and restart sequences, with RESTART masked out of every write
  of MODE1 that is not a restart.
- The all-call address and the three sub-addresses as configuration, and
  the software reset.
- The suite runs over a double of the part and covers every line of
  `src/`.
- The device program `examples/servo_sweep.nv`, a servo and an LED on a
  breakout. Built for the nRF52840-DK it is 20,096 B of flash, for the
  nRF54LM20-DK 21,072 B and for the FRDM-MCXN947 22,280 B; before the Nordic
  buses landed, with the register paths left out, the two DK images were
  12,680 B and 13,524 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries a PCA9685.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, which
  drive I2C with TWIM0 and TWIM22, so their images carry the register paths.
  A reply from the part over either bus is NOT VERIFIED.
- The `Pwm` trait of embedded-hal-nv 0.3.2 has no place for what the part
  does beyond a duty: its methods answer nothing, so a refusal is not seen
  through the trait; `init` takes a frequency for one channel where the
  part has one for all sixteen; and the duty is in parts per thousand where
  the part counts 4096. The module's functions answer `Bool` and counts.
- Not implemented: `Identifiable`, because the part has no identification
  register; the test mode register; a bus clock above 400 kHz.
