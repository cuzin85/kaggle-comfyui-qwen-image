# Qwen-Image-2.1 on a free Kaggle GPU via ComfyUI

Run **Qwen-Image-2.1** (official INT8 weights) through **ComfyUI** on Kaggle's free
**2 × Tesla T4** — the ~17 GB of weights are fetched from Hugging Face in about 90 seconds —
and use the full ComfyUI **web UI in your browser** — text-to-image *and* image-edit —
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
2. **Opaque weight handling** — ~17 GB must be present every run, and nobody measures
   whether caching it in a Kaggle Dataset or re-downloading is actually faster.
3. **Headless only** — one prompt per run, no way to actually use the ComfyUI UI.

This project addresses all three: pinned versions, a *measured* answer to the weight-caching
question (short version: downloading beats attaching a Dataset — see below), and an
interactive browser session with a graceful session timer.

## Architecture

```
Kaggle notebook (2 x Tesla T4, 15 GB each)
        |
        +-- ComfyUI (main.py --lowvram --force-fp16)          :8188
        |       models <-  Hugging Face  (or a Kaggle Dataset, if you attach one)
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
  08_qwen_models_dataset/               optional: builds the ~17 GB Kaggle Dataset from HF
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
   account).

2. **Skip the model Dataset by default.** Notebook `07` ships with an empty
   `dataset_sources` and downloads the weights from Hugging Face on each run — measured
   *faster* and far more predictable than attaching a Dataset (numbers below). Notebook `08`
   is the optional builder if you still want to try it:

   ```bash
   kaggle kernels push -p kaggle/08_qwen_models_dataset   # optional
   ```

   If you build one and attach it yourself, add it to `dataset_sources` of notebook `07`.

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

## Model weights: why the Dataset is **not** attached

Every session needs ~17 GB of weights, and there are two ways to get them: attach a
**Kaggle Dataset** that already holds them, or **download from Hugging Face** on each run.
The Dataset looks obviously better — but it was not, and the numbers below come from real
runs of the same kernel on the same account.

| Configuration | First log line appears | Reaches `COMFYUI IS READY` |
|---|---|---|
| **Dataset attached**, good case | ~80 s | — |
| **Dataset attached**, bad case | 170 s, then nothing; abandoned after ~10 min | never |
| **No Dataset, weights downloaded from HF** | **10.1 s** | **~182 s** |

Without the Dataset the whole chain — clone ComfyUI, fetch `cloudflared`, **download all
17.28 GB from Hugging Face**, start the server, open the tunnel — completed in
**181.95 s** (`ready_at_seconds`). The weights themselves took roughly **87 s**
(`qwen_models_ready` at 120.88 s, after `cloudflared` at 34.04 s), i.e. **~200 MB/s**.

So attaching the Dataset:

- **sometimes** works and saves the download;
- **sometimes** hangs inside Kaggle's mount step **before the notebook even starts** — in
  those runs the notebook prints *nothing at all*, because its first line comes only after
  the container is up;
- is not reproducible: 80 s vs 10+ min on the same kernel is a bad trade, and the bad case
  costs GPU time and nerves with no output to show for it.

**Decision: do not attach the model Dataset by default.** `dataset_sources` in
[`kaggle/07_qwen_comfyui_quick_interactive/kernel-metadata.json`](kaggle/07_qwen_comfyui_quick_interactive/kernel-metadata.json)
is left empty and the notebook downloads from `Comfy-Org/Qwen-Image-2.1` — fast and
predictable. The Dataset stays an *optional* fast path if you have confirmed that your own
account mounts it reliably.

Attaching one is still safe, because the notebook resolves whatever is mounted robustly:

1. It **searches `/kaggle/input`** (depth-limited 0..5) for a folder containing all three
   model files at full size — `/kaggle/input/<name>/`,
   `/kaggle/input/datasets/<user>/<name>/`, `/kaggle/input/<name>/<name>/`.
2. If found, the weights are read straight from the read-only mount
   (`model_source = kaggle_dataset`) — no copy, no download.
3. Otherwise it builds a writable `ComfyUI/models` root and downloads the pinned files from
   `Comfy-Org/Qwen-Image-2.1` (`model_source = huggingface_fallback`). Any valid file still
   present under `/kaggle/input` is **reused via symlink**, so a partially broken Dataset
   does not force a full 17 GB fetch.
4. Every branch ends with a **minimum-size check** per file.

The run records which path was taken in
`/kaggle/working/interactive_test_status.json` (`model_source`, `models_root`,
`model_fallback_details`) — the fastest way to tell "the Dataset did not mount" from
"Kaggle was just slow", and to see how many files were reused vs downloaded.

### Reading the startup log

The notebook prints diagnostics *before* it touches the filesystem:

```
BOOT: notebook code started
DATASET SCAN: scanning /kaggle/input (depth-limited 0..5) ...
DATASET SCAN: done in 0.0s -> None
```

`BOOT` proves the notebook is executing, and `DATASET SCAN` reports how long model
discovery took. If `BOOT` never appears, the stall is on Kaggle's side (worker boot /
dataset attach) rather than in this code — a useful distinction when a run sits at zero log
output.

The search is depth-limited (0..5) rather than an unbounded walk: `/kaggle/input` can be a
slow FUSE mount, and an unbounded `rglob` would walk every attached dataset before the
notebook prints anything.

`dataset_sources` ships **empty**, so `model_source` is `huggingface_fallback` by default.
If you attach a Dataset and it turns out to be slower, drop `dataset_sources` again — both
paths are measured and reported, so the choice is always visible in the log.

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
aims to be the reproducible, INT8, fully measured variant. Related public projects:
[`chandan11248/qwen-image-21-t4`](https://github.com/chandan11248/qwen-image-21-t4)
(Qwen-Image-2.1 GGUF on Kaggle T4 with a mini-UI) and
[`kayas881/comfyui-kaggle`](https://github.com/kayas881/comfyui-kaggle) (generic
ComfyUI-on-Kaggle runner).


