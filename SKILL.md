---
name: comfyui-skill-openclaw
description: |
  Generate images utilizing ComfyUI's powerful node-based workflow capabilities. Supports dynamically loading multiple pre-configured generation workflows from different instances and their corresponding parameter mappings, converting natural language into parameters, driving local or remote ComfyUI services, and ultimately returning the images to the target client.
  
  **Use this Skill when:**
  (1) The user requests to "generate an image", "draw a picture", or "execute a ComfyUI workflow".
  (2) The user has specific stylistic, character, or scene requirements for image generation.
---

# ComfyUI Agent SKILL

## Core Execution Specification

As an OpenClaw Agent equipped with the ComfyUI skill, your objective is to translate the user's conversational requests into strict, structured parameters and hand them over to the underlying Python scripts to execute workflows across multi-server environments.

### UI Management Shortcut

If the user asks you to open, launch, or bring up the local Web UI for this skill, run:

```bash
python3 ./ui/open_ui.py
```

This command will:
- reuse the UI if it is already running
- start it in the background if it is not running
- try to open the browser to the local dashboard automatically

### Native ComfyUI API Surface

This skill is primarily a workflow execution client for a local or remote ComfyUI server.

The core native ComfyUI routes relevant to this skill are:

- `POST /prompt` to submit a workflow run
- `GET /history/{prompt_id}` to poll for completion
- `GET /view` to download generated images

Other native ComfyUI routes such as `/ws`, `/queue`, `/interrupt`, `/upload/image`, `/object_info`, and `/system_stats` exist upstream but are not required for the basic execution path implemented here.

For the route-level reference and the distinction between native ComfyUI routes and this repository's own manager API, see [`docs/comfyui-native-routes.md`](./docs/comfyui-native-routes.md).

### Server Health Check

Before running a workflow, check whether the target ComfyUI server is online.

You can query the manager API endpoint:

```http
GET /api/servers/{server_id}/status
```

This returns JSON with `"status": "online"` or `"status": "offline"`.

**Recommended agent flow:** Before Step 3 (Trigger Image Generation), run a server status check. If offline, ask the user to start ComfyUI and retry once it is online.

### Step 0: AI-Native Workflow Auto-Configuration (Optional)

If the user provides you with a new ComfyUI workflow JSON and asks you to "configure it" or "add it":

#### ⚠️ Workflow Format Check (Critical)
ComfyUI has two JSON formats. **Only API format works with this skill.**

- **API format** ✅: Top-level keys are node IDs (strings like `"1"`, `"6"`), each with `class_type` and `inputs`.
  ```json
  { "1": { "class_type": "KSampler", "inputs": { "seed": 42, ... } } }
  ```
- **UI format** ❌: Has a `nodes` array at top level — exported from ComfyUI browser by default.
  ```json
  { "nodes": [...], "links": [...], "last_node_id": 99 }
  ```

**If the workflow is UI format**, you must export it as API format first:
- In ComfyUI browser: enable Dev Mode (Settings → Dev Mode), then use "Save (API format)"
- Or convert programmatically — but this is error-prone for complex workflows with custom nodes

#### Steps to add a workflow:
1. Check the existing server configurations or default to `local`.
2. Save the **API format** JSON to `./data/<server_id>/<new_workflow_id>/workflow.json`
   (**Note**: path is `data/<server_id>/<workflow_id>/workflow.json`, NOT `data/<server_id>/workflows/`)
3. Analyze the JSON structure (look for `inputs` inside node definitions, e.g., `KSampler`'s `seed`, `CLIPTextEncode`'s `text` for positive/negative prompts, `EmptyLatentImage` for width/height).
4. Automatically generate a schema mapping file and save it to `./data/<server_id>/<new_workflow_id>/schema.json`
   (**Note**: path is `data/<server_id>/<workflow_id>/schema.json`, NOT `data/<server_id>/schemas/`)
   The schema format must follow:
   ```json
   {
     "workflow_id": "<new_workflow_id>",
     "server_id": "<server_id>",
     "description": "Auto-configured by OpenClaw",
     "enabled": true,
     "parameters": {
       "prompt": { "node_id": "3", "field": "text", "required": true, "type": "string", "description": "Positive prompt" }
     }
   }
   ```
5. Verify with `~/agent-venv/bin/python3 scripts/registry.py list --agent` — the new workflow should appear.
6. Tell the user that the new workflow on the specific server is successfully configured and ready to be used.

### Step 1: Query Available Workflows (Registry)

Before attempting to generate any image, you must **first query the registry** to understand which workflows are currently supported and enabled:
```bash
python ./scripts/registry.py list --agent
```

**Return Format Parsing**:
You will receive a JSON containing all available workflows. Notice they are uniquely identified by a combination of `server_id` and `id` (or path format `<server_id>/<workflow_id>`):
- For parameters with `required: true`, if the user hasn't provided them, you must **ask the user to provide them**.
- For parameters with `required: false`, you can infer and generate them yourself based on the user's description (e.g., translating and optimizing the user's scene), or simply use empty values/random numbers (e.g., `seed` = random number).
- Never expose underlying node information to the user (do not mention Node IDs); only ask about business parameter names (e.g., prompt, style).
- If multiple workflows match the user prompt across different servers, you may list them acting as candidates, OR simply pick the most relevant one and execute it directly to provide the best user experience.

### Step 2: Parameter Assembly and Interaction

Once you have identified the workflow to use and collected/generated all necessary parameters, you need to assemble them into a compact JSON string.
For example, if the schema exposes `prompt` and `seed`, you need to construct:
`{"prompt": "A beautiful landscape, high quality, masterpiece", "seed": 40128491}`

*If critical parameters are missing, politely ask the user using `notify_user`. For example: "To generate the image you need, would you like a specific person or animal? Do you have an expected visual style?"*

### Step 3: Trigger the Image Generation Task

Once the complete parameters are collected, execute the workflow client in a command-line environment (ensure your current working directory is the project root, or navigate to it first).

Pass the full identifier as `<server_id>/<workflow_id>`.

> **Note**: Outer curly braces must be wrapped in single quotes to prevent bash from incorrectly parsing JSON double quotes.

```bash
python ./scripts/comfyui_client.py --workflow <server_id>/<workflow_id> --args '{"key1": "value1", "key2": 123}'
```

**Blocking and Result Retrieval**:
- This script will automatically submit the task to the matched server and **poll to wait** for ComfyUI to finish rendering, then download the image locally.
- If executed successfully, the standard output of the script will finally provide a JSON containing an `images` list, where the absolute paths are the generated image files.
- Under the hood, this flow uses the native ComfyUI route sequence `POST /prompt` -> `GET /history/{prompt_id}` -> `GET /view`.

### Step 4: Send the Image to the User

Once you obtain the absolute local path to the generated image, use your native capabilities to present the file to the user (e.g., in an OpenClaw environment, returning the path allows the client to intercept it and convert it into rich text or an image preview).

## Common Troubleshooting & Notices
1. **ComfyUI Offline**: If the script returns "Error connecting to ComfyUI", run a server status check and ask the user to start the ComfyUI service for that server URL before retrying.
2. **Schema Not Found**: If you directly called a workflow the user mentioned verbally, but the script reports a missing Schema, perform Step 1 `registry.py` and tell the user they need to first go to the Web UI panel to upload and configure the mapping for that workflow on the desired server.
3. **Parameter Format Error**: Ensure that the JSON passed via `--args` is a valid JSON string wrapped in single quotes.
4. **HTTP 405 on POST /prompt**: Caused by system-level `HTTP_PROXY` env var (e.g. `HTTP_PROXY=http://127.0.0.1:6190`). `urllib` routes through the proxy in absolute-URI format which ComfyUI rejects. **Fix already applied** in `comfyui_client.py`: all urllib openers use `ProxyHandler({})` to bypass proxy for localhost. If you see 405 again, verify this fix is in place.

## i2i (Image-to-Image) Workflow Guide

### Prerequisites
1. Upload input image to ComfyUI first:
   ```bash
   curl -s -X POST http://127.0.0.1:8188/upload/image \
     -F "image=@/path/to/input.png" \
     -F "type=input" -F "overwrite=true"
   ```
   Returns `{"name": "filename.png", ...}` — use the `name` value as the `image` parameter.

2. Workflow must be in **API format** (flat dict with node IDs as keys), NOT UI format (has `nodes` array).
   - UI format: exported from ComfyUI browser as "Save (API format)" or converted manually.
   - The `workflow.json` in each skill data folder must be API format.

### Ready-to-use i2i workflow: `local/flux-i2i`
Located at: `data/local/flux-i2i/`
- Model: `FLUX.1-schnell 可炼丹版本_OpenFLUX_V1.safetensors`
- CLIP: `t5xxl_fp16.safetensors` + `clip_l.safetensors` (flux type)
- VAE: `ae.safetensors`
- Parameters: `prompt` (required), `image` (required), `denoise` (default 0.65), `steps` (default 20), `seed`

Example call:
```bash
cd ~/Documents/10.github/Tool-packages/comfyui-skill
~/agent-venv/bin/python3 scripts/comfyui_client.py \
  --workflow local/flux-i2i \
  --args '{"prompt": "photorealistic woman, high quality", "image": "input.png", "denoise": 0.65}'
```

### Adding a new i2i workflow
1. Export workflow from ComfyUI in **API format** (File → Save (API format))
2. Save to `data/local/<workflow-id>/workflow.json`
3. Create `data/local/<workflow-id>/schema.json` mapping key params (prompt, image, denoise, seed, etc.) to node IDs
4. Verify with `~/agent-venv/bin/python3 scripts/registry.py list --agent`
