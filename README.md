<div align="right">

**[English](README.md)** · [简体中文](README.zh-CN.md)

</div>

# agent-debug-notes

Debugging notes from 39 real sessions across Codex, Trae and WorkBuddy — the problems that cost the most time, written down so they cost it once.

<sub>Markdown · no code, all conclusions traceable to a session</sub>

---

## What this is

Session transcripts are long and mostly redundant. The useful part is usually one sentence — the moment the actual cause was identified. These notes are that sentence, with enough context to reproduce the situation.

Selection rule: **keep only conclusions that carry a specific parameter, command, threshold or reproduction path.** No general advice.

## ComfyUI on an AMD ROCm instance

This is the bulk of it — one `gfx1100` machine, from first start to running LTX-2.3.

### "Reconnecting" has three different causes

The ComfyUI page reconnecting is not a GPU problem. It is the front end losing the back end, and there are three distinct reasons, in order of how often they occur:

| Symptom | Actual cause | How to tell |
|---|---|---|
| Opens fine, dies on execute | process killed by memory pressure | `/queue` and `/history` both return `Failed to fetch` |
| Dies after a long generation | tunnel reset (radeon-tunnel) | ComfyUI process still alive |
| Dies when the notebook cell ends | foreground process torn down with it | PID gone, `/tmp/comfyui.log` gone too |

If `/queue` and `/history` both fail, the back end really is down — this is not a browser cache problem.

### Foreground start is the root of most of it

Running it as a notebook foreground process means the ComfyUI and tunnel processes get cleaned up whenever the cell ends or the Jupyter connection blips.

```bash
nohup python main.py > /tmp/comfyui.log 2>&1 &
```

The notebook cell then has to stay running, which is easy to forget while debugging.

### 22B BF16 exits hard

Loading `ltx-2.3-22b-dev.safetensors` makes the process disappear with **no Python traceback in the log**. That is a hard exit under ROCm memory pressure, not a code error.

Working parameter set, in order of importance:

```bash
# 1. AMD memory segmentation (required)
export PYTORCH_HIP_ALLOC_CONF=expandable_segments:True

# 2. disable async weight offload — the most common hard-exit path on gfx1100
python main.py --disable-async-offload --cpu-vae

# 3. further conservative
python main.py --lowvram
```

`--disable-async-offload` is the key one here. Async weight offload on AMD during 22B inference is the path most likely to die. Slower, but the process survives.

### Model filenames: case and suffix

The workflow JSON and the machine disagree:

| In the workflow JSON | Actually on disk |
|---|---|
| `ltx-2.3-22b-dev-FP8.Safetensors` | `ltx-2.3-22b-dev.safetensors` |
| `gemma_3_12B_it_fp4_mixed.safetensors` | `gemma_3_12B_it.safetensors` |

Case and the `FP8` / `fp4_mixed` suffix differ — the load will fail.

### Connection refused: what it actually means

```
PingSucceeded : True
TcpTestSucceeded : False
Connection refused
```

This combination says: the host is reachable, the local network is fine, and the key was never even tested — the cloud side is not mapping the SSH port. Not your problem.

Order of operations, and do not re-run the SSH command in between:

1. Confirm the instance shows as *running* — "ready" is not the same thing
2. Stop it, wait for *stopped*, then start again
3. Wait 1–3 minutes, re-copy the SSH command from the console (the port changes on rebuild)
4. If it still fails, contact the platform with the instance ID, public IP and port

### The dependency cascade

Installing with `--no-deps` leaves second-level dependencies missing, and the failures come in a chain:

```
missing packaging / mako
→ missing regex (transformers)
→ missing charset_normalizer / urllib3 (requests)
→ missing multidict / propcache (yarl)
→ missing aiohappyeyeballs / aiosignal / attrs / frozenlist (aiohttp)
→ tokenizers version conflict
```

Two specifics worth remembering:

- `tokenizers==0.23.0` **has no stable release on PyPI**, only rc builds. Downgrade to `0.22.2` instead of hunting for 0.23.0.
- System Python is governed by PEP 668 and will refuse a plain `pip install`. `--break-system-packages` gets past it (acceptable in a container).

### The flag is not what it looks like

```bash
# wrong: --disable-custom-nodes is not a valid flag
# right: --disable-all-custom-nodes
```

Success looks like `To see the GUI go to: http://0.0.0.0:8188` in the log.

### Reaching it through Jupyter

Replace `/lab` with `/proxy/8188/` in the JupyterLab URL.

A 404 on `/proxy/` usually means `jupyter-server-proxy` is missing, or the reverse proxy is not forwarding that path. Config lives in `/opt/venv/etc/jupyter`.

### Recovering data before destroying an instance

Executing a command does not mean files survive a destroy — the instance's scratch disk is usually deleted.

Order: `df -h` and `ls -lah /workspace` first, find whether the platform offers a persistent directory (`/data`, `/mnt`, `/persistent`, `/root/autodl-tmp`), pack and copy there, **then** destroy.

## Cocos Creator / WeChat mini-game

### Check platform support before building

Confirm the build tool supports `wechatgame` before running a Creator 3.8.8 build. Output goes to `NewProject/build/wechatgame`; the build produces artifacts and engine cache only, never touching gameplay source.

### Bundle constraints

- `ch2-8` sub-package has `isBundle=true`, with **zero cross-bundle references from the main package**
- 123 textures in the main package needs watching against the 4 MB limit
- main textures ≤ 1.5 MB

### How a mode registry is validated

The refactor checker looks for `onQuestionStart` / `onAnswer` / `onResult` on each gameplay subclass — three methods is the criterion for "implemented".

## Responsive layout

### Inputs must be 16px

```css
input { font-size: 16px; }  /* stops iOS Safari zooming on focus */
```

Below 16px, iOS zooms the page when the field receives focus.

### Horizontal clipping looks like missing content

`overflow-x: hidden` on `html/body`, combined with navigation that depends on horizontal scrolling, produces "the content exists but I cannot see it on the phone".

The fix is real single-column reflow with full wrapping — not clipping.

### The three properties that stop overflow

```css
.card      { overflow-wrap: break-word; }
.longtext  { word-break: break-all; }
.container { min-width: 0; }            /* required on flex/grid children */
```

`min-width: 0` is the one most often missed. A flex or grid child defaults to `min-width: auto` and gets stretched by long text inside it.

### Do not trust a 320px screenshot tool

Screenshot tools clip the browser's minimum layout width to the output size, so they cannot verify a narrow layout. Use device emulation instead, or you will read a clipping artifact as a page regression.

### Breakpoints

- 320px and below: smaller margins and type
- 390px: single column, vertical contact block
- Desktop: two-column project grid

## Two product decisions worth recording

### Text input → recording first

A text-first version can demonstrate that a diagnostic pipeline exists. It cannot evaluate pace, pauses, filler words, whether the delivery sounds read, or how confident it feels.

Recording becomes the primary entry point; text input drops to a backup for structural analysis of a script.

### Generic scenarios → job-talk practice

Rather than four parallel expression scenarios, one: practicing a job talk. The reasoning is that the campus is running its own competition, those teams are reachable users on day one, and that is a far stronger case for institutional support than a general-purpose speaking tool.

## Practice-quiz product notes

- Sequential mode surfaces unseen questions first
- Random mode and mock exams skip the most recent 120 seen questions
- De-duplicate on question + options + answer in non-set mode; keep repeats in set mode (a set reproduces the original paper)

## Source

| Source | Count | What it holds |
|---|---|---|
| `.codex/sessions/` | 39 rollouts | most of the debugging, with full diagnosis chains |
| `.trae-cn/memory/` | 45 session memories | 743 structured `learned` entries |
| `.workbuddy/projects/` | project sessions | delivery records |

## Note

Only reusable technical conclusions are included. Hostnames, ports and instance IDs have been replaced with placeholders; no API keys or personal information.
