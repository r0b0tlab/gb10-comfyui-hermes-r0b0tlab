# GB10 ComfyUI — Hermes Agent Skill

An [Agent Skills](https://agentskills.io) compliant skill for deploying and
operating ComfyUI on NVIDIA GB10 / DGX Spark hardware, with Hermes Agent
integration for automated content generation.

## What This Skill Does

- Installs ComfyUI optimized for GB10 (Grace-Blackwell, sm_121, ARM64, 128GB unified memory)
- Downloads the right models — FLUX.1 Krea, SD3.5, Wan 2.2, Qwen-Image, Hunyuan3D
- Runs batch image generation, video creation, and 3D asset pipelines
- Automates viral content workflows (AI influencer, memes, upscale + repurpose)
- Integrates with Hermes Agent for cron-scheduled content drops and Telegram delivery

## Quick Start

```bash
# 1. Install ComfyUI (Docker)
git clone https://github.com/luix93/DGX-Spark-ComfyUI.git && cd DGX-Spark-ComfyUI
cp .env.example .env   # edit paths
docker compose up --build -d

# 2. Download models
comfy model download --url "https://huggingface.co/Comfy-Org/flux1-dev/resolve/main/flux1-dev-fp8.safetensors" --relative-path models/checkpoints

# 3. Generate
python3 <skill-dir>/scripts/run_workflow.py --workflow workflows/flux_dev_txt2img.json --args '{"prompt": "a test", "steps": 4}' --output-dir ./outputs
```

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill — install, flags, models, quick workflows |
| `references/gb10-optimization.md` | Complete flags reference, env vars, volume mounts, pitfall details |
| `references/workflows.md` | Full viral content recipes with examples and parameters |
| `LICENSE` | MIT |

## Hardware

Built for the NVIDIA GB10 (DGX Spark, Gigabyte AI TOP):
- Grace-Blackwell superchip (sm_121)
- 128GB unified LPDDR5x memory (CPU + GPU shared)
- ARM64 (aarch64) architecture
- CUDA 13.0+, PyTorch cu130+
- Docker with NVIDIA Container Toolkit

The 128GB unified memory means you can load every major model simultaneously.
A 28GB Wan 2.2 video model + 12GB FLUX.1 + 6.5GB SD3.5 = 46.5GB — barely
a third of available memory.

## Agent Skills Compliance

This skill follows the [Agent Skills specification](https://agentskills.io/specification):
- `SKILL.md` with required `name` and `description` frontmatter
- Progressive disclosure: references loaded on demand
- Under 500 lines in main SKILL.md
- Valid name: lowercase, hyphens, matches directory

## Credits

- **luix93** — DGX-Spark-ComfyUI Docker setup (https://github.com/luix93/DGX-Spark-ComfyUI)
- **martimramos** — DGX Spark ML Guide (https://github.com/martimramos/dgx-spark-ml-guide)
- **ComfyUI team** — ComfyUI and comfy-cli (https://github.com/comfyanonymous/ComfyUI)
- **Hermes Agent** — Agent framework by Nous Research
- **r0b0tlab** — GB10 optimization, skill packaging, viral workflow recipes
