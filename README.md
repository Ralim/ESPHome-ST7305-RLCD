# ST7305 RLCD Driver for ESPHome

ESPHome display driver component for ST7305-based reflective LCD displays. Ported from the [waveshare-s3-rlcd-4.2 reference driver](https://github.com/waveshareteam/ESP32-S3-RLCD-4.2/tree/main/Example/XiaoZhi/XiaoZhiCode_V2.1.0/main/boards/waveshare-s3-rlcd-4.2).

**Note:** _This driver component is vibe coded._

![PXL_20260201_010637447 RAW-01 COVER-rs](https://github.com/user-attachments/assets/b4923a7d-15ca-4806-bb80-156f32c50c4d)

## Supported Panels

| Model               | Resolution   | Size  | Orientation\*   | Compatible Panels     |
| ------------------- | ------------ | ----- | --------------- | --------------------- |
| `WAVESHARE_400X300` | 400×300      | 4.2"  | Landscape (2×4) | GooDisplay GDTL042T71 |
| `OSPTEK_200X200`    | 200×200      | 1.54" | Portrait (4×2)  | Osptek YDP154H008     |
| `CUSTOM`            | User-defined | -     | User-defined    | Any ST7305 panel      |

\* _Orientation refers to the pixel arrangement blocks. The content can be rotated in 90° increments._

### Panel Equivalents

Many ST7305 panels from different manufacturers share identical specifications:

- **4.2" Landscape (400×300)**: Waveshare ESP32-S3-RLCD-4.2, GooDisplay GDTL042T71
- **1.54" Portrait (200×200)**: Osptek YDP154H008

If your panel matches the resolution and orientation of a predefined model, use that model even if the manufacturer differs.

## Features

- Multiple panel support with predefined configurations
- Custom panel support for unlisted ST7305 displays
- Full ESPHome display API (print, line, rectangle, circle, etc.)
- Rotation support: 0°, 90°, 180°, 270°
- Power modes for battery optimization
- Deep sleep friendly: panel init deferred to first update, reset pin held through deep sleep, optional skip of re-init on wake
- PSRAM support for lookup tables

## Installation

Add this to your ESPHome yaml.

```yaml
# External component
external_components:
  - source: github://kylehase/ESPHome-ST7305-RLCD
    components: [st7305_rlcd]
    refresh: 0s
```

## Configuration Examples

### Waveshare 400×300 (Default)

```yaml
spi:
  clk_pin: GPIO39
  mosi_pin: GPIO38

display:
  - platform: st7305_rlcd
    model: WAVESHARE_400X300
    cs_pin: GPIO40
    dc_pin: GPIO5
    reset_pin: GPIO41
    rotation: 0
    lambda: |-
      it.rectangle(0, 0, it.get_width(), it.get_height(), COLOR_ON);
      it.print(10, 10, id(font), "Hello World!");
```

### Osptek 200×200

```yaml
display:
  - platform: st7305_rlcd
    model: OSPTEK_200X200
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    lambda: |-
      it.circle(100, 100, 80, COLOR_ON);
```

### GooDisplay GDTL042T71 (use Waveshare preset)

```yaml
display:
  - platform: st7305_rlcd
    model: WAVESHARE_400X300 # Same specs as GDTL042T71
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
```

### Custom Panel

For panels not in the predefined list:

```yaml
display:
  - platform: st7305_rlcd
    model: CUSTOM
    width: 320
    height: 240
    orientation: LANDSCAPE # or PORTRAIT
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    lambda: |-
      it.print(0, 0, id(font), "Custom Panel");
```

## Configuration Options

| Option            | Required | Default             | Description                                               |
| ----------------- | -------- | ------------------- | --------------------------------------------------------- |
| `model`           | No       | `WAVESHARE_400X300` | Panel model                                               |
| `cs_pin`          | Yes      | -                   | SPI chip select pin                                       |
| `dc_pin`          | Yes      | -                   | Data/Command selection pin                                |
| `reset_pin`       | No       | -                   | Hardware reset pin (internal GPIO, held in deep sleep)    |
| `rotation`        | No       | 0                   | Display rotation (0, 90, 180, 270)                        |
| `update_interval` | No       | `never`             | Auto-refresh interval (use `component.update` for manual) |
| `reset_on_boot`   | No       | `true`              | Reset + init panel on first update (false: keep as-is)    |

### Custom Panel Options

When `model: CUSTOM`, these are required:

| Option        | Description                                         |
| ------------- | --------------------------------------------------- |
| `width`       | Panel width in pixels                               |
| `height`      | Panel height in pixels                              |
| `orientation` | `LANDSCAPE` (2×4 blocks) or `PORTRAIT` (4×2 blocks) |

## Rotation

| Setting         | Description            |
| --------------- | ---------------------- |
| `rotation: 0`   | Default orientation    |
| `rotation: 90`  | Rotated 90° clockwise  |
| `rotation: 180` | Upside-down            |
| `rotation: 270` | Rotated 270° clockwise |

Use `it.get_width()` and `it.get_height()` in lambda for rotation-aware dimensions.

## Power Management

The ST7305 is a reflective LCD - content is retained in low-power states.

### Power Methods

| Method              | Power | Description                                    |
| ------------------- | ----- | ---------------------------------------------- |
| `sleep()`           | ~10µA | Lowest power, content hidden, 120ms wake delay |
| `wake()`            | -     | Exit sleep mode                                |
| `low_power_mode()`  | ~1mA  | ~1Hz refresh, for static content               |
| `high_power_mode()` | ~5mA  | ~51Hz refresh, for animations                  |
| `display_on()`      | -     | Turn display on                                |
| `display_off()`     | Low   | Turn display off, RAM retained                 |
| `reset_and_init()`  | -     | Force panel hardware reset + init (~320ms)     |

### Panel Init and Deep Sleep

The panel hardware reset + init sequence (~320ms of blocking delays) is not run
in `setup()`; it runs once at the start of the first display update. With the
default `update_interval: never`, nothing is sent to the panel until
`component.update` is called.

On ESP32 targets the reset pin is latched high (`gpio_hold_en` +
`gpio_deep_sleep_hold_en`) after init, so the panel is not reset while the MCU
is in deep sleep. This requires `reset_pin` to be a native GPIO (not an I/O
expander pin). Note `gpio_deep_sleep_hold_en` is chip-wide on ESP32/S2/S3/C2/C3,
so any other pads held with `gpio_hold_en` are also held during deep sleep.

If the panel stays powered while the MCU deep sleeps, its configuration and
image survive, and the reset + init can be skipped on wake:

- `reset_on_boot: false` skips it on every boot.
- `set_reset_on_boot(bool)` decides at runtime; it must be called before the
  first update.
- `reset_and_init()` forces it at any time (e.g. if the panel lost power).

To skip it only on a deep sleep wake, and still reset + init the panel on power
on, reset button, watchdog or crash:

```yaml
esphome:
  on_boot:
    # Below the display's setup priority (600) so its buffer is allocated
    priority: 400
    then:
      - lambda: |-
          id(my_display).set_reset_on_boot(esp_reset_reason() != ESP_RST_DEEPSLEEP);
      - component.update: my_display

deep_sleep:
  run_duration: 10s
  sleep_duration: 5min

display:
  - platform: st7305_rlcd
    id: my_display
    cs_pin: GPIO40
    dc_pin: GPIO5
    reset_pin: GPIO41
    update_interval: never
    lambda: |-
      it.print(10, 10, id(font), "Hello World!");
```

See `components/st7305_rlcd/example.yaml` for a complete configuration.

**Note:** only the reset pin is held during deep sleep. CS/CLK/MOSI return to
their default pad state, so keep CS pulled high externally if spurious SPI
traffic during sleep is a concern.

## Colors

- `COLOR_ON` = Black (pixel on)
- `COLOR_OFF` = White (pixel off)

## Technical Details

### Pixel Block Structure

ST7305 panels pack 8 pixels per byte in different arrangements:

**Landscape (400×300 Waveshare):** 2 columns × 4 rows per byte

```
Bit 7: (row 0, col 0)  Bit 6: (row 0, col 1)
Bit 5: (row 1, col 0)  Bit 4: (row 1, col 1)
Bit 3: (row 2, col 0)  Bit 2: (row 2, col 1)
Bit 1: (row 3, col 0)  Bit 0: (row 3, col 1)
```

**Portrait (200×200 Osptek):** 4 columns × 2 rows per byte

```
Bit 7: (row 0, col 0)  Bit 6: (row 0, col 1)  Bit 5: (row 0, col 2)  Bit 4: (row 0, col 3)
Bit 3: (row 1, col 0)  Bit 2: (row 1, col 1)  Bit 1: (row 1, col 2)  Bit 0: (row 1, col 3)
```

### Memory Usage

| Model     | Resolution | Buffer | LUTs (PSRAM) |
| --------- | ---------- | ------ | ------------ |
| Waveshare | 400×300    | 15KB   | ~360KB       |
| Osptek    | 200×200    | 5KB    | ~120KB       |

**Note:** Requires ESP32 with PSRAM for lookup tables.

### Pin Configuration - Waveshare ESP32-S3-RLCD-4.2

```yaml
spi:
  clk_pin: GPIO39
  mosi_pin: GPIO38

display:
  - platform: st7305_rlcd
    model: WAVESHARE_400X300
    cs_pin: GPIO40
    dc_pin: GPIO5
    reset_pin: GPIO41
```

## Troubleshooting

### Display shows nothing

1. Check SPI wiring (CLK, MOSI, CS, DC)
2. Verify reset pin is connected
3. Check power supply (3.3V)
4. With `update_interval: never`, make sure `component.update` is called (the panel is only initialised on the first update)
5. If `reset_on_boot` is false (or set false at runtime) but the panel lost power, call `reset_and_init()`

### Display shows garbage

1. Verify correct model is selected
2. Check pixel block orientation matches panel

### Custom panel doesn't work

1. Ensure width/height are correct
2. Try both LANDSCAPE and PORTRAIT orientations
3. Check panel uses ST7305 controller (not ST7306, etc.)

## Version History

- **Unreleased** - Panel init deferred to first update, reset pin held through deep sleep, `reset_on_boot` option and `reset_and_init()`; `reset_pin` must be a native GPIO
- **v2.0.0** - Multi-panel support (Waveshare, Osptek, Custom)
- **v1.0.0** - Initial release (Waveshare 400×300 only)

## References

### Datasheets

- [ST7305 Controller Datasheet](https://files.waveshare.com/wiki/common/ST_7305_V0_2.pdf)
- [Osptek YDP154H008 (200×200)](https://admin.osptek.com/uploads/YDP_154_H008_V3_c24b455ff9.pdf)

### Development Boards

- [Waveshare ESP32-S3-RLCD-4.2](https://www.waveshare.com/wiki/ESP32-S3-RLCD-4.2)
- [GooDisplay GDTL042T71 (400×300)](https://www.good-display.com/product/455.html)
