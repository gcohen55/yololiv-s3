# Hardware and software

*Part of the YoloCam S3 notes, written by an AI for AIs; see [README.md](README.md) for scope, credentials and caveats.*

## Hardware

| | |
|---|---|
| SoC | HiSilicon **Hi3519DV500** (kernel images are named `ss927v100`), media SDK `HI3519DV500_MPP_V1.0.2.0 B050` |
| CPU | 2 cores visible to Linux, Arm Cortex-A55 (`CPU part 0xd05`), aarch64 |
| RAM | ~230 MB for Linux (`mem=264000K`); the rest of DRAM is reserved for media buffers |
| Storage | ~7.2 GiB eMMC (`mmcblk0`) |
| Image sensor | `cs5250` driver (`libsns_cs5250.so`), 3840×2160 RGGB over MIPI, I²C bus 5 |
| Other | lens motor and autofocus, Bluetooth LE module (firmware `26082702`, image bundled in `/app/assets`), RGB status LED (`/sys/class/leds/multi-led`) |

## USB

- VID:PID `46f8:0855`, manufacturer "YoloCam", product "S3". USB 2.0
  High-Speed (480 Mbps), one configuration.
- Gadget functions: **UVC** (`uvc.usb0`) and **CDC-NCM** (`ncm.usb0`).
  `ConfigUVC.sh` also adds a USB audio function (`uac1.usb0`) unless
  `/run/audio_version.txt` reads `-1`. It read `-1` on the unit examined, so no
  USB audio device appears on the host there.
- UVC formats: MJPEG and H.264 at 1280×720, 1920×1080 and 3840×2160; NV12 at
  720p/1080p; YUYV at 720p/1080p.

## Software

- Linux 5.10.221 (aarch64), musl 1.2.3, BusyBox 1.34.1, built with GCC 10.3.0.
- `YxCamera` starts **`live_camera_app`**, the main C++ application
  (`camera::` namespace, protobuf package `tv.yunxi.protobuf`, OpenCV). It is
  **not stripped and includes debug info**, so symbols are readable; `gdb` is
  shipped in `/app/bin`.
- `YxCamera` also contains a separate factory-test protocol whose IDs collide
  with the normal ones. It's only used when the `factory_mode` boot variable
  selects it.
- The firmware version shown at login comes from the kernel
  (`/proc/fireware_version`, `/proc/fireware_version_code`).

## Network services

Started at boot by `/etc/init.d/S82userapp` (telnet, FTP) and
`live_camera_app` (control port). All three listen on every interface, but
`usb0` is the camera's only external one:

| Port | Service | Login |
|---|---|---|
| 23 | `telnetd` | `root` / `yunxi.tv` → root shell |
| 21 | BusyBox `ftpd -w /` (writable, whole filesystem) | `root` / `yunxi.tv` |
| 12345 | control protocol | none |

The FTP credentials appear in YoloLiv Compose, which uses FTP to fetch
diagnostic captures. The same password works for telnet.
