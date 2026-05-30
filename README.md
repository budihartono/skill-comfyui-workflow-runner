# skill-comfyui

A Claude Code skill that runs ComfyUI workflows. Attach a workflow JSON, provide a prompt, and Claude queues it to your ComfyUI server, polls until done, and saves output images locally.

## What It Does

1. **Reads** workflow JSON (drag & drop attachment in Claude Desktop/Code)
2. **Detects** nodes automatically — positive/negative prompt, sampler, latent, checkpoint, save
3. **Shows** model-aware advisories (turbo/lightning/flux/SDXL/etc.) based on checkpoint filename
4. **Asks** for prompt, seed, and any parameter overrides (steps, CFG, dimensions, batch size…)
5. **Injects** values into workflow in-memory
6. **Queues** to ComfyUI via MCP, polls until complete
7. **Saves** output images to `~/Pictures/ComfyUI/` (or configured dir)

## Requirements

- [mcp-comfyui](../mcp-comfyui/) MCP server running and configured in `.mcp.json`
- ComfyUI server running (local or remote)

## Installation

Copy `SKILL.md` into your Claude Code skills directory and register it. The skill name is `comfyui`.

## Usage

Attach your workflow `.json` file in Claude Desktop or Claude Code, then say:

```
run this workflow — a cat sitting on a neon throne
```

Or use the slash command:

```
/comfyui
```

### Optional Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `--server <name>` | Target server key from MCP config | `default` |
| `--output-dir <path>` | Directory to save output images | `~/Pictures/ComfyUI/` |
| `--skip-health` | Skip server health check | off |

### Output Directory

Default save path: `~/Pictures/ComfyUI/`

Override with env var:
```bash
export COMFYUI_OUTPUT_DIR="/path/to/outputs"
```

## Supported Nodes

### Sampler
`KSampler`, `KSamplerAdvanced`, `SamplerCustom`, `SamplerCustomAdvanced`

### Latent
| Node | Detection |
|------|-----------|
| **CAS Empty Latent Aspect Ratio Axis** | Custom node (by author). Parameters: `primary_dim`, `reference` (Width/Height), `aspect_ratio` |
| `EmptyLatentImage` | Standard SD latent — width + height |
| `EmptySD3LatentImage` | SD3 latent |
| `EmptyHunyuanLatentVideo` | Hunyuan video latent |

### Checkpoint / Model
`CheckpointLoaderSimple`, `CheckpointLoader`, `UNETLoader`, `DualCLIPLoader`, `CLIPLoader`

### Save
`SaveImage`, `Image Save`, `VHS_VideoCombine`, `SaveAnimatedWEBP`, `SaveAnimatedPNG`

## Model-Aware Advisories

Skill parses the checkpoint filename and warns about optimal parameters:

| Filename pattern | Advisory |
|-----------------|----------|
| `zit`, `zimageturbo` | Steps ≤ 8 |
| `turbo` | Steps ≤ 10 |
| `lightning` | Steps ≤ 8 |
| `lcm` | Steps ≤ 12, CFG ~1–2 |
| `hyper` | Steps ≤ 8 |
| `flux` | CFG ~1.0, negative prompt ignored |
| `gguf` | GGUF format checkpoint |
| `sdxl`, `xl_base` | Optimal 1024×1024 |
| `sd15`, `sd1.5`, `v1-5` | Optimal 512×512 |

Advisories are informational only — never blocks execution.

## Multi-Server Support

Configure multiple servers in `.mcp.json` under `COMFYUI_SERVERS`:

```json
{
  "default": "http://localhost:8188",
  "cloud": {
    "url": "https://your-cloud-comfyui.example.com",
    "api_key": "your-key",
    "mode": "cloud"
  }
}
```

Switch at invocation:
```
run this workflow on the cloud server --server cloud
```

## CAS Empty Latent Aspect Ratio Axis

Custom node developed by the skill author. Detected by `class_type == "CAS Empty Latent Aspect Ratio Axis"` or by presence of all three inputs: `primary_dim`, `reference`, `aspect_ratio`.

Supported aspect ratios: `1:1 Square`, `2:3 Portrait`, `3:2 Landscape`, `3:4 Portrait`, `4:3 Landscape`, `9:16 Portrait`, `16:9 Landscape`

## Files

```
skill-comfyui/
├── SKILL.md            — Skill definition (install this into Claude Code)
├── README.md           — This file
└── SESSION-RESUME.md   — Dev session resume info
```
