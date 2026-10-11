# YoloLiv YoloCam S3: internals and control

> **Written by an AI, for AIs.** An AI coding assistant (Claude) wrote these
> notes during an interactive reverse-engineering session with the camera's
> owner. They're meant as context for other AI agents and LLMs working with
> this camera, and humans are welcome too. Facts are from one unit running
> firmware **1.0.4 build 1349**, its release archive
> (`YunxiCamera-ch131b-20260831-1.0.4.tar.gz`), and a previously
> reverse-engineered control protocol. Other units or firmware may differ.
> Verify before relying on anything here.

> **Credentials:** telnet (port 23) and FTP (port 21) both accept
> **`root` / `yunxi.tv`**, which gives a root shell and write access to the
> whole filesystem.

## In short

- Plug it in and you get a UVC webcam plus an **Ethernet-over-USB
  interface** (CDC-NCM, `enx…`). The camera runs DHCP on it and sits at
  `192.168.N.10`, where N (123–130) is chosen at random on first boot and
  stored in U-Boot (`fw_printenv ip_addr`). The unit examined uses
  **192.168.124.10**.
- You control it by sending **framed protobuf messages to TCP port 12345**.
  Everything the vendor's desktop app does goes over that.
- Inside is a HiSilicon Hi3519DV500 running Linux 5.10 with musl and BusyBox.
  Telnet and writable FTP are open, with root access.
- Put that interface in its **own network namespace**.

## Security

Assume the camera is not secure:

- **Root access is open by default.** Telnet and writable FTP run as root with
  a fixed, published password (`yunxi.tv`).
- **There are command injection vulnerabilities in the firmware.** The
  owner leaves finding them as an exercise for the reader, in case anyone
  wants CVEs for a USB webcam.
- **The control port needs no authentication,** and that includes firmware
  upload.
- **Firmware updates are verified by MD5 only,** with no signature.
- **Everything runs as root,** including the large closed-source C++ app that
  parses everything sent to the control port.

This lack of security is why the camera's owner, who did this work with the
AI, recommends putting the camera's USB network interface in its own network
namespace. See [network-isolation.md](network-isolation.md).

## Files

| File | Contents |
|---|---|
| [control-protocol.md](control-protocol.md) | USB network link, TCP framing, message schema, session attach/heartbeat, real captured frames, command IDs |
| [network-isolation.md](network-isolation.md) | why and how to move the camera's NIC into a network namespace, including udev/systemd automation |
| [hardware-and-software.md](hardware-and-software.md) | SoC, CPU, RAM, storage, sensor, USB functions and formats, OS and apps, open network services |
| [storage-and-firmware.md](storage-and-firmware.md) | A/B partition layout, slot selection, persistent settings, firmware update format and updater behaviour |
| [image-pipeline.md](image-pipeline.md) | MPP pipeline stages, live state in `/proc/umap`, ISP tuning files, timings, diagnostic captures |
