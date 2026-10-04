# Results

All numbers below come from real runs on Kaggle's free 2 × Tesla T4 (15 GB VRAM each).

## Environment

- Python 3.12.13, PyTorch 2.10.0+cu128, CUDA 12.8, driver 580.159.04.
- 2 × Tesla T4 (15 636 037 632 bytes each); CUDA smoke test returned `passed: true`.
- Internet reachable from the notebook: HTTP 200.

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

## Notes

- These are single-run samples, not averages. Deterministic timing on shared Kaggle
  infrastructure is not possible; treat them as ballpark figures.
- Fresh repeat runs (text-to-image, image-edit, multi-reference) are planned and will be
  added here.
