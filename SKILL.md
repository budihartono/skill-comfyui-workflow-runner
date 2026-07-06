---
name: ComfyUI Workflow Runner
description: >
  Queue a ComfyUI workflow for image generation. Injects prompt and parameters
  into the correct nodes, queues to the ComfyUI server, polls until complete,
  and saves output images locally.

  Trigger when:
  - User says "run this workflow", "queue this in ComfyUI", "generate with ComfyUI",
    "send to ComfyUI", "run in ComfyUI", "use this workflow"
  - User attaches a .json file (workflow) and optionally an image file
  - User mentions image generation, ComfyUI, or diffusion
  - User types /comfyui
  - User says "generate an image" and a workflow file is in context
---

# ComfyUI Skill

Queue a ComfyUI workflow, inject prompt + parameters + images, poll for completion, save outputs locally.

**MCP tools available:** `mcp__comfyui__comfyui_health`, `mcp__comfyui__comfyui_queue_prompt`, `mcp__comfyui__comfyui_poll_status`, `mcp__comfyui__comfyui_get_outputs`, `mcp__comfyui__comfyui_list_models`, `mcp__comfyui__comfyui_upload_image`

---

## Step 0 — Receive Workflow

Workflow arrives as a file attachment (drag & drop). Read the JSON. Do NOT validate it — assume it is always correct.

Parse as ComfyUI API format:
```json
{
  "node_id": {
    "class_type": "NodeTypeName",
    "_meta": { "title": "Node Title" },
    "inputs": { "param": "value", "link": ["other_node_id", 0] }
  }
}
```

---

## Step 1 — Detect Nodes

Scan all nodes. Build a detection map. Use this priority order:

### 1a. Sampler Node
Match `class_type` in: `KSampler`, `KSamplerAdvanced`, `SamplerCustom`, `SamplerCustomAdvanced`

If multiple samplers found → `AskUserQuestion` to pick primary.

### 1b. Positive & Negative Prompt Nodes
**Method:** Graph traversal from sampler.
- `sampler["inputs"]["positive"][0]` → node_id of positive CLIPTextEncode
- `sampler["inputs"]["negative"][0]` → node_id of negative CLIPTextEncode

If traversal fails (no `positive`/`negative` key, or not a list link):
- Fallback: find `CLIPTextEncode` nodes, check `_meta.title` for "positive" / "negative" (case-insensitive)
- Still ambiguous → `AskUserQuestion` to confirm

### 1c. Latent Node
Check in order:
1. `class_type == "CAS Empty Latent Aspect Ratio Axis"` → **CAS node**
2. Any node where all three inputs `primary_dim`, `reference`, `aspect_ratio` exist → **CAS node** (renamed)
3. `class_type` in `EmptyLatentImage`, `EmptySD3LatentImage`, `EmptyHunyuanLatentVideo` → **standard latent**

### 1d. Checkpoint / Model Node
`class_type` in: `CheckpointLoaderSimple`, `CheckpointLoader`, `UNETLoader`, `DualCLIPLoader`, `CLIPLoader`

Extract model filename from `inputs["ckpt_name"]` or `inputs["unet_name"]` or `inputs["model_name"]`.

### 1e. Save Node
`class_type` in: `SaveImage`, `Image Save`, `VHS_VideoCombine`, `SaveAnimatedWEBP`, `SaveAnimatedPNG`

### 1f. Image Load Node
`class_type` in: `LoadImage`, `LoadImageMask`, `LoraLoader`, `ControlNetLoader`

For each image load node found, record node_id and the input field that expects the image filename (e.g., `image` for LoadImage).

---

## Step 2 — Model-Aware Advisory (show BEFORE asking overrides)

If checkpoint node found, check model filename (case-insensitive) and show a compact advisory:

| Pattern in filename | Advisory |
|--------------------|---------:|
| `zit` or `zimageturbo` | ⚡ Z-Image Turbo detected — steps should be ≤ 8 |
| `turbo` (not `zimageturbo`) | ⚡ Turbo model — steps should be ≤ 10 |
| `lightning` | ⚡ Lightning model — steps should be ≤ 8 |
| `lcm` | ⚡ LCM model — steps ≤ 12, CFG ~1–2 |
| `hyper` | ⚡ Hyper model — steps should be ≤ 8 |
| `flux` | ℹ️ FLUX model — CFG typically 1.0; negative prompt ignored |
| `gguf` | ℹ️ GGUF format checkpoint |
| `sdxl` or `xl_base` | ℹ️ SDXL — optimal resolution 1024×1024 |
| `sd15` or `sd1.5` or `v1-5` or `v1_5` | ℹ️ SD 1.5 — optimal resolution 512×512 |

Show advisory as a short blockquote. Do NOT block. User proceeds with any value.

---

## Step 3 — Collect Parameters

Show detected node mapping compactly, then collect overrides. Keep messages short.

**Format for detected mapping:**
```
Detected nodes:
  Positive prompt → node #3 (CLIPTextEncode) — "current text preview..."
  Negative prompt → node #6 (CLIPTextEncode) — "current text preview..."
  Sampler         → node #4 (KSampler) — steps: 20, cfg: 7, seed: 42
  Latent          → node #5 (CAS Empty Latent Aspect Ratio Axis) — 768px Height, 2:3 Portrait
  Checkpoint      → node #1 — dreamshaper_8.safetensors
```

**Collect via `AskUserQuestion`:**

Always collect:
- Positive prompt text (required — no default)
- Seed: integer or `random` (default: `random`)

Collect only if node exists in workflow:
- Negative prompt text (show current value as default)
- Steps (show current value; if model advisory triggered, show recommended max)
- CFG (show current value)
- Denoise (show current value; only show if ≠ 1.0 or if KSamplerAdvanced)
- **If CAS node:** `primary_dim` in pixels (e.g. 768) + `aspect_ratio` (see enum below) + `reference` (Width or Height)
- **If standard latent:** width + height (integers)
- Batch size (show current value)
- Save filename prefix (if SaveImage node has `filename_prefix`)
- **If image load node detected:** Ask for image file (required if node exists). Accept file path or file attachment.

**Aspect ratio options for CAS node** (show current workflow value as preselected):
- `1:1 Square`
- `2:3 Portrait`
- `3:2 Landscape`
- `3:4 Portrait`
- `4:3 Landscape`
- `9:16 Portrait`
- `16:9 Landscape`
- Custom (user types their own)

Do NOT ask for parameters not present in the workflow.

---

## Step 4 — Inject Parameters into Workflow

Mutate the parsed workflow dict in-memory. Do not write to disk.

```python
# Positive / Negative prompts
workflow[pos_id]["inputs"]["text"] = positive_prompt
if neg_id:
    workflow[neg_id]["inputs"]["text"] = negative_prompt

# Seed
actual_seed = seed if seed != "random" else random_int_0_to_2pow32()
workflow[sampler_id]["inputs"]["seed"] = actual_seed          # KSampler
# KSamplerAdvanced uses "noise_seed" instead of "seed"
if workflow[sampler_id]["class_type"] == "KSamplerAdvanced":
    workflow[sampler_id]["inputs"]["noise_seed"] = actual_seed

# Sampler params (only if user provided override)
if steps_override:  workflow[sampler_id]["inputs"]["steps"] = steps
if cfg_override:    workflow[sampler_id]["inputs"]["cfg"] = cfg
if denoise_override: workflow[sampler_id]["inputs"]["denoise"] = denoise

# CAS node
if cas_id:
    if primary_dim_override: workflow[cas_id]["inputs"]["primary_dim"] = primary_dim
    if aspect_ratio_override: workflow[cas_id]["inputs"]["aspect_ratio"] = aspect_ratio
    if reference_override: workflow[cas_id]["inputs"]["reference"] = reference
    if batch_size_override: workflow[cas_id]["inputs"]["batch_size"] = batch_size

# Standard latent
if latent_id:
    if width_override:  workflow[latent_id]["inputs"]["width"] = width
    if height_override: workflow[latent_id]["inputs"]["height"] = height
    if batch_size_override: workflow[latent_id]["inputs"]["batch_size"] = batch_size

# Save node filename prefix
if save_id and prefix_override:
    workflow[save_id]["inputs"]["filename_prefix"] = filename_prefix
```

---

## Step 4.5 — Upload Image (if needed)

If image load nodes detected and user provided image file:

```
1. mcp__comfyui__comfyui_upload_image(server=server, image_path=user_image_path)
   - Returns: { filename, subfolder } (the ComfyUI server path reference)
   
2. Inject uploaded image reference into each image load node:
   workflow[image_node_id]["inputs"]["image"] = [filename, subfolder]
```

Only call if at least one image node + image file provided. Skip otherwise.

---

## Step 5 — Health Check + Queue

Determine server name: from `--server <name>` argument if provided, else `"default"`.

```
1. mcp__comfyui__comfyui_health(server=server)
   - If status != "ok" → report error, stop
   - Show: "Server OK (Xms)"

2. mcp__comfyui__comfyui_queue_prompt(server=server, workflow=mutated_workflow)
   - Returns: { prompt_id, number, server }
   - Show: "Queued — prompt_id: <id>, position #<number>"
```

---

## Step 6 — Poll Until Complete

Poll loop. Show one-line status updates.

```
while True:
    result = mcp__comfyui__comfyui_poll_status(server=server, prompt_id=prompt_id)

    status == "queued"    → print "⏳ Queued (position #N)..." → wait, continue
    status == "running"   → print "⚙️  Running..." → wait, continue
    status == "completed" → break → proceed to Step 7
    status == "error"     → print "❌ Error: <details>" → stop
    status == "cancelled" → print "🚫 Cancelled" → stop
```

Poll interval: 3 seconds for first 3 polls, then 6 seconds. Do NOT poll faster than 2 seconds.

Do NOT dump full JSON of poll results — only show status + position.

---

## Step 7 — Get Outputs + Save Locally

```
mcp__comfyui__comfyui_get_outputs(server=server, prompt_id=prompt_id)
→ { outputs: [{ filename, subfolder, absolute_path, view_url, type }] }
```

**Save directory:** `~/Pictures/ComfyUI/` by default. Override with `COMFYUI_OUTPUT_DIR` env var or `--output-dir` arg. Create dir if not exists.

For each output file:
- **Local mode** (`absolute_path` present): file already exists on disk at ComfyUI's output dir. Report the path. Optionally copy to save dir if `absolute_path` is not already in save dir.
- **Cloud mode** (`view_url` present, no `absolute_path`): download bytes from `view_url`, write to save dir. Handle 302 redirects.

**Filename collision:** if file already exists in save dir, append `_1`, `_2`, etc. before extension.

Report saved paths:
```
✅ Done — 1 image saved:
   ~/Pictures/ComfyUI/ComfyUI_00123.png
```

---

## Optional: List Available Models

If user says "what models are available", "list checkpoints", "show me models", etc.:
```
mcp__comfyui__comfyui_list_models(server=server, filter="checkpoints")
```
Show as a compact list. Do NOT call this unless explicitly requested.

---

## Argument Reference

| Arg | Effect |
|-----|--------|
| `--server <name>` | Target server from MCP config (default: `"default"`) |
| `--output-dir <path>` | Override local save directory |
| `--skip-health` | Skip the health check call |

---

## Token Efficiency Rules

- Never dump full workflow JSON to conversation
- Only show relevant node IDs and field values
- Poll status: one line per update, not raw JSON
- Skip `comfyui_list_models` unless user asks
- Don't re-read workflow after injection — mutate in-memory and queue directly
- Collapse detection step into a compact summary table, not prose
