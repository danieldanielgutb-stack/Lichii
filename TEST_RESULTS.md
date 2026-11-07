# Test results

Status of the reported failure: **not resolved**.

The success bar was a Windows build that shows the Ocarina of Time
introduction (picture, subtitles, and audio) on OpenGL on NVIDIA. That run
did not happen.

## Environment

| Item | Result |
| --- | --- |
| Host | Linux sandbox, no Visual Studio, no MSVC, no Windows |
| GPU | No NVIDIA device available to this process |
| Game | No copy of The Legend of Zelda: Ocarina of Time was present, and none was downloaded |
| Keys / firmware / ROM | Not used and not distributed |

## What was executed

1. Source at `nx2` `9fa4a71` was read, including
   `src/video_core/host1x/nx2_video_decoder.cpp`, `ffmpeg.cpp`,
   `externals/ffmpeg/CMakeLists.txt`, `cpmfile.json`, `nx2_gsv.h`, and the
   OpenGL shader cache / program loader.
2. Build 93's published MSVC zip was downloaded and `lichii.exe` was scanned.
   `libdav1d` is linked. `libaom` is not. See `CHANGELOG_OPENGL_FIX.md`.
3. Issues [#133](https://github.com/zortos293/Lichii/issues/133),
   [#116](https://github.com/zortos293/Lichii/issues/116), and
   [#106](https://github.com/zortos293/Lichii/issues/106) were read.
   Build 75 fails with `FFmpeg has no decoder for "video/av1"`. Build 93's
   MSVC binary does not match that failure. Issue #106, where libdav1d does
   open, still shows a solid movie (green on RADV, described as black outside
   captions on NVIDIA) because the sampled image stays empty.

## What was not executed

| Check | Result |
| --- | --- |
| A. Emulator boot | Not run |
| B. Ocarina of Time boot | Not run. No game |
| C. Introduction movie | Not run |
| D. Picture + subtitles + audio together | Not run |
| E. Menu and 3D scene | Not run |
| F. OpenGL GLASM | Not run. No NVIDIA OpenGL 4.6 context |
| G. OpenGL GLSL | Not run |
| H. Restart stability | Not run |
| `cmake` / MSVC x64 Release | Not run. This machine has no Visual Studio 2022 toolchain and the official dependency bundle was not configured |
| FPS, driver version | No capture |

A Windows zip with `lichii.exe` was **not** produced. Shipping the unmodified
build 93 archive would not be a fix.

## How to tell the two causes apart on a real run

After these changes, `eden_log.txt` should contain one of:

- `FFmpeg has no decoder for "video/av1"` or `neither the built-in AV1 decoder, libdav1d, nor libaom-av1`: the movie bytes are never decoded. This was build 75, not the inspected build 93 MSVC binary.
- `selected FFmpeg decoder "libdav1d"` and `decoded frame` with `y[min,max]` both near 0: frames exist but are empty.
- `decoded frame` with a real luma range, then `wrote image` with `sampled_nonzero 0`: conversion dropped the picture.
- `has no device mapping`: the texture cache was left holding the empty image.
- `GLASM program rejected at position`: that program did not assemble. The same pipeline should then log `compiling the GLSL fallback`. If the movie appears only after that line, the rejection was the OpenGL failure. If the movie is still black and no GLASM line was logged, the picture is being sampled empty and the shader compiler is not the cause.
