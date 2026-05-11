# GB10 ComfyUI — Optimized Flags & Environment Reference

## Full Flag Reference

Every flag the DGX-Spark-ComfyUI Docker applies automatically. If running a
manual venv install, pass these to `python main.py`:

```bash
python main.py \
  --listen 0.0.0.0 \
  --normalvram \
  --disable-dynamic-vram \
  --reserve-vram 1 \
  --disable-pinned-memory \
  --use-sage-attention \
  --force-fp16 \
  --bf16-unet \
  --bf16-vae \
  --bf16-text-enc \
  --dont-upcast-attention \
  --disable-mmap
```

### What Each Flag Does

| Flag | Purpose | Why GB10 Needs It |
|------|---------|-------------------|
| `--normalvram` | Standard VRAM mode | Unified fabric has no CPU/GPU split — don't fight it |
| `--disable-dynamic-vram` | Skip dynamic offloading | Dynamic VRAM doesn't work correctly on GB10 |
| `--reserve-vram 1` | Leave 1 GB for system | Prevents OOM from kernel allocations |
| `--disable-pinned-memory` | Skip pinning | No PCIe bus — pinning adds overhead with no benefit |
| `--use-sage-attention` | sm_121 attention kernels | SageAttention 2 compiled specifically for Blackwell |
| `--force-fp16` | Flash Attention path | Required to activate optimized attention code paths |
| `--bf16-unet` | UNet in BF16 | Blackwell's native compute precision — zero-cost vs FP16 |
| `--bf16-vae` | VAE in BF16 | Same precision benefit for decoder |
| `--bf16-text-enc` | Text encoder in BF16 | Same precision benefit for CLIP/T5 |
| `--dont-upcast-attention` | Keep FP16/BF16 in attention | Avoids unnecessary FP32 upcast penalty |
| `--disable-mmap` | Disable memory mapping | Combined with copy=False patch, prevents double allocation |

## Critical Environment Variables

```bash
# Disable PyTorch's CUDA caching allocator — let unified memory manager handle it
PYTORCH_NO_CUDA_MEMORY_CACHING=1

# Managed memory prefers device allocation (GPU side of unified pool)
CUDA_MANAGED_FORCE_DEVICE_ALLOC=1

# Triton doesn't support sm_121a yet — disable all torch.compile paths
TORCH_COMPILE_DISABLE=1
TORCHDYNAMO_DISABLE=1

# Lazy CUDA module loading = faster startup, lower memory footprint
CUDA_MODULE_LOADING=LAZY

# Match Grace CPU core count (20 cores on GB10)
OMP_NUM_THREADS=20
```

## Never-Use Flags

| Flag | Why Never |
|------|-----------|
| `--gpu-only` | Splits memory into GPU/CPU pools on a system with no split |
| `--high-vram` | Pins everything → immediate OOM on unified fabric |
| `--lowvram` | Forces CPU offloading on a system with no PCIe barrier |
| `--cpu` | Pointless on hardware with a GPU |

## Double-VRAM Fix (Manual Installs)

On unified memory, `tensor.to()` with `copy=True` (PyTorch default) duplicates
every model load. The fix patches one line in ComfyUI:

In `comfy/utils.py`, find the `tensor_to` function and change:
```python
return t.to(device, copy=True)   # BEFORE — doubles memory usage
```
to:
```python
return t.to(device, copy=False)  # AFTER — single allocation
```

The DGX-Spark-ComfyUI Docker applies this at build time. Manual/venv installs
must apply it manually.

## PyTorch Installation (cu130)

```bash
# ONLY for manual/venv installs — Docker handles this automatically
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130

# Verify
python3 -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
# Expected: True  NVIDIA GB10
```

Standard PyTorch wheels (`pip install torch`) only support up to sm_90.
The GB10 uses sm_121 (Blackwell), which requires cu130 nightly wheels
(sm_120 binary-compatible with sm_121).

## SageAttention 2 Compilation (Manual Installs)

```bash
git clone https://github.com/thu-ml/SageAttention.git
cd SageAttention
TORCH_CUDA_ARCH_LIST="12.0" pip install -e . --no-build-isolation
```

No prebuilt ARM64 + sm_121 wheel exists. The Docker image compiles from source
at build time.

## Docker Volume Mounts

| Host Path | Container | Purpose |
|-----------|-----------|---------|
| `$COMFYUI_HOST_PATH/models` | `/opt/ComfyUI/models` | Checkpoints, VAEs, LoRAs |
| `$COMFYUI_DATA_PATH/custom_nodes` | `/opt/ComfyUI/custom_nodes` | Custom nodes + Manager |
| `$COMFYUI_DATA_PATH/user` | `/opt/ComfyUI/user` | Settings, Manager config |
| `$COMFYUI_DATA_PATH/output` | `/opt/ComfyUI/output` | Generated images/videos |
| `$COMFYUI_DATA_PATH/input` | `/opt/ComfyUI/input` | Source files |
| `$COMFYUI_DATA_PATH/workflows` | `/opt/ComfyUI/workflows` | Saved workflows |

All volumes persist across container rebuilds. Models and custom nodes
survive `docker compose down && docker compose up --build`.

## Health Check

```bash
# Docker install
docker compose ps                     # both services healthy
curl http://localhost:8188/system_stats  # GPU: "NVIDIA GB10"

# Venv install
curl http://localhost:8188/system_stats
```

Verify SageAttention is loaded by checking the server startup log for
`--use-sage-attention`. If missing, the attention kernel didn't compile.

## Sources

- luix93/DGX-Spark-ComfyUI: https://github.com/luix93/DGX-Spark-ComfyUI
- martimramos dgx-spark-ml-guide: https://github.com/martimramos/dgx-spark-ml-guide
- NVIDIA official playbook: https://build.nvidia.com/spark/comfy-ui/instructions
- ComfyUI on DGX Spark: https://comfyui.org/en/comfyui-on-nvidia-dgx-spark
