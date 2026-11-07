# OpenGL fix notes (branch `fix/zelda-oot-opengl-black-screen`)

Base: Lichii `nx2` `9fa4a71fa051aa857e2d8f370aa31739a4f062c2` (after nx2-build-93 `09b78bd`).

## What build 93 actually contains

The Windows artifact `lichii-windows-x86_64.zip` from
[nx2-build-93](https://github.com/zortos293/Lichii/releases/tag/build-93) is the MSVC package
(42,137,088 byte `lichii.exe`). It is not the MinGW package.

Strings in that executable include `libdav1d decoder`, `libdav1d`, and
`libavcodec/libdav1d.c`, plus the compile path
`C:\dev\nx2work\ffmpeg-av1\src\FFmpeg-...\libavcodec`. There is no `libaom-av1`.
So the log line `FFmpeg has no decoder for "video/av1"` from issues
[#116](https://github.com/zortos293/Lichii/issues/116) and
[#133](https://github.com/zortos293/Lichii/issues/133) (build 75) does **not**
describe the MSVC build 93 binary. Pull request
[#284](https://github.com/zortos293/Lichii/pull/284) only added the MinGW AV1
package. MSVC was already selecting `ffmpeg-ci-av1` (Takoroka/ffmpeg-av1
`9.0.2-av1-1`) at `09b78bd`.

Ocarina of Time's intro is an AV1 movie (`video/av1`, 1920x1080) delivered
through `nx2:vdecsvc`, described to the game as pitch-linear NV12 (gsv 961,
kind `0xa`). Subtitles are a separate guest draw. A black background with
those subtitles and ~59 FPS means the present loop is running and the
subtitle programs produced fragments. It does not prove the movie texture
was sampled.

## Code changes

1. `CompileProgram` always reads `GL_PROGRAM_ERROR_POSITION_NV`. A rejected
   GLASM program used to keep the name from `glGenProgramsARB` and was bound
   anyway, and only when Renderer Debugging was on. A failed fragment program
   with a live vertex program draws no fragments (cleared black) while other
   pipelines, such as subtitles, still assemble. Rejected programs are now
   deleted. That pipeline is compiled again as GLSL. GLSL and Vulkan are
   unchanged. GLASM is still used when the driver accepts it.
2. The GLSL fallback is emitted like the GLSL backend, not like a second
   GLASM pass. Binding numbers accumulate across the pipeline instead of
   restarting at 0 in every stage, and each stage's legacy attributes
   (texture coordinates included) are rewritten onto the generics the
   previous stage actually wrote. The first attempt restored the assembly
   info but gave every stage its own binding range and an empty legacy
   mapping, so a rejected movie shader still sampled nothing. The assembly
   stage info the next GLASM stage reads is still restored. Transform
   feedback varyings stay in the GLSL shader; the NV attrib list is not
   applied on top of that fallback.
3. If `avcodec_find_decoder(AV_CODEC_ID_AV1)` returns null, the decoder tries
   `libdav1d` and then `libaom-av1` by name.
4. The first decoded frames log format, range, and a luma/chroma sample. A
   render that refuses to write logs the reason. The first written images log
   whether any sampled luma byte was non-zero and whether the GPU texture
   cache was invalidated. A missing device mapping is an error: the cache
   keeps the previous image, which is empty.

## Not claimed

This branch does not claim that Ocarina of Time's introduction renders on
OpenGL, GLASM, GLSL, or NVIDIA. No Windows build was produced here. See
`TEST_RESULTS.md`.
