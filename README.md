# Qwen-Image-2.1 on a free Kaggle GPU via ComfyUI

Run **Qwen-Image-2.1** (official INT8 weights) through **ComfyUI** on Kaggle's free
**2 × Tesla T4**, keep the ~17 GB model set cached as a **Kaggle Dataset**, and use
the full ComfyUI **web UI in your browser** — text-to-image *and* image-edit —
through a **Cloudflare Quick Tunnel**.

Everything is pinned (ComfyUI commit, official workflow commit, exact weight filenames
and sizes), so a fresh run is reproducible instead of "pull `latest` and hope".

> Status: this repository is assembled from a private research log. The Kaggle kernels,
> pins and measured numbers below come from real T4 runs. Improvements and fresh
> measurements are added incrementally.

## Why this exists

Free GPU notebooks are a great way to try large image models, but most "ComfyUI on
Kaggle/Colab" snippets share three problems:

1. **Not reproducible** — `main`/`latest` everywhere, no pinned commits.
2. **Re-download tens of GB every session** — the Qwen-Image-2.1 INT8 set is ~17 GB.
3. **Headless only** — one prompt per run, no way to actually use the ComfyUI UI.

This project addresses all three: pinned versions, a reusable Kaggle Dataset for the
weights (with a Hugging Face fallback), and an interactive browser session with a
graceful session timer.

## Architecture

```
Kaggle notebook (2 x Tesla T4, 15 GB each)
        |
        +-- ComfyUI (main.py --lowvram --force-fp16)          :8188
        |       models ->  Kaggle Dataset (/kaggle/input)  --(fallback)-->  Hugging Face
        |
        +-- cloudflared quick tunnel  ->  https://<random>.trycloudflare.com
                                                |
                                          your browser
```

## Verified environment

| Item | Value |
|---|---|
| Accelerator | 2 × Tesla T4, 15 GB VRAM each |
| Python / PyTorch / CUDA | 3.12.13 / 2.10.0+cu128 / CUDA 12.8 (driver 580.159.04) |
| System RAM | ~31 GiB total |
| ComfyUI commit | `93810483a4739a1588236919a3128d3070244146` |
| Workflow templates commit | `a7acaf8cee9ccccdaf4b26ce04a9fced4c4d13de` |
| Model repository | `Comfy-Org/Qwen-Image-2.1` |
| Model set size | 17 283 091 766 bytes (~17.3 GB) |

## Measured results (2 × T4)

| Scenario | Notebook | Time |
|---|---|---|
| Text-to-image, 1024×1024, 25 steps, official workflow (headless) | `01` | workflow 295.84 s; full notebook 350.44 s |
| Text-to-image, 1024×1024 (interactive UI) | `07` | 286.6 s (live log) / 261 s (observed) |
| Image-edit, first run (interactive UI) | `07` | 464.4 s |
| Image-edit, run with saved output (interactive UI) | `07` | 609.69 s |

- Peak VRAM during image-edit: ~10.28 GiB of 14.56 GiB on GPU 0; GPU 1 almost idle.
- Free system RAM during one check: ~4.68 GiB of ~31 GiB (ComfyUI used CPU offload).
- **No OOM** in any run; all 25 sampling steps completed.

Headline: a 1024×1024 Qwen-Image-2.1 generation lands around **5 minutes** on free
Kaggle hardware, first image of a session included.

## Repository layout

```
kaggle/
  00_environment_diagnostic/            GPU/CUDA sanity check on 2 x T4 (no downloads)
  01_qwen_image_comfy/                  headless Qwen-Image-2.1 t2i, pinned ComfyUI + official workflow
  02_comfyui_tunnel_test/               ComfyUI + Cloudflare tunnel smoke test (no model)
  07_qwen_comfyui_quick_interactive/    interactive UI: t2i + image-edit, graceful session timer
  08_qwen_models_dataset/               builds the ~17 GB Kaggle Dataset from Hugging Face
docs/
  RUNBOOK.md                            verified commands and pins
  RESULTS.md                            measurements and notes
results/
  00_environment_diagnostic/            JSON report + raw log
  01_qwen_image_comfy/                  1024x1024 PNG + JSON report
```

Each `kaggle/*` folder is a Kaggle *kernel* source: a notebook plus a
`kernel-metadata.json` that `kaggle kernels push` understands.

## How to run

Prerequisites: a Kaggle account, the
[Kaggle CLI](https://github.com/Kaggle/kaggle-api), and `kaggle auth login` done once.
Enable **GPU** and **Internet** on the kernels.

1. **Set your Kaggle username.** In every `kaggle/*/kernel-metadata.json`, replace
   `YOUR_KAGGLE_USERNAME` in `id` with your own username (kernels are pushed under your
   account). Do the same in the `dataset_sources` of notebook `07` after you create the
   model Dataset.

2. **Build the model Dataset once** (CPU-only, Internet on). This downloads the three
   INT8 files into the kernel output:

   ```bash
   kaggle kernels push -p kaggle/08_qwen_models_dataset
   ```

   Then create a Kaggle **Dataset** from that output, e.g.
   `<your-user>/qwen-image-21-int8-models`, and attach it to notebook `07`.

3. **Baseline headless text-to-image** (GPU on):

   ```bash
   kaggle kernels push -p kaggle/01_qwen_image_comfy -t 3600
   ```

4. **Interactive ComfyUI in the browser** (GPU on). Set `SESSION_LIMIT_MINUTES` in the
   first cell, then:

   ```bash
   kaggle kernels push -p kaggle/07_qwen_comfyui_quick_interactive -t 10800
   ```

   Watch the live log for `COMFYUI IS READY` and `TUNNEL_URL=...`, open the URL, load the
   bundled *Text to Image* / *Image Edit* workflows, generate, then stop the notebook.

## Model caching and the Hugging Face fallback

Downloading ~17 GB every session is wasteful, so notebook `07` resolves the weights
robustly instead of trusting one hard-coded path:

1. It **recursively searches `/kaggle/input`** for a folder that contains all three model
   files at full size, so any mount layout works — `/kaggle/input/<name>/`,
   `/kaggle/input/datasets/<user>/<name>/`, `/kaggle/input/<name>/<name>/`.
2. If found, the weights are read directly from the read-only mount
   (`model_source = kaggle_dataset`) — no copy, no download.
3. Otherwise it builds a writable `ComfyUI/models` root and downloads the pinned files from
   `Comfy-Org/Qwen-Image-2.1` (`model_source = huggingface_fallback`). Any valid file still
   present under `/kaggle/input` is **reused via symlink** instead of being re-downloaded,
   so even a partially broken dataset does not force a full 17 GB fetch.
4. Every branch ends with a **minimum-size check** per file.

The run records which path was taken in
`/kaggle/working/interactive_test_status.json` (`model_source`, `models_root`,
`model_fallback_details`) — the fastest way to tell "the dataset did not mount" from
"Kaggle was just slow", and to see how many files were reused vs downloaded.

### Reading the startup log

The notebook prints diagnostics *before* it touches the filesystem:

```
BOOT: notebook code started
DATASET SCAN: scanning /kaggle/input (depth-limited 0..5) ...
DATASET SCAN: done in 0.8s -> /kaggle/input/qwen-image-21-int8-models
```

`BOOT` proves the notebook is executing, and `DATASET SCAN` reports how long model
discovery took. If `BOOT` never appears, the stall is on Kaggle's side (worker boot /
dataset attach) rather than in this code — a useful distinction when a run sits at zero log
output.

The search is depth-limited (0..5) rather than an unbounded walk: `/kaggle/input` can be a
slow FUSE mount, and an unbounded `rglob` would walk every attached dataset before the
notebook prints anything.

**If attaching the Dataset makes startup slower than downloading,** drop `dataset_sources`
from `kernel-metadata.json`. The notebook then finds no dataset and falls back to Hugging
Face automatically (`model_source = huggingface_fallback`) — on a fast link that can be the
quicker option. Both paths are measured and reported.

## Multiple reference images (image-edit)

The official *Qwen Image 2.1 - Image Edit* workflow ships with two `LoadImage` nodes wired
into `image_1` / `image_2` and **eight image slots in total** (Qwen-Image-2.1 accepts up to
10 references). Load the workflow in the ComfyUI UI and attach the images you need — no
workflow editing required.

## Security notes

- A **Cloudflare Quick Tunnel URL is public and unauthenticated**. Anyone who has the URL
  can use your ComfyUI instance and burn your GPU quota. Do not upload sensitive images, do
  not share the URL, and stop the notebook when you are done.
- Quick Tunnel is for temporary tests. For longer sessions, use a **named** Cloudflare
  tunnel with **Cloudflare Access** (a documented next step).
- The ComfyUI server listens on `127.0.0.1` only; the tunnel is the single entry point.
- Never commit Kaggle credentials (`kaggle.json`, OAuth tokens), `.env`, or tunnel URLs.

## Limitations and failed experiments

Honest engineering log — things that did **not** make the cut:

- **GGUF quantisation** of the diffusion model was evaluated as an alternative to INT8;
  it lowers the footprint but is slower per step on T4 for this workload, so the official
  INT8 weights were kept.
- **Video (Wan 2.2 5B)** ran end-to-end through the same stack but took ~38 minutes per clip
  with poor quality, so it was dropped.
- Kaggle Notebooks are a **batch** environment: a long-lived interactive session works, but
  there is no watchdog yet for a "process alive / tunnel dead" state.

## Licence and attribution

- **This repository's own code and docs:** MIT (see [`LICENSE`](LICENSE)).
- **Weights:** [Qwen/Qwen-Image-2.1](https://huggingface.co/Qwen/Qwen-Image-2.1) is released
  under the **Qwen Research License** (non-commercial). Weights are **not** redistributed
  here; the kernels download them under your own agreement with that licence.
- **ComfyUI:** [Comfy-Org/ComfyUI](https://github.com/Comfy-Org/ComfyUI), **GPL-3.0** — cloned
  at the pinned commit, not vendored.
- **Workflow templates:** [Comfy-Org/workflow_templates](https://github.com/Comfy-Org/workflow_templates).
- **cloudflared:** [cloudflare/cloudflared](https://github.com/cloudflare/cloudflared), **Apache-2.0**.

## Related work

The general idea of running ComfyUI on free Kaggle/Colab GPUs exists elsewhere; this repo
aims to be the reproducible, INT8, dataset-cached variant. Related public projects:
[`chandan11248/qwen-image-21-t4`](https://github.com/chandan11248/qwen-image-21-t4)
(Qwen-Image-2.1 GGUF on Kaggle T4 with a mini-UI) and
[`kayas881/comfyui-kaggle`](https://github.com/kayas881/comfyui-kaggle) (generic
ComfyUI-on-Kaggle runner).


