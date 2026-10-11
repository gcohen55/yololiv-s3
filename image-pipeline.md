# Image pipeline and diagnostics

*Part of the YoloCam S3 notes, written by an AI for AIs; see [README.md](README.md) for scope, credentials and caveats.*

## Image pipeline

- Stages are the HiSilicon MPP chain: VI (sensor input + ISP) → VPSS (scaling)
  → VENC (MJPEG/H.264) → UVC. Live state of every stage is readable in
  `/proc/umap/{isp,vi,vpss,venc,uvc}`, e.g. current ISO and exposure,
  noise-reduction and sharpening parameters, per-frame processing times.
- ISP tuning ships as INI "scene" files under
  `/app/bin/param/sensor_cs5250*/default/`, with per-ISO tables. They're parsed
  by `/app/lib64/libstream.so`.
- At 1280×720 MJPEG, 30 fps, the camera's own counters show about 19 ms of VI
  and 18 ms of VPSS processing per frame, and about 30 ms per frame through
  the UVC path.
- In indoor light, auto exposure ran at the full frame time (33 ms at 30 fps),
  ISO 100–200.
- VI 3DNR (temporal noise reduction) is present but **disabled** in the stock
  configuration.
- A reboot re-enumerates the camera on USB after about 12 seconds; the video
  pipeline is running about 18 seconds after boot.

## Diagnostics

GET on IDs 92 (raw), 93 (YUV), 100 (raw-AE) or 110 (raw+YUV) makes the camera
write a capture to `/app_data/cache` and reply with its path, size and
dimensions. The capture triggers on a GET, so a GET isn't always read-only on
this camera. A 4K YUV capture is 12,441,600 bytes (3840×2160, 4:2:0). Fetch
the file over FTP and delete it afterwards.

IDs 56 (start profiler) and 57 (profiler packaging done) exist in the enum, and
Compose fetches `/app_data/profiler_log.gz` over FTP after a PUSH on ID 57.
On firmware 1.0.4, though, the ID 56 handler in `live_camera_app` calls
`VideoModule::setStartProfiler` in `/app/lib64/libvideomodule.so`, which is a
stub: it returns 0 without starting anything or calling the finish callback.
Expect no log file and no PUSH on ID 57. (This comes from static
disassembly; it hasn't been exercised on a camera.)
