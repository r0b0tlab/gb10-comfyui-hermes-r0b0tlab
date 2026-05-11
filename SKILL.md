---
name: gb10-comfyui-hermes-r0b0tlab
description: Deploy and operate ComfyUI on NVIDIA GB10 / DGX Spark (sm_121 Blackwell, ARM64, 128GB unified memory) for automated image, video, and 3D content generation. Use when setting up ComfyUI on GB10 hardware or when automating viral content pipelines through Hermes Agent.
license: MIT
compatibility: Requires NVIDIA GB10 (DGX Spark, Gigabyte AI TOP) or any Grace-Blackwell sm_121 system with Docker + NVIDIA Container Toolkit. ComfyUI models require HuggingFace access. Python 3.10+ needed for skill scripts.
metadata:
  version: "1.3.0"
  author: "@mr-r0b0t on X — r0b0tlab"
  platform: gb10, dgx-spark, arm64
  models-supported: flux.1, sdxl, sd3.5, flux2-klein, flux2-dev, nvfp4
  repo: https://github.com/r0b0tlab/gb10-comfyui-hermes-r0b0tlab
---

# GB10 ComfyUI — Hermes Agent Skill

Deploy ComfyUI on the NVIDIA GB10 superchip and automate content generation
through Hermes Agent. The GB10's 128GB unified memory lets you run every major
model simultaneously — a 28GB Wan 2.2 video model + 12GB FLUX.1 + 6.5GB SD3.5
still leaves 80GB free.

## Quick Install

```bash
git clone https://github.com/luix93/DGX-Spark-ComfyUI.git && cd DGX-Spark-ComfyUI
# Place comfy_kitchen wheel in wheels/
cp .env.example .env   # edit COMFYUI_HOST_PATH and COMFYUI_DATA_PATH
docker compose up --build -d
curl http://localhost:8188/system_stats   # verify → JSON with GPU info
```

ComfyUI on :8188, optional mobile UI on :3000.

This Docker setup auto-applies: CUDA 13.1, SageAttention 2 (compiled for sm_121),
NVFP4 quantization via Comfy Kitchen, double-VRAM fix (copy=False patch in
comfy/utils.py), ComfyUI-Manager, and all optimized flags below.

## Critical Flags (applied automatically by Docker)

| Flag | Why |
|------|-----|
| `--normalvram` | Don't fight unified memory fabric |
| `--disable-dynamic-vram` | Models stay loaded; dynamic VRAM doesn't work on GB10 |
| `--reserve-vram 1` | 1 GB buffer for system |
| `--disable-pinned-memory` | No PCIe barrier on unified fabric |
| `--use-sage-attention` | sm_121-optimized attention (compiled from source) |
| `--force-fp16` | Flash Attention path |
| `--bf16-unet --bf16-vae --bf16-text-enc` | Blackwell native precision |
| `--dont-upcast-attention` | Keep attention in FP16/BF16 for speed |
| `--disable-mmap` | Combined with copy=False patch, fixes double memory |

**NEVER use on GB10:** `--gpu-only` or `--high-vram` (cause OOM by fighting
unified fabric).

## Environment Variables

```bash
export PYTORCH_NO_CUDA_MEMORY_CACHING=1   # unified memory manager handles allocation
export CUDA_MANAGED_FORCE_DEVICE_ALLOC=1  # prefer device allocation
export TORCH_COMPILE_DISABLE=1            # Triton lacks sm_121a support
export CUDA_MODULE_LOADING=LAZY           # faster startup
export OMP_NUM_THREADS=20                 # match Grace CPU cores
```

## Model Download Priority

**FLUX.2 [klein] 4B is the recommended primary model for GB10.** Sub-second
generation, Apache 2.0, unified txt2img + image editing. Officially supported
in ComfyUI with workflow templates. Downloads verified working on GB10.

**Primary — FLUX.2 [klein] 4B distilled (4-step, sub-second):**

```bash
# Diffusion model (~3.8 GB, FP8)
wget --header="Authorization: Bearer $HF_TOKEN" \
  -O models/diffusion_models/flux-2-klein-4b-fp8.safetensors \
  "https://huggingface.co/black-forest-labs/FLUX.2-klein-4b-fp8/resolve/main/flux-2-klein-4b-fp8.safetensors"

# Text encoder (~7.5 GB)
wget --header="Authorization: Bearer $HF_TOKEN" \
  -O models/text_encoders/qwen_3_4b.safetensors \
  "https://huggingface.co/Comfy-Org/flux2-klein-4B/resolve/main/split_files/text_encoders/qwen_3_4b.safetensors"

# VAE (~321 MB)
wget --header="Authorization: Bearer $HF_TOKEN" \
  -O models/vae/flux2-vae.safetensors \
  "https://huggingface.co/Comfy-Org/flux2-dev/resolve/main/split_files/vae/flux2-vae.safetensors"

# Official workflow template
curl -L "https://raw.githubusercontent.com/Comfy-Org/workflow_templates/refs/heads/main/templates/image_flux2_klein_text_to_image.json" \
  -o workflows/flux2_klein_4b_txt2img.json
```

**Model storage:**
```
models/diffusion_models/flux-2-klein-4b-fp8.safetensors
models/text_encoders/qwen_3_4b.safetensors
models/vae/flux2-vae.safetensors
```

**Fallback models (battle-tested, ungated):**

```bash
# FLUX.1 Dev fp8 (~17 GB) — Comfy-Org community repack
wget --header="Authorization: Bearer $HF_TOKEN" \
  -O models/checkpoints/flux1-dev-fp8.safetensors \
  "https://huggingface.co/Comfy-Org/flux1-dev/resolve/main/flux1-dev-fp8.safetensors"

# SDXL Base 1.0 (~6.5 GB) + VAE (~320 MB) — public, ungated
wget --header="Authorization: Bearer $HF_TOKEN" \
  -O models/checkpoints/sd_xl_base_1.0.safetensors \
  "https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors"
wget --header="Authorization: Bearer $HF_TOKEN" \
  -O models/vae/sdxl_vae.safetensors \
  "https://huggingface.co/stabilityai/sdxl-vae/resolve/main/diffusion_pytorch_model.safetensors"
```

**⚠️ Gated or format-limited:**

| Model | Size | Status |
|-------|------|--------|
| FLUX.2 [klein] 9B | ~8 GB | Needs HF access (gated repo) |
| SD3.5 Large | 6.5 GB | Gated — request at stabilityai HF |
| ERNIE-Image-Turbo | ~16 GB | Diffusers format — ComfyUI support TBD |
| Wan 2.2 14B | 28 GB | Check availability |
| Qwen-Image | 8 GB | NVFP4 version available from Comfy-Org HF |
| Hunyuan3D 2.1 | 8 GB | Check ComfyUI compatibility |

## NVFP4 Conversion (Blackwell GPUs — 2× faster, 50% smaller)

NVFP4 uses Blackwell FP4 tensor cores. Convert FP8 checkpoints to NVFP4:

```bash
# FLUX.2 [klein] 4B: 3.8 GB → ~1.9 GB
docker exec comfyui python3 convert_fp8_to_nvfp4.py \
  --input /opt/ComfyUI/models/diffusion_models/flux-2-klein-4b-fp8.safetensors \
  --output /opt/ComfyUI/models/diffusion_models/flux-2-klein-4b-nvfp4.safetensors

# FLUX.1 Dev fp8: 17 GB → ~8.5 GB
docker exec comfyui python3 convert_fp8_to_nvfp4.py \
  --input /opt/ComfyUI/models/checkpoints/flux1-dev-fp8.safetensors \
  --output /opt/ComfyUI/models/checkpoints/flux1-dev-nvfp4.safetensors
```

**Requirements:** Blackwell GPU, PyTorch cu130 (in Docker), comfy_kitchen ≥ 0.2.7.
**Never use NVFP4 without cu130 PyTorch** — runs 2× slower.

**NVFP4 Compatibility:** Works with CheckpointLoaderSimple (FLUX.1 Dev, SDXL, SD3.5).
Breaks UNETLoader for FLUX.2 [klein] 4B (produces static). FLUX.2 [dev] 32B
appears to work with UNETLoader.

## Utility Nodes

```bash
comfy node install comfyui-impact-pack          # face detailer, regional prompting
comfy node install comfyui-animatediff-evolved  # video generation
comfy node install comfyui-controlnet-aux       # ControlNet preprocessors
comfy node install comfyui-essentials           # common helpers
comfy node update all
```

## Running Workflows

All execution uses the skill scripts from the Hermes comfyui skill directory.
Replace `<skill-dir>` with the path to your comfyui skill installation.

```bash
# Single image
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/flux_dev_txt2img.json \
  --args '{"prompt": "cyberpunk city at night, neon reflections", "steps": 20}' \
  --output-dir ./outputs

# Batch: 50 variations with random seeds
python3 <skill-dir>/scripts/run_batch.py \
  --workflow workflows/flux_dev_txt2img.json \
  --args '{"prompt": "minimalist desk setup", "steps": 20}' \
  --count 50 --randomize-seed --parallel 3 \
  --output-dir ./outputs/batch
```

## Viral Content Pipelines

See `references/workflows.md` for full recipes. Quick overview:

| Pipeline | Models | Output |
|----------|--------|--------|
| Batch image generation | FLUX.1 Krea | 50-100 variations for A/B testing |
| Image-to-video | Wan 2.2 FLF2V | Smooth start→end frame transitions |
| AI influencer | FLUX + FaceID + Wan 2.2 | Consistent character with animation |
| Text graphics | Qwen-Image | Memes, quote cards, infographics |
| Upscale + repurpose | FLUX + ESRGAN | 1 image → 5 platform crops |
| 3D mockups | Hunyuan3D 2.1 + FLUX | 3D asset → product scene |

## Hermes Integration

```bash
# Cronjob for daily content drop
python3 <skill-dir>/scripts/run_batch.py \
  --workflow workflows/daily.json \
  --args '{"prompt": "trending topic of the day"}' \
  --count 10 --randomize-seed --parallel 3 \
  --output-dir ./outputs/daily-$(date +%Y%m%d)

# Deliver outputs via send_message
# Files land in ./outputs/ — use MEDIA: paths for Telegram delivery
```

## Pitfalls

1. **PyTorch must be cu130+** — standard wheels don't include sm_121. If
   `torch.cuda.is_available()` is False, use `pip install torch --index-url
   https://download.pytorch.org/whl/cu130`.

2. **Double-VRAM with --disable-mmap** — `tensor.to(copy=True)` duplicates
   memory on unified systems. The Docker setup patches this; manual installs
   must apply the fix to `comfy/utils.py`.

3. **Dynamic VRAM doesn't work** — always use `--disable-dynamic-vram`.

4. **Pinned memory is waste** — no PCIe barrier on unified fabric. Use
   `--disable-pinned-memory`.

5. **SageAttention needs source compilation** — no prebuilt ARM64 wheel.
   Docker handles this. Manual: `TORCH_CUDA_ARCH_LIST="12.0" pip install -e .`

6. **torch.compile broken** — Triton lacks sm_121a. Set `TORCH_COMPILE_DISABLE=1`.

7. **Only ARM64 containers** — x86_64 images won't run.

## References

- `references/gb10-optimization.md` — Complete flags reference, env vars, volume mounts, double-VRAM fix details
- `references/workflows.md` — Full viral content recipes with example commands and parameters
- **DGX-Spark-ComfyUI Docker:** https://github.com/luix93/DGX-Spark-ComfyUI
- **GB10 ML Guide:** https://github.com/martimramos/dgx-spark-ml-guide
- **NVIDIA Playbook:** https://build.nvidia.com/spark/comfy-ui/instructions
