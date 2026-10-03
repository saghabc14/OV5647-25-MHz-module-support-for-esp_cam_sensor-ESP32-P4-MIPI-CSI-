# OV5647 25 MHz module support for `esp_cam_sensor` (ESP32-P4, MIPI-CSI)

A new OV5647 mode for [`esp_cam_sensor`](https://github.com/espressif/esp-video-components/tree/master/esp_cam_sensor) that works with **Raspberry-Pi-style OV5647 modules that carry their own 25 MHz crystal**, such as the Innomaker "RPi 1080P OV5647 M12" module. The stock OV5647 modes are written for a 24 MHz input clock and do not start these modules.

Mode added: `MIPI_2lane_25Minput_RAW10_640x480_62fps` (2 lanes, RAW10, 640x480, about 62 fps, 291.7 Mbps per lane, GBRG Bayer).

## The problem

On an ESP32-P4 board with an OV5647 module that has its own 25 MHz crystal and no clock supplied by the host:

- The sensor answers on I2C (PID `0x5647`, registers read back correctly).
- No frame ever arrives. `VIDIOC_DQBUF` times out.
- The CSI receiver never sees a high-speed clock: `phy_rxclkactivehs` stays 0, only data lane 0 reaches LP-11, the clock lane and lane 1 never do, no DMA completions, no CSI error flags.

## What the cause turned out to be

1. **The register tables.** The Espressif OV5647 tables assume a 24 MHz input clock. With the 800x640 RAW8 table at 25 MHz the sensor never launched a clock. Loading the register set from the Linux `ov5647` driver (which targets 25 MHz modules) made the clock lane go high-speed immediately and frames flowed.
2. **A separate hardware fault, worth ruling out first.** One module in my testing had its lane 1 P and clock P wires shorted to ground (22 ohm and 2 ohm with power off). With a short like that no software setting helps. Check with a multimeter (power off, module plugged in): the MIPI P/N pins on the 15-pin connector should read open to ground.

Things I ruled out (so you can skip them): I2C/SCCB, the enable pin, reset/PWDN GPIOs, the crystal, the ISP owning the CSI bridge, receiver lane rate (200, 292 and 400 Mbps tried), the `0x4800` MIPI control register values, `0x300E`, and the P4 PHY supply.

Useful measurement while the sensor is streaming: a healthy idle lane reads about 2.5 V on both wires of a pair (LP-11). A pair with one wire near 0 V is not being driven correctly.

## What is in this upload

Same relative paths as in `esp_cam_sensor` 2.6.0:

| File | Change |
|---|---|
| `sensors/ov5647/private_include/ov5647_mipi_2lane_25Minput_640x480_raw10_62fps.h` | **New.** The register set (copied from the Linux driver: common registers plus the 640x480 10-bit mode) |
| `sensors/ov5647/ov5647.c` | New ISP info entry, format index 5 and the format entry; `ov5647_set_stream()` skips the DVP pad-output writes (`0x3000` to `0x3002`) for this mode; `OV5647_AE_TARGET_DEFAULT` raised from `0x50` to `0x68` (my tuning choice, drop it if you prefer) |
| `sensors/ov5647/Kconfig.ov5647` | New option `CAMERA_OV5647_MIPI_RAW10_640X480_62FPS`, its default-format choice, default index 5 |
| `sensors/ov5647/private_include/ov5647_settings.h` | Line-rate define and the include for the new header |

One deliberate difference from the Linux register set: `0x3503` is `0x00` (sensor auto exposure) instead of Linux's `0x03` (manual exposure driven by controls). Without that change the image can come out black or fixed-exposure.

## How to use it

1. Copy the 4 files over a copy of `esp_cam_sensor` 2.6.0 (ESP-Video 2.5.0). The simplest way to keep your changes across component updates is a local copy referenced from your project's `main/idf_component.yml`:

   ```yaml
   dependencies:
     espressif/esp_cam_sensor:
       override_path: "../components/esp_cam_sensor"
   ```

2. Enable the mode in `sdkconfig` (or menuconfig under Component config, Espressif Camera Sensors, OV5647):

   ```
   CONFIG_CAMERA_OV5647_MIPI_RAW10_640X480_62FPS=y
   CONFIG_CAMERA_OV5647_MIPI_DEFAULT_FMT_RAW10_640X480_62FPS=y
   # CONFIG_CAMERA_OV5647_CSI_LINESYNC_ENABLE is not set
   ```

   Delete any existing `sdkconfig` first so the new options take effect.

3. Build and flash. On boot you should see:

   ```
   ov5647: Detected Camera sensor PID=0x5647
   example: video0: width=640 height=480 format=RGBP fps=62 ...
   ```

   The first JPEG after boot is black (it is the sensor's first frame, `sequence=0`). Later frames are fine.

## Tested with

- Waveshare ESP32-P4-WIFI6-POE-ETH, ESP32-P4 chip revision v1.3
- ESP-IDF 6.0.3, `esp_cam_sensor` 2.6.0, `esp_video` 2.5.0
- Innomaker OV5647 module (25 MHz crystal, M12 lens), 15-pin connector J3, I2C SCL=8 SDA=7, reset/PWDN `-1`
- `simple_video_server` example: JPEG about 46 KB per frame, live MJPEG stream OK

## Related finding on ESP32-P4 rev v1.3 (not part of the 4 files)

Do **not** enable `CONFIG_ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER` on chip rev v1.3. The board crash-loops right after the sensor is detected with `Illegal instruction` in `create_ipa_array` (`esp_ipa_pipeline_create`). The prebuilt `esp_ipa` library contains a RISC-V Zba `sh2add` instruction that the rev v1.3 core does not implement. Without the pipeline controller there is no automatic white balance, so the picture is dark with a yellow-green tint. Setting fixed ISP gains through the ISP device (`/dev/video20`) after video init works well: red balance 0.85, blue balance 2.0, brightness +40, contrast 150 gave a neutral, well-exposed image in my lighting (`V4L2_CID_RED_BALANCE` / `V4L2_CID_BLUE_BALANCE` take gain x 1000).

## Licence and credits

- The register values in the new header come from the Linux kernel `drivers/media/i2c/ov5647.c` (GPL-2.0, copyright Synopsys, Inc. and other contributors). The register addresses and values are hardware settings, but if you reuse this, please respect that driver's licence and credit it.
- The rest of `esp_cam_sensor` is Espressif's, Apache-2.0.

No warranty. This was developed by trial and error on one module and one board revision; reports from other modules and chip revisions are welcome.
