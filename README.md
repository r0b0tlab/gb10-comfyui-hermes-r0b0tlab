# GB10 ComfyUI — Hermes Agent Skill

> **[@mr-r0b0t](https://x.com/mr-r0b0t) on X** — r0b0tlab

An [Agent Skills](https://agentskills.io) compliant skill for deploying and
operating ComfyUI on NVIDIA GB10 / DGX Spark hardware, with Hermes Agent
integration for automated content generation.

## Verified Models (All Producing Output on GB10)

| Model | Format | Loader | Steps | CFG | Image |
|-------|--------|--------|-------|-----|-------|
| FLUX.1 Dev | NVFP4 | CheckpointLoaderSimple | 50 | 3.5 | [pug](assets/pug_flux1_dev.png) |
| SDXL Base | NVFP4 | CheckpointLoaderSimple | 30 | 7.0 | [pug](assets/pug_sdxl.png) |
| SD3.5 Large | NVFP4 | CheckpointLoaderSimple + DualCLIPLoader | 25 | 4.0 | [pug](assets/pug_sd35.png) |
| FLUX.2 klein 4B | FP8 | UNETLoader + CLIPLoader(type=flux2, qwen) | 4 | 1.0 | [pug](assets/pug_klein.png) |
| FLUX.2 dev 32B | NVFP4 | UNETLoader + CLIPLoader(type=flux2, mistral) | 50 | 1.0 | [pug](assets/pug_dev.png) |

**NVFP4 Compatibility:** NVFP4 metadata works with CheckpointLoaderSimple (FLUX.1, SDXL, SD3.5)
but breaks UNETLoader for FLUX.2 klein 4B. FLUX.2 dev 32B with UNETLoader works.

All models published at https://huggingface.co/r0b0tlab with verified configs.

## Quick Start

```bash
# 1. Install ComfyUI (Docker)
git clone https://github.com/luix93/DGX-Spark-ComfyUI.git && cd DGX-Spark-ComfyUI
cp .env.example .env   # edit paths
docker compose up --build -d

# 2. Download models from HuggingFace
# Models at https://huggingface.co/r0b0tlab

# 3. Generate
curl -X POST http://localhost:8188/api/prompt -H "Content-Type: application/json" -d '{"prompt": {...}}'
```

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill — install, flags, models, verified settings |
| `references/gb10-optimization.md` | Full flags reference, env vars, volume mounts, pitfalls |
| `references/workflows.md` | Viral content workflow recipes |
| `assets/` | Verified model output images |
| `LICENSE` | MIT |

## Hardware

Built for the NVIDIA GB10 (DGX Spark, Gigabyte AI TOP):
- Grace-Blackwell superchip (sm_121)
- 128GB unified LPDDR5x memory
- ARM64 (aarch64), CUDA 13.0+, PyTorch cu130+

## Credits

- **[@mr-r0b0t](https://x.com/mr-r0b0t) — r0b0tlab** — GB10 optimization, skill packaging, verified model outputs
- **luix93** — DGX-Spark-ComfyUI Docker setup
- **martimramos** — DGX Spark ML Guide
- **ComfyUI team** — ComfyUI and comfy-cli
- **Hermes Agent** — Agent framework by Nous Research
