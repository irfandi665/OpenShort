# RTX 4060 Local Hardware Profile — Design

## Intent

Make a local OpenShorts installation use the available RTX 4060 (8 GB) for
transcription and video processing without making the application unusable on
machines where CUDA, NVENC, or the GPU renderer is unavailable.

## Goals

- Add one opt-in environment profile: `HARDWARE_PROFILE=rtx4060`.
- Apply GPU-oriented defaults only when a caller has not explicitly set the
  individual setting.
- Prefer CUDA Whisper at `float16`, GPU decode/render, and NVENC encoding.
- Keep one simultaneous ASR job and conservative NVENC parallelism for 8 GB
  VRAM.
- Preserve existing encoder probing and CPU fallback behavior.
- Make the effective profile visible in startup logs.

## Non-goals

- No dynamic VRAM scheduler or GPU-specific changes to the Gemini API flow.
- No removal of CPU support or automatic activation on arbitrary machines.
- No changes to video quality presets beyond the existing NVENC settings.

## Approaches considered

1. **Runtime profile (selected).** A small, idempotent Python module sets
   defaults early in both API and worker entry points. Explicit environment
   values win, while the existing NVENC probe remains the fallback boundary.
2. **`.env` template only.** Simple but does not ensure every entry point
   receives the settings or make the selected profile observable.
3. **Dynamic VRAM scheduling.** More adaptive but substantially larger and
   unnecessary for one fixed RTX 4060 target.

## Design

Create a focused module that reads `HARDWARE_PROFILE` once. When its value is
`rtx4060`, it supplies unset values for:

```text
FFMPEG_ENCODER=auto
FFMPEG_GPU_CUT=1
GPU_RENDER=1
WHISPER_DEVICE=cuda
WHISPER_COMPUTE=float16
TRANSCRIBE_BACKEND=whisper
ASR_GPU_CONCURRENCY=1
JOB_NVENC_SLOTS=2
```

`FFMPEG_ENCODER=auto` is intentional: it lets the existing NVENC probe select
`libx264` if the driver or encoder is unavailable. The profile must never
overwrite explicitly supplied environment values. The API and worker entry
points apply it before loading configuration that consumes these values.

The project documentation and `.env` template will show the one-line profile
activation and explain how to override a setting for troubleshooting.

## Acceptance criteria

- With `HARDWARE_PROFILE=rtx4060`, unset values resolve to the profile values.
- A caller-specified individual setting is retained.
- An unknown or absent profile changes nothing.
- The application logs the active profile and the resulting GPU-oriented
  configuration without exposing secrets.
- Existing tests continue to pass; new tests prove the three behaviors above.
