# Muse on the FoloToy AI Passport

English | [简体中文](README.zh_CN.md)

An unofficial, community port of Meta's
[Muse Gadget SDK](https://github.com/facebookincubator/muse-gadget-sdk)
ESP32 firmware to the [FoloToy AI Passport](https://github.com/FoloToy/ai-passport),
an open ESP32-C3 wearable. Pair it with the Muse app, hold a button to talk,
and read Muse's reply on its screen, in Chinese too.

> **Experimental.** Not affiliated with Meta or FoloToy. Flashing replaces the
> stock firmware; back it up first (below). Proceed at your own risk.

This repository is the full SDK with the port applied, so it builds and flashes
as is. Two pieces are already in the upstream SDK:
[#31](https://github.com/facebookincubator/muse-gadget-sdk/pull/31) (the pairing
fix) and [#32](https://github.com/facebookincubator/muse-gadget-sdk/pull/32)
(CJK captions). The board itself is under review in
[#33](https://github.com/facebookincubator/muse-gadget-sdk/pull/33), for
[issue #11](https://github.com/facebookincubator/muse-gadget-sdk/issues/11); this
repository tracks that branch on top of the latest upstream `main`. The SDK's
own overview is in [`README.upstream.md`](README.upstream.md).

## What works

| | |
|---|---|
| Pairing | BLE pairing with the Muse app, confirmed with the OK button; Wi-Fi (2.4 GHz) |
| Push-to-talk | Hold **OK** to record; the voice note streams to Muse as you speak |
| Replies | Text in the caption band at the bottom, a few seconds to half a minute after you let go; Chinese and Japanese included. A reply too big for the device shows "REPLY TOO LONG: SEE MUSE APP" |
| Menu | **UP** / **DOWN** open the menu and move through it: volume, brightness, mic gain, sleep, Status, Battery, power off |
| Battery | Level and voltage from the CW2017 gauge |
| Screen | 240x320 ST7789P3 with the SDK's compact layout (as on the StickS3), rounded corners masked |

Not supported: spoken replies (Muse sends text only; a board with PSRAM can add
its own TTS), images, the home-network tunnel, over-the-air updates and the
charging state. Untested: long-term stability and deep-sleep current.

## Hardware

| | |
|---|---|
| Chip | ESP32-C3, single core, 160 MHz, ~400 KB SRAM, **no PSRAM** |
| Flash | 8 MB embedded, DIO |
| Display | 240x320 ST7789P3, SPI, rounded corners |
| Audio | ES8311 codec, mic and speaker over I2S |
| Buttons | UP, DOWN, OK on the right edge, one ADC ladder on GPIO0 |
| Battery | CW2017 fuel gauge, 500 mAh |
| USB | The chip's own USB Serial/JTAG (`/dev/cu.usbmodem*`, `/dev/ttyACM*`) |

Pins, the panel's init table and the button windows come from FoloToy's BSP
(`components/bsp/include/bsp_pins.h` in
[FoloToy/ai-passport](https://github.com/FoloToy/ai-passport)).

## Build and flash

You need:

- **ESP-IDF v6.0.1** with the ESP32-C3 toolchain. Other versions aren't
  supported.
- **An SDK token** from [gadgets.muse.ai](https://gadgets.muse.ai/settings/sdk-tokens).
  Read the [Gadget SDK Terms](https://gadgets.muse.ai/sdk-terms). The token is
  built into the firmware: don't share your build or commit the token.
- **The Muse app** on your phone, with Settings > Devices > Developer mode on.

```sh
# 1. ESP-IDF 6.0.1 (once)
git clone -b v6.0.1 --recursive https://github.com/espressif/esp-idf.git ~/esp/esp-idf-v6.0.1
~/esp/esp-idf-v6.0.1/install.sh esp32c3
. ~/esp/esp-idf-v6.0.1/export.sh

# 2. This firmware
git clone https://github.com/timzenxia/muse-ai-passport
cd muse-ai-passport/esp32

# 3. Your SDK token (build-muse-ai-passport/ is ignored by Git)
mkdir -p build-muse-ai-passport
echo 'CONFIG_GADGET_SDK_TOKEN="mgst_..."' > build-muse-ai-passport/sdkconfig

# 4. Build
tools/muse/board.sh build ai-passport

# 5. Back up the stock firmware, keep it outside Git
python -m esptool --chip esp32c3 -p PORT read-flash 0 0x800000 ai-passport-backup.bin

# 6. Erase (the stock firmware's NVS layout differs) and flash
python -m esptool --chip esp32c3 -p PORT erase-flash
tools/muse/board.sh flash ai-passport PORT
```

`PORT` is the Passport's serial port, `/dev/cu.usbmodem*` on macOS or
`/dev/ttyACM*` on Linux. Turn the Passport on first: switched off, it doesn't
show up on USB.

Then in the Muse app: Settings > Devices > Add Device, pick
`MuseGadget-XXXXXX`, press **OK** on the Passport when asked, and choose your
Wi-Fi. Hold **OK** and talk.

To go back to the stock firmware:

```sh
python -m esptool --chip esp32c3 -p PORT write-flash 0 ai-passport-backup.bin
```

More detail: the "AI Passport port" section of
[`esp32/devices/README.md`](esp32/devices/README.md), and
[`esp32/AGENTS.md`](esp32/AGENTS.md) for building, flashing and monitoring.

## How it fits

The C3's ~400 KB of SRAM holds both code and heap, and there is no PSRAM, so
the UI, BLE pairing, the encrypted session to Muse and a voice note barely fit
together. The port:

- moves ~42 KB of driver code out of IRAM, and trims the BLE, Wi-Fi, TCP and
  LVGL buffers ([`esp32/devices/sdkconfig.muse-ai-passport`](esp32/devices/sdkconfig.muse-ai-passport));
- reuses the Cardputer ADV's smaller session buffers, with 12 KB inbound frames
  instead of its 17 KB (with 17 KB the session comes up, but every voice note
  fails for lack of heap). A reply frame over 12 KB ends the session, which
  reconnects, and the screen says "REPLY TOO LONG: SEE MUSE APP";
- sizes the pairing decrypt buffers to the message instead of two fixed 8 KB;
- draws the menu's button hint flat, since a turned one needs a 14 KB layer;
- adds a 16x16 GNU Unifont fallback for CJK captions (in flash, no RAM), and
  line breaking for text without spaces.

With the session up, about 37 KB of heap is free; during an upload, about 9 KB.

The board driver is
[`esp32/components/muse/boards/board_ai_passport.c`](esp32/components/muse/boards/board_ai_passport.c).

## License

The SDK is Meta's, under the Apache License 2.0 ([`LICENSE`](LICENSE)); the
port's changes are under the same license. These keep their own licenses:

| Path | Upstream | License |
|---|---|---|
| [`esp32/components/minimp3/include/minimp3.h`](esp32/components/minimp3) | [lieff/minimp3](https://github.com/lieff/minimp3) | CC0-1.0 |
| [`esp32/main/pixel_font.c`](esp32/main/pixel_font.c) | Adafruit GFX `glcdfont.c` | BSD-2-Clause |
| [`esp32/components/muse/fonts/muse_font_cjk_16.c`](esp32/components/muse/fonts) | [GNU Unifont](https://unifoundry.com/unifont/) 16.0.04 | SIL OFL 1.1, or GPLv2+ with the font embedding exception |

The Apache License does not cover the [Jollybot avatar](esp32/avatar), which is
Meta's. Hardware facts are from [FoloToy/ai-passport](https://github.com/FoloToy/ai-passport)
(MIT). Dependencies fetched at build time keep their own licenses.

Muse is Meta's; FoloToy and AI Passport are FoloToy's. This project is not
endorsed by either.
