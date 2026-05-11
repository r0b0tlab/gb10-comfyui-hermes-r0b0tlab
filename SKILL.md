---
name: gb10-comfyui-hermes-r0b0tlab
description: Deploy and operate ComfyUI on NVIDIA GB10 / DGX Spark (sm_121 Blackwell, ARM64, 128GB unified memory) for automated image, video, and 3D content generation. Use when setting up ComfyUI on GB10 hardware or when automating viral content pipelines through Hermes Agent.
license: MIT
compatibility: Requires NVIDIA GB10 (DGX Spark, Gigabyte AI TOP) or any Grace-Blackwell sm_121 system with Docker + NVIDIA Container Toolkit. ComfyUI models require HuggingFace access. Python 3.10+ needed for skill scripts.
metadata:
  version: "1.0.0"
  author: mr-r0b0t (r0b0tlab)
  platform: gb10, dgx-spark, arm64
  models-supported: flux, sd3.5, wan-2.2, qwen-image, hunyuan3d
  repo: https://github.com/mr-r0b0t/gb10-comfyui-hermes-r0b0tlab
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

Download in this order. All fit simultaneously on GB10.

**Tier 1 — workhorses (download immediately):**
1. FLUX.1 Krea [dev] (~12 GB) — "opinionated" diverse images, best for creative
2. FLUX.1 Dev fp8 (~12 GB) — balanced quality/speed
3. SD3.5 Large (~6.5 GB) — strong all-rounder, TensorRT-optimizable

**Tier 2 — specialized:**
4. Wan 2.2 14B (~28 GB) — video generation (only GB10 runs 14B without delay)
5. Qwen-Image (~8 GB) — multilingual text rendering for memes/infographics
6. Hunyuan3D 2.1 (~8 GB) — text/image → 3D assets with PBR materials
7. SDXL Base (~6.5 GB) — fast light model for batch experiments

```bash
# Flux Dev fp8
comfy model download \
  --url "https://huggingface.co/Comfy-Org/flux1-dev/resolve/main/flux1-dev-fp8.safetensors" \
  --relative-path models/checkpoints

# SD3.5 Large
comfy model download \
  --url "https://huggingface.co/stabilityai/stable-diffusion-3.5-large/resolve/main/sd3.5_large.safetensors" \
  --relative-path models/checkpoints
```

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
