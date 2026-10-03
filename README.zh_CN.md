# 在 FoloToy AI Passport 上运行 Muse

[English](README.md) | 简体中文

这是一个非官方的社区移植：把 Meta
[Muse Gadget SDK](https://github.com/facebookincubator/muse-gadget-sdk) 的 ESP32
固件移植到 [FoloToy AI Passport](https://github.com/FoloToy/ai-passport)（一款开源的
ESP32-C3 可穿戴设备）。用 Muse App 配对后，按住按键说话，Muse 的回复会显示在屏幕上，支持中文。

> **实验性项目。** 与 Meta、FoloToy 均无关联。刷机会覆盖出厂固件，请先按下文备份。风险自负。

本仓库是打好移植补丁的完整 SDK，克隆下来就能直接编译、烧录。这些改动也已提交给上游：
[#31](https://github.com/facebookincubator/muse-gadget-sdk/pull/31)（配对修复）、
[#32](https://github.com/facebookincubator/muse-gadget-sdk/pull/32)（中日韩字幕）、
[#33](https://github.com/facebookincubator/muse-gadget-sdk/pull/33)（板级支持），对应
[issue #11](https://github.com/facebookincubator/muse-gadget-sdk/issues/11)。
SDK 原本的介绍见 [`README.upstream.md`](README.upstream.md)。

## 已支持的功能

| | |
|---|---|
| 配对 | 通过 Muse App 蓝牙配对，按 OK 键确认；Wi-Fi（仅 2.4 GHz） |
| 按住说话 | 按住 **OK** 录音，边说边上传给 Muse |
| 回复 | 松开后 3 到 9 秒，回复以文字显示在屏幕上，支持中文和日文 |
| 菜单 | **UP** / **DOWN** 打开菜单并上下移动：音量、亮度、麦克风增益、休眠、Status、Battery、关机 |
| 电量 | 由 CW2017 电量计读取电量百分比和电压 |
| 屏幕 | 240x320 ST7789P3，遮住圆角，字幕避开外壳遮挡 |

暂不支持：语音回复（Muse 目前只返回文字；带 PSRAM 的板子可以自行接入 TTS）、显示图片、家庭网络隧道、OTA 升级、充电状态。
尚未测试：长时间稳定性、深度睡眠功耗、超长回复。

## 硬件

| | |
|---|---|
| 芯片 | ESP32-C3，单核 160 MHz，约 400 KB SRAM，**无 PSRAM** |
| Flash | 8 MB 内置，DIO |
| 屏幕 | 240x320 ST7789P3，SPI，圆角 |
| 音频 | ES8311 编解码器，麦克风和扬声器走 I2S |
| 按键 | 右侧 UP、DOWN、OK 三个键，共用 GPIO0 上的一组 ADC 分压电阻 |
| 电池 | CW2017 电量计，500 mAh |
| USB | 芯片自带的 USB Serial/JTAG（macOS 为 `/dev/cu.usbmodem*`，Linux 为 `/dev/ttyACM*`） |

引脚、屏幕初始化参数和按键电压窗口均来自 FoloToy 的 BSP
（[FoloToy/ai-passport](https://github.com/FoloToy/ai-passport) 中的 `components/bsp/include/bsp_pins.h`）。

## 编译与烧录

需要准备：

- **ESP-IDF v6.0.1**，并安装 ESP32-C3 工具链。不支持其他版本。
- **SDK token**，在 [gadgets.muse.ai](https://gadgets.muse.ai/settings/sdk-tokens) 申请，并阅读
  [Gadget SDK Terms](https://gadgets.muse.ai/sdk-terms)。token 会编译进固件，请不要分享你编译出的固件，也不要把 token 提交到 Git。
- 手机上安装 **Muse App**，并打开 Settings > Devices > Developer mode。

```sh
# 1. 安装 ESP-IDF 6.0.1（只需一次）
git clone -b v6.0.1 --recursive https://github.com/espressif/esp-idf.git ~/esp/esp-idf-v6.0.1
~/esp/esp-idf-v6.0.1/install.sh esp32c3
. ~/esp/esp-idf-v6.0.1/export.sh

# 2. 获取本固件
git clone https://github.com/timzenxia/muse-ai-passport
cd muse-ai-passport/esp32

# 3. 写入你的 SDK token（build-muse-ai-passport/ 已被 Git 忽略）
mkdir -p build-muse-ai-passport
echo 'CONFIG_GADGET_SDK_TOKEN="mgst_..."' > build-muse-ai-passport/sdkconfig

# 4. 编译
tools/muse/board.sh build ai-passport

# 5. 备份出厂固件，备份文件不要放进 Git
python -m esptool --chip esp32c3 -p PORT read-flash 0 0x800000 ai-passport-backup.bin

# 6. 整片擦除（出厂固件的 NVS 布局不同，不擦除会无法启动），然后烧录
python -m esptool --chip esp32c3 -p PORT erase-flash
tools/muse/board.sh flash ai-passport PORT
```

`PORT` 是 Passport 的串口，macOS 上为 `/dev/cu.usbmodem*`，Linux 上为 `/dev/ttyACM*`。
请先开机，Passport 关机时不会出现在 USB 设备里。

然后在 Muse App 中：Settings > Devices > Add Device，选择 `MuseGadget-XXXXXX`。按提示在
Passport 上按 **OK** 确认，再选择 Wi-Fi。完成后按住 **OK** 就可以说话了。

恢复出厂固件：

```sh
python -m esptool --chip esp32c3 -p PORT write-flash 0 ai-passport-backup.bin
```

更多细节见 [`esp32/devices/README.md`](esp32/devices/README.md) 中的 "AI Passport port" 一节；
编译、烧录、看日志的方法见 [`esp32/AGENTS.md`](esp32/AGENTS.md)。

## 内存是怎么挤出来的

ESP32-C3 只有约 400 KB SRAM，代码和堆共用这一块内存，也没有 PSRAM。界面、蓝牙配对、到 Muse
的加密会话、语音上传要同时放下非常紧张。本移植做了这些调整：

- 把约 42 KB 驱动代码移出 IRAM，并收紧 BLE、Wi-Fi、TCP、LVGL 的缓冲
  （见 [`esp32/devices/sdkconfig.muse-ai-passport`](esp32/devices/sdkconfig.muse-ai-passport)）；
- 沿用 Cardputer ADV 的小会话缓冲，入站帧为 12 KB；
- 配对解密缓冲按消息实际大小分配，不再固定申请两块 8 KB；
- 菜单的按键提示改为横排，旋转显示需要一块 14 KB 的图层；
- 字幕使用 16x16 的 GNU Unifont 作为中日韩后备字体（放在 flash 中，不占内存），并支持无空格文字的换行。

会话建立后约有 37 KB 空闲堆，上传语音时最低约 9 KB。

板级驱动见
[`esp32/components/muse/boards/board_ai_passport.c`](esp32/components/muse/boards/board_ai_passport.c)。

## 许可

SDK 属于 Meta，使用 Apache License 2.0（[`LICENSE`](LICENSE)）；本移植的改动使用相同许可。以下文件保留各自的许可：

| 路径 | 来源 | 许可 |
|---|---|---|
| [`esp32/components/minimp3/include/minimp3.h`](esp32/components/minimp3) | [lieff/minimp3](https://github.com/lieff/minimp3) | CC0-1.0 |
| [`esp32/main/pixel_font.c`](esp32/main/pixel_font.c) | Adafruit GFX `glcdfont.c` | BSD-2-Clause |
| [`esp32/components/muse/fonts/muse_font_cjk_16.c`](esp32/components/muse/fonts) | [GNU Unifont](https://unifoundry.com/unifont/) 16.0.04 | SIL OFL 1.1，或 GPLv2+ 附字体嵌入例外 |

[Jollybot 头像](esp32/avatar) 归 Meta 所有，不在 Apache 许可范围内。硬件资料来自
[FoloToy/ai-passport](https://github.com/FoloToy/ai-passport)（MIT）。编译时下载的依赖保留各自的许可。

Muse 是 Meta 的产品；FoloToy 和 AI Passport 归 FoloToy 所有。本项目未获二者认可或背书。
