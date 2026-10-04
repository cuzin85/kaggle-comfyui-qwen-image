# Results

All numbers below come from real runs on Kaggle's free 2 × Tesla T4 (15 GB VRAM each).

## Environment

- Python 3.12.13, PyTorch 2.10.0+cu128, CUDA 12.8, driver 580.159.04.
- 2 × Tesla T4 (15 636 037 632 bytes each); CUDA smoke test returned `passed: true`.
- Internet reachable from the notebook: HTTP 200.

## Startup: Dataset attached vs download from Hugging Face

Notebook `07`, same kernel, same account (04.10.2026). This is the experiment that decided
the default: **the model Dataset is no longer attached.**

| Configuration | First log line appears | Reaches `COMFYUI IS READY` |
|---|---|---|
| Dataset attached, good case | ~80 s | — |
| Dataset attached, bad case | 170 s, then no output; abandoned after ~10 min | never |
| **No Dataset, weights from Hugging Face** | **10.1 s** | **181.95 s** |

Timeline of the no-Dataset run (`elapsed_seconds` measured from notebook start):

| Stage | elapsed_seconds |
|---|---|
| `BOOT: notebook code started` | 10.1 s (from kernel session start) |
| `DATASET SCAN: done in 0.0s -> None` | 0.02 |
| `cloudflared` downloaded (40 122 749 bytes) | 34.04 |
| `qwen_models_ready` — all three weights fetched | 120.88 |
| `downloading_official_ui_workflows` | 120.92 |
| `tunnel_ready_at_seconds` | 179.51 |
| `ready_at_seconds` | **181.95** |

- Weights: 17 283 091 766 bytes in ≈ 87 s (34.04 → 120.88) ≈ **200 MB/s**.
- `model_source = huggingface_fallback`, all three files reported `downloaded`.
- The tunnel answered **HTTP 200 in 0.54 s** immediately after ready.

## Text-to-image

| Run | Resolution | Steps | Time |
|---|---|---|---|
| `01`, headless, official workflow | 1024×1024 | 25 | workflow 295.84 s; notebook 350.44 s |
| `07`, interactive UI | 1024×1024 | — | 286.6 s (log) / 261 s (observed) |

## Image-edit (interactive UI, `07`)

| Run | Time |
|---|---|
| First run | 464.4 s |
| Run with saved output | 609.69 s |

## Memory

- Image-edit: GPU 0 ≈ 10.28 GiB of 14.56 GiB; GPU 1 almost idle.
- Free system RAM during a check: ≈ 4.68 GiB of ~31 GiB (CPU offload active).
- No OOM in any recorded run.

## Repeat run, 04.10.2026 (notebook `07`, no Dataset attached)

Same kernel, same account, code byte-identical to this repo (`sha256 960bfc46…06029`):
only `kernel-metadata.json` `id`/`title` differ (real username instead of the
`YOUR_KAGGLE_USERNAME` template placeholder). `dataset_sources` was empty, so
`model_source = huggingface_fallback`.

| What | Measured |
|---|---|
| `BOOT: notebook code started` | ~10 s after kernel session start |
| `DATASET SCAN: done in 0.0s -> None` | 0.02 s |
| `qwen_models_ready` (17 283 091 766 bytes from Hugging Face) | 120.88 s (run 1), **116.04 s** (run 2) |
| `tunnel_ready_at_seconds` | 179.51 s (run 1), **166.60 s** (run 2) |
| `ready_at_seconds` (`COMFYUI IS READY`) | 181.95 s (run 1), **168.96 s** (run 2) |
| Tunnel check right after ready | HTTP 200 in 0.54 s (run 1), 0.77 s (run 2) |
| **Text-to-image** → `Qwen_image_2.1_00001.png` | **309.07 s** (ComfyUI `Prompt executed`) |
| **Image-edit, 1 reference** → `Qwen_image_2.1_00002.png` | **506.12 s** |
| **Image-edit, 2 references** → `Qwen_image_2.1_00003.png` | **652.96 s** (ComfyUI display) |

Note on the third timing: the PNG was observed in `interactive_output`
(`GENERATION OUTPUT OBSERVED: Qwen_image_2.1_00003.png`) and ComfyUI displayed
`652.96 s`, but the matching `GENERATION PROMPT EXECUTED: 652.96 seconds` line had not
arrived through the kernels log stream before the session was stopped. The first two
timings coincide between the streamed log and the UI, so the UI number is recorded as the
source for the third one. The hold-loop regex that extracts `Prompt executed in N seconds`
deserves a closer look (tail latency in `comfyui_interactive.log` vs stream delay).

## Notes

- These are single-run samples, not averages. Deterministic timing on shared Kaggle
  infrastructure is not possible; treat them as ballpark figures.
- The Dataset-vs-download A/B above was likewise single-run per configuration, plus one
  repeated no-Dataset run (`qwen_models_ready` 120.88 s vs 116.04 s).
