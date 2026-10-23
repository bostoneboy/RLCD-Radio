# Changelog

## 0.3.7 - 2026-10-23

### 更新内容 / What's Changed

- **全面支持 HTTP 301/302/303/307/308 流媒体重定向跳转 (`Audio::parseHttpResponseHeader` / `Audio::httpPrint`)**：
  - 修复含 `302 Found` 动态调度跳转的在线电台（如 `https://playerservices.streamtheworld.com/api/livestream-redirect/YES933_PREM.aac` 等）无法播放的问题。
  - 放行 `301`、`302`、`303`、`307`、`308` 重定向状态码，并在 `3xx` 跳转响应阶段跳过对跳转响应体 `Content-Type`（如 `text/html`、`application/xml`）的音频格式校验，防止在解析到 `Location:` 头之前误判退出或将跳转响应体误识别为音频数据。
  - 完整支持绝对地址（`http://` / `https://`）、协议相对路径（`//...`）及根相对路径（`/...`）三种 `Location:` 跳转格式。
  - 在执行重定向请求前主动关闭当前连接并重置 `m_f_connectionClose` 状态，修复跨主机/跨协议（HTTP ↔ HTTPS）或 `Connection: close` 下连接状态残留导致的二次握手失败。
- **优化 HTTP 响应头读取与请求头规范 (`Audio::parseHttpResponseHeader` / `Audio::dismantle_host`)**：
  - 修复等待服务器响应头期间 `!m_client->available()` 循环无延时导致的 100% CPU 空转忙等问题，加入 `vTaskDelay(2)` 让出 CPU 给 Wi-Fi / lwIP / TLS 协议栈，并仅在读到有效字节（`audioFileRead() >= 0`）时扫描 `\r\n\r\n` 结束符；响应头超时时正确标记 `m_f_timeout = true`。
  - 修复显式携带默认端口（如 `:443` 或 `:80`）的重定向 URL 在 `Host:` 请求头中附带冗余默认端口号的问题，提升对严格 CDN 与 Icecast 服务器的兼容性。
  - 移除重定向请求中非标准的 `Icy-MetaData:2` 请求头，补齐 `Pragma: no-cache`、`Cache-Control: no-cache`、连接超时设置及 `.ogg`/`.flac`/`.opus` 等后缀的编码器预期识别。
- **修复多级跳转与 TLS 握手期间误触发 15 秒断流重连看门狗 (`src/main.cpp`)**：
  - 在 `startAudio()` 完成 `audio.connecttohost(url)` 调用后立即刷新 `lastDataTime`。
  - 在 `my_audio_info()` 收到连接建立（`has been established`）、重定向跳转（`redirect to` / `next URL:`）、解码器初始化（`Decoder has been initialized`）及流就绪（`stream ready`）事件时同步刷新 `lastDataTime`，防止多次 TLS 握手 + 缓冲填充耗时超过 15 秒导致陷入无限重连循环。

### 固件下载与烧录说明

| 文件 | 说明 | 烧录偏移地址 |
| --- | --- | --- |
| `firmware.factory.bin` | 完整合并镜像（含 Bootloader、分区表、应用），推荐首次烧录使用 | `0x0` |
| `firmware.bin` | 仅应用固件（适合已有分区表设备的升级烧录） | `0x10000` |
| `SHA256SUMS.txt` | 固件 SHA256 校验值 | - |

## 0.3.6 - 2026-10-22

### 更新内容 / What's Changed

- **大幅扩充 24px 简繁中文字库 (`ui_font_cn_24`)**：字库容量由原本仅 106 个简体字扩展至 **14,137 个字符**，完整覆盖：
  - **简体中文**：GB2312 全部一、二级常用汉字（6,763 字）及扩展符号。
  - **繁体中文**：Big5 常用繁体字（5,401 字）、OpenCC 简繁/港台异体字表全部字符，以及 Rime 词频前 1,500 个扩展高频字（涵盖 `瑋`、`喆`、`嬅`、`峯`、`靚` 等歌手与曲名常用字），完美支持港台及新马华语电台（如 YES 933、CAPITAL 958、BCC 中广流行网/音乐网等）的实时节目单显示。
  - **标点与符号**：完整包含中文全角标点（`《》〈〉「」『』【】（）、，。：；！？—…～`）、全角 ASCII 及 Latin-1 扩展符号。
- **优化 LVGL 缺字占位与中文换行**：
  - 关闭 `LV_USE_FONT_PLACEHOLDER`（设为 `0`），防止电台元数据中夹带 Emoji 或特殊控制字符时在 1-bit 单色反射屏上画出乱码黑框。
  - 扩展 `_lv_txt_is_break_char()` 断行范围至 `0x4E00..0x9FFF`、全角标点 `0x3000..0x303F` 及全角字符 `0xFF01..0xFF65`，使带书名号或括号的长节目名能正常自动换行。
- **增强流媒体 UTF-8 元数据容错 (`Audio::latinToUTF8`)**：
  - 当电台下发的 UTF-8 `StreamTitle` 末尾因定长截断出现残缺字节时，原地剔除残缺字节并保留合法的 UTF-8 中文字符，避免整条中文标题被误判为 Latin-1 重编码成乱码。
- **修复节目信息显示细节**：
  - 修复调节音量 1.5 秒提示消失后调用 `showCurrentStation()` 清空 `ui_Label2` 导致当前节目名消失的问题。
  - 自动去除无歌手字段电台（如 `" - 現場談話節目..."`）开头的冗余 `"- "` 前缀。

### 固件下载与烧录说明

| 文件 | 说明 | 烧录偏移地址 |
| --- | --- | --- |
| `firmware.factory.bin` | 完整合并镜像（含 Bootloader、分区表、应用），推荐首次烧录使用 | `0x0` |
| `firmware.bin` | 仅应用固件（适合已有分区表设备的升级烧录） | `0x10000` |
| `SHA256SUMS.txt` | 固件 SHA256 校验值 | - |

## 0.3.5 - 2026-09-21

- Added dual physical button control: BOOT (GPIO0) short-press volume up, long-press next station; KEY (GPIO18) short-press volume down, long-press previous station.
- Added unified `setVolume()` function shared across WebUI, physical buttons and AirPlay.
- Switched AirPlay volume mode from software PCM scaling to hardware ES8311 register control via `RAOP_VOLUME_HARDWARE`.
- Added LCD volume feedback overlay with 1.5 s auto-hide.
- Fixed ES8311 volume curve: ported the `esp_codec_dev` dB-to-register mapping so that 0–100% maps to −50 dB–0 dB instead of the broken linear mapping that caused silence at low volume and clipping at high volume.
- AirPlay disconnect no longer restores the previous volume, keeping the unified volume setting.

## 0.3.4 - 2026-08-22

- Start the network radio before enabling AirPlay, then pause it synchronously when an AirPlay client begins its handshake.
- Release the radio decoder and TLS resources before the RAOP RSA handshake to avoid fragmented internal-memory allocation failures.
- Restore network-radio playback after a failed or disconnected AirPlay session.
- Fixed iOS AirPlay volume control by parsing RAOP volume values and applying clamped software gain to PCM audio.
- Added AirPlay handshake, heap and volume diagnostics for device-side troubleshooting.
- Hardware-verified AirPlay discovery, connection, playback, spectrum visualization and iPhone volume control.

## 0.3.3 - 2026-08-21

- Fixed AirPlay being disabled because the FFT task could not allocate a contiguous internal-RAM stack.
- Moved the application background-task stacks and RAOP worker-task stacks to PSRAM.
- Moved the large RAOP RTSP/RTP contexts and AirPlay event queue to PSRAM.
- Reserve background tasks and the AirPlay listener before starting TLS radio playback.

## 0.3.2 - 2026-08-21

- Updated the classic AirPlay RAOP Bonjour TXT record for current iOS discovery.
- Corrected the RAOP protocol version from `vn=3` to `vn=65537`.
- Added the standard password, audio, feature and server-version discovery fields.
- Log the published `_raop._tcp` instance and port for discovery diagnostics.

## 0.3.1 - 2026-08-21

- Fixed missing clock and spectrum updates when AirPlay consumed memory before the original UI tasks were created.
- Start and verify the ADC, NTP and FFT tasks before initializing the AirPlay receiver.
- Retry NTP every five seconds until the first successful synchronization.
- Report RAOP mDNS registration and RTSP task creation failures instead of reporting a false successful startup.

## 0.3.0 - 2026-08-21

- Added AirPlay 1 / RAOP audio receiving with ALAC decoding and mDNS discovery.
- Added automatic source switching: AirPlay pauses the radio and disconnecting resumes the selected station.
- Reused the existing ES8311, I2S output and FFT display path for AirPlay PCM.
- Added AirPlay enable and receiver-name settings to the Chinese WebUI.
- Added upstream attribution and GPLv3 notices for the vendored RAOP component.

## 0.2.4 - 2026-08-18

- Added the firmware version to the WebUI title and configuration page.

## 0.2.3 - 2026-08-17

- Fixed the audio watchdog callback override so the application implementation is linked instead of the library's empty weak default.
- Prevented false stream reconnects while decoded PCM data is still arriving normally.

## 0.2.2 - 2026-08-17

- Fixed periodic audio reconnects caused by an obsolete `audio_process_i2s` callback signature.
- Increased the audio-data watchdog timeout from 5 to 15 seconds to tolerate brief network stalls.

## 0.2.1 - 2026-08-17

- Fixed a reboot caused by transient SPI DMA allocation failure after the audio decoder allocated its buffers.
- Added a persistent 15 KB internal DMA transfer buffer while keeping the display framebuffer and lookup tables in PSRAM.
- Reduced the SPI transfer size and queue depth to match the reflective LCD framebuffer.
- Set the audio decoder to full digital volume and made the WebUI ES8311 volume range accurately use 0-100.
- Changed display transfer failures to drop and log the affected frame instead of aborting the device.

## 0.2.0 - 2026-08-17

- Added a 24 px embedded Simplified Chinese LVGL font and localized the device and web configuration interfaces.
- Replaced the default station list with five direct Chinese MP3 radio streams and added `name|URL` configuration support.
- Added Wi-Fi setup through the `RLCD-Radio-Setup` access point and Chinese setup page.
- Pinned PioArduino 55.03.311 with Arduino-ESP32 3.3.11 and C++20 for reproducible headless builds.
- Fixed display coordinate bounds, station switching bounds, NTP persistence and initialization order, and Arduino-ESP32 3.x BSP compatibility.
- Verified generation of both the application image and a single-file factory image for transfer to another flashing machine.

## 0.1.0 - 2026-08-15

- Established the PlatformIO/Arduino baseline for the ESP32-S3 reflective LCD radio.
- Recorded the planned Chinese localization scope: LVGL CJK font support, Chinese UI and configuration text, and a small set of Chinese radio presets.
- Documented the intended headless workflow: build firmware artifacts on the build host and transfer the generated binaries for flashing elsewhere.
