# Test results

Status of the reported failure: **code fix in place, not run on NVIDIA**.

The success bar is a Windows build that shows the Ocarina of Time introduction
(picture, subtitles, and audio) on OpenGL on NVIDIA. This machine cannot boot
that build: no Visual Studio, no NVIDIA OpenGL 4.6 context, and no game dump.

## What the fallback was doing wrong

A rejected GLASM movie shader was recompiled as GLSL, but not as the GLSL
backend would compile it:

- Each stage started its texture and uniform bindings at 0. The host binds
  the fragment stage after the vertex stage, so the movie sampler read the
  wrong unit.
- Legacy texture coordinates were rewritten with an empty cross-stage map, so
  the fragment shader did not read the coordinates the vertex shader wrote.
- The runtime info was the assembly one, so transform-feedback varyings were
  left out and then the NV attrib list was still applied.

Subtitles that do not use those varyings kept drawing. The movie plane stayed
black. The fallback now uses one binding range, the previous stage's converted
outputs, and a GLSL runtime info (`use_assembly_shaders` false). Assembly info
is restored for the next GLASM stage. A pipeline that actually fell back does
not also call `glTransformFeedbackAttribsNV`.

## What a log should show

- `GLASM program rejected at position`, then `compiling the GLSL fallback`.
  The introduction picture should be visible after that line. If it is still
  black, the picture is not a rejected assembly program.
- `wrote image` with `sampled_nonzero 0`: the decoder produced an empty frame.
- `has no device mapping`: the texture cache kept the previous, empty image.