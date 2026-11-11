# ESPHome on TI SimpleLink: `cc27xx`, `cc23xx` and `cc35xx`

These ESPHome external components are offered as-is, with no promised support.

Three target platforms that let `esphome compile` and `esphome upload` build and flash a TI LaunchPad development kit with TI's own compiler and SDK:

| Platform | Board | Radio | What ESPHome sees |
|---|---|---|---|
| `cc27xx` | LP-EM-CC2755P10 (Cortex-M33) | Zigbee | router or end device, joins Home Assistant's Zigbee integration (ZHA) |
| `cc23xx` | LP-EM-CC2340R5 (Cortex-M0+, 36 kB RAM) | Zigbee | end device (a router fits only with a smaller heap) |
| `cc35xx` | LP-EM-CC35X1 (CC3551E, Wi-Fi 6) | Wi-Fi | native API and mDNS |

It was built and run against ESPHome 2026.9.1 on macOS (Intel). Other versions and other host systems are untested. The CC2755P10 and CC2340R5 platforms are the better-tested ones; the CC35X1 platform is an earlier, smaller proof (see "Known limits").

## Install

1. ESPHome in a Python 3.12 virtual environment: `pip install "cbor2<6" esphome==2026.9.1` (the `cbor2<6` pin matters on an Intel Mac, where cbor2 6 has no wheel).
2. TI's tools, from ti.com (a free myTI login is needed for some). Versions built with:

   | For | Tool | Version |
   |---|---|---|
   | `cc27xx`, `cc23xx` | SimpleLink Low Power F3 SDK | 9.21.00.36 LTS |
   | all | TI Arm Clang (`ti-cgt-armllvm`, comes with CCStudio 21.0.1) | 5.1.1.LTS |
   | `cc27xx`, `cc23xx` | SysConfig | 1.26.3 or 1.27.0 |
   | `cc27xx`, `cc23xx` | UniFlash (its `dslite.sh` flashes) | 9.6.0 |
   | `cc35xx` | SimpleLink Wi-Fi SDK | 10.20.00.39 |
   | `cc35xx` | SysConfig | 1.28.0 |
   | `cc35xx` | SimpleLink Wi-Fi Toolbox (signs and programs) | 4.3.23 |

   The platforms look under `/Applications/ti`, `~/ti` and `/opt/ti` (override with `ESPHOME_TI_ROOT`, or per tool with `ESPHOME_TI_F3_SDK`, `ESPHOME_TI_CGT`, `ESPHOME_TI_SYSCONFIG`, `ESPHOME_TI_DSLITE`, `ESPHOME_TI_WIFI_SDK`, `ESPHOME_TI_TOOLBOX`, or the keys `sdk_path:`, `compiler_path:`, `sysconfig_path:`, `dslite_path:`, `toolbox_path:` in the platform block). A missing tool stops the build and names it.
3. The LaunchPad on an XDS110 debug probe (LP-XDS110 or LP-XDS110ET). `xdsdfu -e` lists the probe's serial number; put it in `probe_serial:` or pass `--device <serial>` to `upload`.

## Use

```yaml
esphome:
  name: my-device

external_components:
  - source: github://Fox-Red-Labs/esphome-ti
    components: [cc27xx, die_temperature, ti_adc, ti_pwm]

cc27xx:
  board: LP_EM_CC2755P10
  probe_serial: XXXXXXXX

zigbee:
  router: true

logger:

switch:
  - platform: gpio
    name: "LED"
    pin: DIO16
```

`esphome compile my-device.yaml`, then `esphome upload my-device.yaml`, then `esphome logs my-device.yaml --device /dev/cu.usbmodem<probe serial>1`. The `cc23xx` platform needs `cc27xx` listed beside it (the two share their code). Complete files are in [`examples/`](examples/).

## What works

Everything below was run on the boards named. "Seen" means a log line or register value from the board; nothing here is a calibrated measurement.

| Feature | CC2755P10 (`cc27xx`) | CC2340R5 (`cc23xx`) |
|---|---|---|
| Build and flash | `esphome compile` 35 s cold, 27 s warm; `esphome upload` 15 s | 31 s cold, 22 s warm; upload 15 s |
| Zigbee join (ZHA) | joined 5.7 s and 6.3 s after two flashes; LED switch 32 of 32 commands, median 85 to 92 ms | end device: 93 of 100 commands over five runs (20 of 20 in the last) |
| Logger (`logger:`, `esphome logs`) | works, 115200 baud on the probe's first serial port | works |
| Pins | every DIO the chip has, see the tables below; 19 DIOs read 1 with a pull-up and 0 with a pull-down and followed an output in a self test | same table, same API (not self-tested on all pins) |
| Saved settings (`preferences:`, `restore_mode`, `globals: restore_value`) | a counter kept over 3 software resets (1, 2, 3, 4) | a counter kept over 3 software resets (1, 2, 3, 4) |
| I2C bus (`i2c:`) with ESPHome's own sensor components | compiles | bus scan and the `hdc2080` and `opt3001` components start; no sensor reading checked yet (the sensors board was not on the LaunchPad during the test) |
| ADC (`ti_adc` sensor) | reads a pin through TI's driver (1.0 to 1.8 V on open pins); not checked against a known voltage | compiles |
| PWM (`ti_pwm` output, `light: monochromatic`) | 25 % and 75 % duty seen in the timer's compare register against a 1 kHz period | compiles and runs; duty not read back |
| SPI bus, UART bus | not done | not done |

Image sizes (linker map): CC2755P10 router with logger, GPIO and settings 281.9 kB flash and 39.7 kB RAM of 165.9 kB; with ADC and PWM 290.2 kB and 40.7 kB. CC2340R5 end device with logger, I2C, PWM and settings 252.3 kB flash and 35.6 kB RAM of 36.9 kB (1.2 kB left at link time, plus whatever the heap has free at run time); use `logger: tx_buffer_size: 256` on this chip.

## Pins

Pins are written as the DIO number (`DIO16`, or `16`). The tables list every DIO of each chip, the BoosterPack header pin it reaches, what the LaunchPad already uses it for, and the signals the chip can route to it (so `i2c:` and `ti_pwm` can check your choice). `SWD` and `32 kHz crystal pad` pins are refused; a pin that is shared with the board (LED, button, backchannel UART, SPI flash) is accepted, with a warning for the last two. Source: TI's board description and SysConfig device data shipped with the F3 SDK, cross-checked against the LP-EM-CC2340R5 quick start guide and the CC2755x10 data sheet. `tools/gen_pins.py` regenerates `components/cc27xx/pindata.json` from them.

A TI driver takes its pin at build time (a SysConfig instance), so `i2c:`, `ti_adc` and `ti_pwm` pins are fixed in the image; GPIO pins can change at run time. A pin given to a driver is taken away from TI's LED and button code, which is why DIO14 can be a PWM output on the CC2340R5.

### LP-EM-CC2340R5 (cc23xx)

| DIO | Header pin | Already used on the board for | Analog in | PWM (timer, channel) | Other signals |
|---|---|---|---|---|---|
| 0 | 10 | free | ADC5 | LGPT3 ch2 | I2C0SDA SPI0CSN |
| 1 | 25 | free | ADC4 | LGPT3 ch1 | UART0RTS |
| 2 | 26 | free | ADC3 |  | UART0CTS |
| 3 | not on the header | 32 kHz crystal pad |  | LGPT1 ch2, LGPT3 ch1 |  |
| 4 | not on the header | 32 kHz crystal pad |  | LGPT0 ch2 | SPI0MOSI UART0TXD |
| 5 | 27 | free | ADC2 | LGPT2 ch2 |  |
| 6 | 38 | SPI flash chip select | ADC1 | LGPT1 ch2 | I2C0SCL SPI0CSN UART0TXD |
| 7 | 28 | free | ADC0 | LGPT3 ch1 |  |
| 8 | 36 | free |  |  | I2C0SDA SPI0SCLK UART0RTS |
| 9 | 12 | button BTN-2 |  | LGPT3 ch0 |  |
| 10 | 13 | button BTN-1 |  |  |  |
| 11 | 18 | BoosterPack SPI chip select |  | LGPT0 ch0 | SPI0CSN SPI0MISO |
| 12 | 14 | SPI flash and BoosterPack SPI data in |  | LGPT1 ch1 | I2C0SDA SPI0MISO SPI0MOSI UART0RXD |
| 13 | 15 | SPI flash and BoosterPack SPI data out |  |  | SPI0MISO SPI0MOSI UART0TXD |
| 14 | 39 | red LED |  | LGPT3 ch2 |  |
| 15 | 40 | green LED |  |  | UART0RXD |
| 16 | not on the header | SWD (debug) |  | LGPT1 ch2 | I2C0SDA SPI0MOSI UART0RXD |
| 17 | not on the header | SWD (debug) |  | LGPT0 ch2 | I2C0SCL SPI0SCLK UART0TXD |
| 18 | 7 | SPI flash and BoosterPack SPI clock |  | LGPT3 ch0 | SPI0SCLK UART0TXD |
| 19 | 17 | free |  | LGPT3 ch1 | SPI0MOSI |
| 20 | 4 | backchannel UART: chip TX | ADC11 | LGPT1 ch0 | SPI0MISO UART0RXD UART0TXD |
| 21 | 37 | free | ADC10 | LGPT0 ch1 | SPI0MISO UART0CTS |
| 22 | 3 | backchannel UART: chip RX | ADC9 | LGPT2 ch0 | UART0RXD |
| 23 | 23 | free | ADC8 | LGPT2 ch1 |  |
| 24 | 9 | free | ADC7 | LGPT1 ch0, LGPT3 ch0 | I2C0SCL SPI0SCLK |
| 25 | 24 | free | ADC6 |  | I2C0SCL SPI0MISO |

### LP-EM-CC2755P10 (cc27xx)

| DIO | Header pin | Already used on the board for | Analog in | PWM (timer, channel) | Other signals |
|---|---|---|---|---|---|
| 0 | 40 | button BTN-2 |  | LGPT0 ch0, LGPT1 ch0 |  |
| 1 | 4 | backchannel UART: chip TX |  | LGPT1 ch0, LGPT1 ch1, LGPT2 ch0 | CAN0TX UART0TXD |
| 2 | 3 | backchannel UART: chip RX |  | LGPT1 ch1, LGPT1 ch2 | CAN0RX UART0RXD |
| 3 | 7 | SPI flash and BoosterPack SPI clock |  | LGPT2 ch0 | I2S0SCLK SPI0SCLK UART1TXD |
| 4 | 14 | SPI flash and BoosterPack SPI data in |  | LGPT1 ch2, LGPT2 ch1 | SPI0MISO SPI0MOSI UART1RXD |
| 5 | 15 | SPI flash and BoosterPack SPI data out |  | LGPT2 ch1, LGPT2 ch2 | SPI0MISO SPI0MOSI |
| 7 | 18 | BoosterPack SPI chip select |  | LGPT2 ch2 | I2S0WS SPI0CSN |
| 9 | 34 | SWD (debug) |  | LGPT0 ch1 | I2C0SCL I2S0SD0 |
| 10 | 33 | SWD (debug) |  | LGPT0 ch2 | I2C0SDA I2S0SD1 |
| 11 | 39 | free |  | LGPT3 ch0 | SPI1MISO SPI1MOSI |
| 12 | 19 | green LED |  | LGPT3 ch1, LGPT3 ch2 | SPI1MISO SPI1MOSI |
| 15 | 35 | free |  | LGPT3 ch1, LGPT3 ch2 | SPI1SCLK |
| 16 | 36 | red LED |  | LGPT3 ch0 | I2S0MCLK SPI1CSN |
| 17 | 32 | free | ADC8 | LGPT0 ch0 | CAN0TX I2S0SCLK UART0RTS |
| 18 | 31 | free | ADC7 | LGPT0 ch1 | CAN0RX I2S0WS UART0CTS |
| 19 | 38 | SPI flash chip select | ADC6 |  | I2S0SD0 SPI0CSN UART0RXD UART0TXD |
| 20 | 30 | button BTN-1 | ADC5 |  | I2S0SD1 SPI0SCLK UART0RXD UART0TXD |
| 21 | 2 | free | ADC4 |  | I2C0SCL SPI0MOSI UART1TXD |
| 22 | 6 | free | ADC3 |  | I2C0SDA SPI0MISO UART1RXD |
| 23 | not on the header | 32 kHz crystal pad |  | LGPT0 ch2, LGPT1 ch0 | SPI1CSN UART1RTS |
| 24 | not on the header | 32 kHz crystal pad |  | LGPT0 ch0 | SPI1SCLK UART1CTS |
| 27 | 9 | free | ADC1 |  | I2C0SCL SPI1MOSI |
| 28 | 10 | free | ADC0 |  | I2C0SDA SPI1MISO |



## Components in this repository

- `cc27xx`, `cc23xx`: the target platforms (schema, pins, `run_compile`, `upload_program`), `ti_build.py` (SysConfig, compile, link, image CRC, Intel hex, DSLite), `periph.py` (YAML to SysConfig), `shim/` (what ESPHome's Zephyr-flavoured surface needs, answered with FreeRTOS and TI's drivers).
- `cc35xx`: the Wi-Fi platform (TI's Wi-Fi SDK, Toolbox signing and programming).
- `die_temperature`: chip temperature sensor on TI's driver (ESPHome's `internal_temperature` accepts only its own platform list).
- `ti_adc`: voltage on an analog-capable DIO. `ti_pwm`: PWM output on a DIO for `light` and `fan`.

Four workarounds, each in the code with its reason: ESPHome's Zigbee component switches its switch support on by the name `nrf52`, so the platform adds that name to the loaded integrations; `esphome upload` skips loading external components when a validated-config cache exists, so the platform removes the cache; ESPHome's `i2c:`, `logger:` and `binary_sensor` schemas are gated on `nrf52`, so the platform lets its own names through those gates and picks the nrf52 defaults (except GPIO interrupts, which default off); the shim stands in for Zephyr.

## Known limits

- Zigbee is the only radio on `cc27xx` and `cc23xx`. A CC2340R5 router needs `heap_size: 0x23BC` and then has no heap left.
- No SPI or UART bus (`spi:`, `uart:`): ESPHome's schemas for them name no platform from outside its tree and need more than a gate.
- GPIO interrupts are not wired up; a `gpio` binary sensor polls the pin in ESPHome's loop (`use_interrupt: false` is the default here).
- Two tasks log (the Zigbee stack's and ESPHome's); lines can interleave in rare cases. Log stack-trace decoding is not available.
- No over-the-air update on any board. Upload is over the XDS110 probe.
- `esphome upload` programs the whole application flash, which also clears the saved settings and the Zigbee network state; remove the device from ZHA before a second flash (a rejoin after re-flashing did not happen once; cause not found).
- The CC2340R5 chip temperature reads about 10 C below the room (cause not found). Sleepy end devices answer Home Assistant commands only at their poll time, so the platform keeps the receiver on unless `zigbee: sleepy: true`.
- `cc35xx`: the network name and password are built into the image; the native API runs without encryption (the Noise library compiles to about 25.5 kB of flash for this chip but is not wired in, and the random number source is still a stub); Home Assistant adoption was not run (the test network isolated its clients); an API client on the board itself got 20 of 20 switch commands; settings do not survive a reset; no over-the-air update.
- macOS only; the build scripts call TI's command-line tools by their macOS paths and could be made to run elsewhere.

## Licence

C/C++ under GPL-3.0, everything else MIT, as ESPHome does it; see [LICENSE](LICENSE) and [NOTICE](NOTICE). TI's SDKs and tools are not part of this repository.
