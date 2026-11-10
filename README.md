# pca9685-nv

A driver for the PCA9685, a sixteen-channel, 12-bit PWM controller on an
I2C bus from NXP, specified in the data sheet "PCA9685: 16-channel,
12-bit PWM Fm+ I2C-bus LED controller", Rev. 4, 16 April 2015
([nxp.com/docs/en/data-sheet/PCA9685.pdf](https://www.nxp.com/docs/en/data-sheet/PCA9685.pdf)).
The driver implements the `Pwm` trait of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) for each
channel and adds a servo helper and an LED duty helper.

**Hardware verification: NOT VERIFIED.** No board on the bench carries
this chip, so the driver has run only against a double of the part, in the
host suite. The device program below builds for the nRF52840-DK, the
nRF54LM20-DK and the FRDM-MCXN947 and has not been flashed.

## The device program

`examples/servo_sweep.nv` drives a hobby servo on channel 0 and an LED on
channel 1 of a PCA9685 breakout. It sets the part to 50 Hz, the frame
rate of a servo, moves the pulse between 1000 and 2000 microseconds in
steps of 10 every 20 ms, and makes the LED's duty follow the pulse. At
each end of the sweep it prints the pulse over the board's UART:

```
end 2000
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use pca9685
use pca9685.{ Pca9685 }

@tier(embedded)
fn main() [hw]
    // The part answers at 0x40 on the board's first I2C bus.
    let pwm = Pca9685.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 },
                             dev_bus.wiring_i2c(0, 0x40))
    hal.timer.delay_ms(10)
    if pwm.begin_with(pca9685.config_at(50)) != 0
        hal.uart.write("pca9685: no answer\n")
        return
    var us = 1000
    loop
        // 1000 to 2000 microseconds is the usual range of a servo.
        let _ = pwm.servo_pulse_us(0, us)
        let _ = pwm.led_duty(1, us - 1000)
        us = us + 10
        if us > 2000
            us = 1000
        hal.timer.delay_ms(20)
```

The full program, with the sweep back and forth and the printed line, is
`examples/servo_sweep.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the boards below and for the emulated board, where it runs the
image and checks that the program reports that the part does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which the program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 12,680 B | 1,164 B | 2,660 B |
| `nrf54lm20-dk` | 13,524 B | 1,218 B | 2,644 B |
| `frdm-mcxn947` | 22,148 B | 6,480 B | 2,748 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables. The
arena, 65,536 B on the Nordic boards, is the runtime's heap and is
reserved whether or not the program uses it.

The two Nordic board packages answer for I2C as absent hardware, so the
build removes the code that follows the first bus read, and their rows are
the image without the driver's register paths. The novo-lang
repository tracks the missing Nordic I2C driver as a feature. The FRDM-MCXN947 board
package drives I2C, and its row is the image with the driver's code in it.
The driver's own code allocates nothing: every buffer is a fixed-size
`Vec` in the caller's frame.

### The effects

Every function of the driver that reaches the bus, the timer or a pin
declares `[hw]`, and nothing else. The driver performs no input or output
of its own and allocates nothing of its own. The bus functions of
devices-nv build one short frame per transaction inside a `system` block
and release it when the call returns. The module checks at every tier,
`embedded` included. The program above declares `[hw]` for the same
reason.

## What it is

The PCA9685 has one 12-bit counter, 0 to 4095, and sixteen outputs that
compare against it. A channel has two 12-bit registers, ON and OFF: its
output goes high when the counter reaches ON and low when it reaches OFF,
so ON sets the phase and OFF - ON the width of the pulse. A bit beside
each, the **full-on** bit of ON and the **full-off** bit of OFF, holds the
output high or low whatever the counters say, and full-off wins. All
channels share one frequency, set by an 8-bit prescale from the
oscillator: `round(osc / (4096 x freq)) - 1`, from 3 (1526 Hz at the
internal 25 MHz) to 255 (24 Hz). The part takes an external clock of up to
50 MHz on EXTCLK.

| Register | Address | Holds | Power-up |
|---|---|---|---|
| MODE1 | 0x00 | RESTART, EXTCLK, AI, SLEEP, SUB1 to SUB3, ALLCALL | 0x11 |
| MODE2 | 0x01 | INVRT, OCH, OUTDRV, OUTNE | 0x04 |
| SUBADR1 to 3 | 0x02 to 0x04 | the sub-addresses | 0xE2, 0xE4, 0xE8 |
| ALLCALLADR | 0x05 | the all-call address | 0xE0 |
| LED0 to LED15 | 0x06 to 0x45 | ON_L, ON_H, OFF_L, OFF_H per channel | OFF_H 0x10 |
| ALL_LED | 0xFA to 0xFD | the same four bytes, written to all sixteen | |
| PRE_SCALE | 0xFE | the prescale | 0x1E (200 Hz) |

## Install

```sh
novo pkg add pca9685-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `pca9685` | The computations (prescale, frequency, duty and pulse counts, servo angle), the register access over the trait `PcaLink`, the start, sleep, wake and restart sequences, the channel functions, the bus and pin framing, the types `Pca9685` and `Pca9685Link`, and the `Pwm` implementation |

## How to choose an entry point

A program on a board uses `Pca9685`, as the board's device table builds
it:

```novo norun:fragment
    devices
        pwm  Pca9685  from "pca9685-nv"  on i2c0  addr 0x40
```

A program that holds its own bus, or a test with a double of the part,
implements `PcaLink` over its bus and calls the module's functions, which
take the link: `pca9685.start(link, osc_hz, config)`,
`pca9685.channel_duty(link, ch, permille)`. The framing of the bus is
`i2c_read`, `i2c_write` and `i2c_call`, generic over `I2cBus`.

| Method of `Pca9685` | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` on the board's bus, on its internal 25 MHz oscillator; nothing is sent |
| `attach_clock(bus, pins, w, osc_hz)` | the same with an external clock of `osc_hz` |
| `begin()`, `begin_with(config)` | bring the part up (below); `begin_with` answers a code, 0 for success |
| `set_frequency_hz(hz)`, `frequency_hz()` | set the frequency of all channels; read it back from the prescale |
| `led_duty(ch, permille)` | one channel's duty in parts per thousand |
| `servo_pulse_us(ch, us)` | one channel's pulse in microseconds, at the prescale the part holds |
| `pulse(ch, on, width)` | a pulse from count `on`, `width` counts long |
| `all_off()` | every channel off |
| `sleep()`, `wake(restart)` | stop the oscillator; start it, and restart the channels that ran |
| `outputs(on)` | drive the OE pin, if the wiring has one (`w.en`) |
| `init`, `set_duty`, `enable`, `disable` | `Pwm`: see below |

The module's functions of the same names take `(link, ...)` and add
`channel_set`, `channel_read`, `channel_disable`, `channel_enable`,
`software_reset`, `set_allcall_address`, `set_subaddress`, `set_responses`
and the register reads. The computations that need no part are
`prescale_for`, `frequency_of`, `period_us_of`, `duty_counts`,
`pulse_counts` and `servo_us_of`.

`Pwm` is implemented per channel as follows:

| `Pwm` method | What the PCA9685 does |
|---|---|
| `init(channel, freq)` | checks the channel, 0 to 15, and sets the frequency of the whole part if its prescale differs; a second channel started at another frequency moves the first |
| `set_duty(channel, duty)` | the duty in parts per thousand, rounded to the nearest of 4096 counts, from count 0; 0 is the full-off bit and 1000 the full-on bit |
| `enable(channel)` | clears the full-off bit that `disable` set, so the channel resumes with the counts it kept |
| `disable(channel)` | sets the full-off bit and keeps the counts |

## The rules a user needs

1. `begin` or `begin_with` has to run before a channel is written. It
   turns register auto-increment on, which is off at power-up and after a
   software reset. A channel is four bytes in one transaction, and with
   auto-increment off they all land in one register.
2. The part has no identification register. `begin` answers whether the
   part acknowledges its address, and nothing more. The driver does not
   implement `Identifiable`.
3. The prescale can be written only while SLEEP is set. The part
   acknowledges a write while it is awake and ignores it. `start` and
   `set_frequency` read the prescale back and answer -2 if it did not
   take.
4. After SLEEP is cleared the oscillator takes 500 microseconds before
   the outputs are valid. The driver waits that long, through the board's
   timer, in `start`, `set_frequency` and `wake`. An external clock needs
   no wait.
5. The outputs stop while the part sleeps. A frequency change puts the
   part to sleep, so a servo loses its signal for about a millisecond.
6. Setting SLEEP over a channel that runs sets MODE1.RESTART. Writing 1 to
   RESTART once the oscillator is back resumes the channels from their
   registers and clears the bit. A write of MODE1 that carries back a
   RESTART it read as 1 would do that by accident, so every write of MODE1
   here masks RESTART out. `set_frequency` and `wake(true)` write it on
   purpose; `start` does with `keep_outputs`.
7. ON and OFF are never written with the same count. A duty of zero is the
   full-off bit, and a duty of 1000 the full-on bit.
8. With MODE2.OCH set, the outputs change on the acknowledge of the last
   of a channel's four registers, and a write of fewer than four does not
   take effect. The driver always writes the four together, `disable` and
   `enable` included.
9. `start` turns the all-call address off. The part answers 0x70 after
   power-up, which an I2C multiplexer may use as well. `set_responses`
   turns it, and the three sub-addresses, on again, and
   `set_allcall_address` and `set_subaddress` change their addresses.
10. `start` clears every channel to off, so a program that restarts does not
    inherit the pulses of the last run. `keep_outputs` leaves the registers
    as they are and restarts the channels that ran.
11. The OE pin, if the wiring names one in `w.en`, is held high while the
    part is configured and driven low at the end of `start`, so the outputs
    do not glitch. The pin is active low.
12. A servo pulse is counted at the prescale the part holds: a count is
    `(prescale + 1) / osc` seconds, 4.88 microseconds at 50 Hz. The driver
    reads the prescale before each `servo_pulse_us`, one extra transaction,
    and refuses a pulse as long as the period or longer.
13. All servos on `servo_pulse_us` rise at count 0, together, which draws
    the current of all of them at once. `pulse` with a different `on` for
    each channel staggers them; the part has the phase for it.
14. The software reset, `software_reset`, resets every PCA9685 on the bus.
    It is a write of 0x06 to address 0, which other parts answer as well.

## Running on a microcontroller

Every function of the module builds at the embedded tier. The driver's own
code builds no list: its bytes move through devices-nv's `DevBytes`, a
32-byte buffer held in the caller's frame. devices-nv's register access
allocates one frame per transaction and releases it before it returns,
because `I2cBus` takes and answers lists. The registry shows the package
on the host layer, the layer of the trait packages it depends on.

## What is not included

- Hardware identification. The part has none.
- The `Identifiable` trait, for the same reason.
- A per-channel frequency. The part has one for all sixteen channels, and
  `Pwm.init` sets it for all of them.
- A refusal from the `Pwm` methods. They answer nothing, so a failed write
  is not seen through the trait. The module's functions answer `Bool`.
- The fast-mode-plus clock above 400 kHz. The driver asks the bus for
  400 kHz; the part takes up to 1 MHz.
- The test mode register at 0xFF, which the data sheet marks reserved.
- A gamma curve for LEDs. The duty is linear; a program that wants
  perceived brightness applies its own curve before `led_duty`.
- Asynchronous operation. The part completes every write on the bus.

## Reference drivers read

Each of these was read for its fixes, and none was ported. The fix is
named at its site in `src/pca9685.nv`.

| Driver | What was taken from it |
|---|---|
| Zephyr `drivers/pwm/pwm_pca9685.c` | the four registers of a channel in one transaction with auto-increment on; the full-off bit for a zero duty and the full-on bit for a full one; the prescale set between a SLEEP write and a wake, with the pending RESTART written back at the end |
| Linux `drivers/pwm/pwm-pca9685.c` | the ALL-CALL and sub-address bits off at start, to avoid collisions; the ALL_LED registers set to full-off at start; OCH cleared; the 500 microsecond wait after SLEEP is cleared; the ALL_LED registers read as zero, so nothing reads them |
| Adafruit `Adafruit_PWMServoDriver` | the prescale rounded to the nearest, `round(osc / (freq x 4096)) - 1`, held to 3 to 255; RESTART masked out of the MODE1 write that sets SLEEP; auto-increment on in the final write |
| `pwm-pca9685` crate (eldruin) | the stored MODE1 does not keep RESTART, because writing it again restarts the part; the prescale below 3 is refused |

## Related packages

- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `Pwm`, `I2cBus`, `GpioOut` and `Timer` traits and the board handles.
- [devices-nv](https://novo-lang.org/packages/devices-nv) names the wiring
  record a board's device table builds and the register helpers this
  driver uses.

## Tests

`novo test tests` runs the module over a double of the part that answers
on an I2C bus. The double does what the data sheet says: auto-increment
moves the register pointer and rolls over after register 69, a write to
PRE_SCALE is ignored while SLEEP is clear, setting SLEEP over a running
channel sets RESTART, a write of 1 to RESTART clears it, EXTCLK stays set,
the ALL_LED registers write all sixteen channels and read as zero, the
part answers its address and, when enabled, the all-call address and the
sub-addresses, and the general call with byte 6 resets every register. It
refuses the n-th write or read on request, so every refusal path is run.

The suite checks the prescale against the data sheet's example (200 Hz at
25 MHz is 30) and against the values of the libraries read (50 Hz is 121),
across the whole range of 24 to 1526 Hz, the duty and pulse counts, the
wrap of the OFF count in the data sheet's two examples, the exact bytes of
every transaction of `start`, `set_frequency`, `sleep`, `wake` and the
channel functions, and each refusal. `bash tests/coverage.sh` merges the
suites' line coverage over `src/`.

## Licence

Apache-2.0.
