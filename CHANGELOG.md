# Changelog

## 0.2.0 (unreleased)

- Logger: `logger:` works on `cc27xx` and `cc23xx`, writing to the LaunchPad's backchannel UART; `esphome logs` reads it. ESPHome's build flags (including the log level) now reach the compiler.
- Pins: every DIO of the LP-EM-CC2755P10 and LP-EM-CC2340R5, with header pin, board use and signal table (`pindata.json`); pins are checked against it.
- Buses: `i2c:` on TI's I2C driver (ESPHome's own sensor components work on top), new `ti_adc` sensor and `ti_pwm` output.
- Saved settings: ESPHome preferences are written to flash through TI's NVS driver and survive a reset.
- A pin given to a TI driver is taken from TI's LED and button code.
- SysConfig 1.27.0 accepted next to 1.26.3.

## 0.1.0

- `cc27xx` and `cc23xx` Zigbee platforms (LED switch, chip temperature), `cc35xx` Wi-Fi platform (native API, mDNS), `die_temperature` sensor.
