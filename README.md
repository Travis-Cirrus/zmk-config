# zmk-config

ZMK firmware config for a wireless 42-key Corne Choc v3.2 (3x6+3) running
Mikoto v7.2 controllers with WS2812 underglow and per-key RGB.

## Building

Firmware is built by GitHub Actions on every push; download the `.uf2` files
from the run's artifacts. Build targets live in `build.yaml`:

| Half  | Board                | Shield         |
| ----- | -------------------- | -------------- |
| Left  | `mikoto@7.2.0//zmk`  | `corne_left`   |
| Right | `mikoto@7.2.0//zmk`  | `corne_right`  |

The `//zmk` suffix selects the ZMK variant of the board, which Zephyr 4.1
requires; the empty middle field means "the board's only SoC".

To flash, double-tap reset on a half to mount it as a USB drive and copy the
matching `.uf2` onto it. Once the firmware is flashed, `&bootloader` on the nav
layer does the same thing without the reset button.

## ZMK version

`config/west.yml` pins ZMK to a specific `main` commit rather than the branch.
Tracking `main` unpinned is what silently broke this repo in December 2025,
when ZMK migrated from Zephyr 3.5 to 4.1 five days after the last green build.
`v0.4` has not been released, so a pinned commit is currently the only way to
get Zephyr 4.1 without following a moving branch.

To upgrade, replace the SHA with a newer one from
[zmkfirmware/zmk](https://github.com/zmkfirmware/zmk/commits/main) and rebuild.
Doing this deliberately means a broken build is always traceable to a change
you made.

## Why there are no displays

On Corne v3 revisions before v3.3, the LED chain's data line and the nice!view's
chip select are the **same net**, landing on the Pro Micro `D1` pad. The PCB
silkscreens that pad `CS` right on the controller footprint, beside `MOSI` (D2)
and `SCK` (D3).

Only one signal can drive a pin. Building with `nice_view` hands `D1` to the
display and the LEDs stay dark no matter what the config says — which is
exactly what happened for a long time on this board.

This config chooses the LEDs. The nice!views are left out of `build.yaml` and
should be unplugged from their sockets, freeing `D1` for the WS2812 chain at
`P0.08`.

### Switching back to displays

Both are possible together, but it costs one wire per half:

1. Add `nice_view_adapter nice_view` back to both shield strings in
   `build.yaml`.
2. Add the chip select override to `config/corne.overlay`:

```dts
&nice_view_spi {
    cs-gpios = <&pro_micro 0 GPIO_ACTIVE_HIGH>;
};
```

3. On each nice!view, bend the `CS` pin outward so it does not seat in the
   socket, and run a thin wire from it to the `D0` pad — the unlabelled pad
   next to the one silkscreened `CS`. No traces need cutting; the socket's `CS`
   position just becomes an unused stub.

With that wiring, never run a TRRS cable between the halves — `D0` is the TRRS
data pad and both controllers would drive the same line.

To drop the LEDs instead and keep the displays unmodified, remove the
underglow settings from `config/corne.conf`, delete `config/corne.overlay`, and
delete the `rgb_layer` from the keymap.

## Battery

54 WS2812s across both halves draw far more than the keyboard itself, and
`CONFIG_ZMK_RGB_UNDERGLOW_ON_START=y` lights them at boot. `AUTO_OFF_IDLE` cuts
external power once the board goes idle, but expect hours rather than weeks of
runtime while they are lit. `CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_USB=y` would
restrict underglow to when the keyboard is plugged in.
`CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` also trades battery for radio range.

`chain-length` is 27 per half (21 per-key + 6 underglow).

## Layers

Lower and Raise are the right-hand thumb keys. Holding both activates Nav via a
conditional layer; RGB is held from there with the top-left key.

| # | Name      | Reached by                              |
| - | --------- | --------------------------------------- |
| 0 | `default` | base                                    |
| 1 | `lower`   | right thumb 2 (`&mo 1`)                 |
| 2 | `raise`   | right thumb 3 (`&mo 2`)                 |
| 3 | `nav`     | Lower + Raise together, or `&mo 3`      |
| 4 | `rgb`     | top-left key while on Nav (`&mo 4`)     |

**Lower** — numbers on the left, symbols on the right.

**Raise** — F1-F12 on the left, arrow cluster on the right.

**Nav** — mouse movement, scrolling and clicks on the left; Home/PgUp/PgDn/End
on the right. The bottom row holds the radio controls: `BT_SEL 0`-`BT_SEL 4`
and `BT_CLR` on the left, USB/BLE output toggles plus `&sys_reset` and
`&bootloader` on the right. `&ext_power EP_TOG` is on the bottom-right of the
home row.

**RGB** — brightness, hue, saturation and effect on the right hand, with the
on/off toggle on the pinky.

Layer order matters: ZMK resolves a keypress from the highest active layer
down, so `rgb` has to sit above `nav` to be reachable while both are held.

## Repo layout

```
build.yaml            GitHub Actions build matrix
config/corne.keymap   layers and bindings
config/corne.conf     Kconfig (sleep, radio, pointing, underglow)
config/corne.overlay  WS2812 strip on P0.08
config/west.yml       ZMK version
```

The `corne` prefix works for both halves because ZMK strips the `_left` /
`_right` suffix when looking for config files.
