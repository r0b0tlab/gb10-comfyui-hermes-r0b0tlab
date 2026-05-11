# Content Generation Workflows — Full Recipes

## 1. Batch Image Generation for Social Media

Generate 50 variations for A/B testing content angles:

```bash
python3 <skill-dir>/scripts/run_batch.py \
  --workflow workflows/flux_krea_txt2img.json \
  --args '{"prompt": "minimalist desk setup, warm lighting, aesthetic workspace, natural light", "steps": 20}' \
  --count 50 --randomize-seed --parallel 3 \
  --output-dir ./outputs/social-batch
```

**Tips:**
- 3 parallel jobs saturates GB10 without queue buildup
- Random seeds produce usable variety without prompt engineering
- FLUX.1 Krea's "opinionated" diversity means each seed feels intentional
- Output at 1024×1024, upscale later (see recipe 5)

## 2. Image-to-Video (Wan 2.2 FLF2V)

Smooth transition between two keyframes:

```bash
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/wan22_flf2v.json \
  --input-image start_frame=./outputs/scene_a.png \
  --input-image end_frame=./outputs/scene_b.png \
  --args '{"prompt": "smooth camera dolly, gentle parallax, cinematic lighting", "frames": 81}' \
  --timeout 1800 \
  --output-dir ./outputs/video
```

**Tips:**
- 81 frames @ 24fps = ~3.4 seconds — ideal for social loops
- End frame should be compositionally similar to start for smooth transitions
- Timeout set to 1800s — Wan 2.2 14B on GB10 takes 5-15 minutes per video
- FLF2V (First-Last Frame to Video) gives more control than pure T2V

## 3. AI Influencer Pipeline

Consistent character across image and video:

**Step 1: Generate base character**
```bash
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/flux_krea_txt2img.json \
  --args '{"prompt": "portrait of character, consistent face, studio lighting, 1024x1024"}'
```

**Step 2: Extract face embedding**
Use IPAdapter or FaceID node from ComfyUI Impact Pack to lock the character.
Load the reference image, extract the embedding, and wire it into subsequent
workflows.

**Step 3: Batch variations**
```bash
python3 <skill-dir>/scripts/run_batch.py \
  --workflow workflows/flux_faceid.json \
  --args '{"prompt": "character in [outfit], [location], [pose]"}' \
  --count 20 --randomize-seed --parallel 3
```

**Step 4: Animate key poses**
Feed key poses into Wan 2.2 FLF2V for character animation loops.

## 4. Text-First Graphics (Qwen-Image)

Qwen-Image handles text accurately across languages:

```bash
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/qwen_image_text.json \
  --args '{"prompt": "motivational quote card: THE ONLY WAY IS THROUGH, bold white text on dark gradient background, minimalist typography, 1080x1080"}'
```

**Tips:**
- Qwen-Image is 7× faster on RTX Blackwell than Apple M3 Ultra
- Supports Chinese, Japanese, Korean, Arabic text natively
- Best for memes, quote cards, infographics, multi-language content
- Keep text short — 1-2 lines renders most reliably

## 5. Upscale + Multi-Platform Repurpose

One image → every platform ratio:

```bash
# Step 1: Generate base image
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/flux_dev_txt2img.json \
  --args '{"prompt": "your prompt", "width": 1024, "height": 1024}'

# Step 2: Upscale to 4K
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/esrgan_upscale.json \
  --input-image image=./outputs/base.png \
  --args '{"scale": 4}'

# Step 3: Crop to platform ratios via ComfyUI crop/transform nodes
#   - 1:1 (1080×1080) → Instagram post
#   - 9:16 (1080×1920) → Stories, Reels, TikTok
#   - 16:9 (1920×1080) → Twitter, YouTube thumbnail
#   - 4:5 (1080×1350) → Instagram portrait
```

## 6. 3D Asset → Product Mockup

**Step 1: Generate 3D asset**
```bash
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/hunyuan3d_t2m.json \
  --args '{"prompt": "modern ceramic coffee mug, matte finish"}'
# Exports FBX/GLB with PBR materials
```

**Step 2: Render at angles**
Use ComfyUI 3D viewer or export to Blender for multi-angle renders.

**Step 3: Place in scene**
```bash
python3 <skill-dir>/scripts/run_workflow.py \
  --workflow workflows/flux_img2img.json \
  --input-image image=./outputs/mug_angle1.png \
  --args '{"prompt": "ceramic mug on wooden desk, morning sunlight, coffee shop aesthetic", "denoise": 0.5}'
```

## 7. Sound-to-Video (Music Visualization)

Use ComfyUI audio-reactive nodes to generate abstract visuals synced to audio.
Load audio file → extract features → drive visual parameters. Great for
podcast clips, music promotion, ambient content loops.

## Automation via Hermes Cronjob

Schedule daily batch generation:

```bash
# Create cronjob in Hermes
hermes cron create \
  --name "comfyui-daily-content" \
  --schedule "0 9 * * *" \
  --prompt "Generate 10 images using the comfyui skill with flux krea workflow. Prompt: trending topic of the day. Use run_batch.py --count 10 --randomize-seed --parallel 3. Save to ./outputs/daily-{date}. Then send the best 3 to Telegram using send_message with MEDIA: paths."
```

## Model Selection by Content Type

| Content Type | Model | Why |
|-------------|-------|-----|
| Creative/viral images | FLUX.1 Krea [dev] | "Opinionated" diverse output, 8× faster on RTX |
| Product shots | FLUX.1 Dev fp8 | Balanced quality/speed |
| Production all-round | SD3.5 Large | TensorRT-optimizable (3× faster with NIM) |
| Video generation | Wan 2.2 14B | Best quality, GB10 runs 14B without delay |
| Text graphics/memes | Qwen-Image | Multilingual, 7× faster on RTX |
| 3D assets | Hunyuan3D 2.1 | PBR materials, FBX/GLB export |
| Light/batch | SDXL Base | Fast, ControlNet-compatible |
