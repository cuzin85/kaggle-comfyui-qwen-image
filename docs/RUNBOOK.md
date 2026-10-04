# Runbook

Verified commands, pins and procedures for running Qwen-Image-2.1 on Kaggle.

## Pins

| Component | Pin |
|---|---|
| ComfyUI | commit `93810483a4739a1588236919a3128d3070244146` |
| ComfyUI workflow templates | commit `a7acaf8cee9ccccdaf4b26ce04a9fced4c4d13de` |
| Qwen-Image-2.1 weights | repo `Comfy-Org/Qwen-Image-2.1` |
| cloudflared | release `2026.9.3` (linux-amd64) |

## Weight files

| Folder | File | Bytes |
|---|---|---|
| `diffusion_models` | `qwen_image_2.1_int8_convrot.safetensors` | 7 256 783 064 |
| `text_encoders` | `qwen3vl_8b_int8_convrot.safetensors` | 9 350 798 360 |
| `vae` | `qwen_image_2.1_vae_bf16.safetensors` | 675 509 688 |

Total: 17 283 091 766 bytes (~17.3 GB).

## Model resolution (notebook 07)

**By default `dataset_sources` is empty**, so this notebook downloads the weights from
Hugging Face on every run. Attaching a model Dataset was measured to be *slower* on
average and to hang occasionally before the notebook even starts — see "Model weights: why
the Dataset is **not** attached" in the README.

If a Dataset *is* attached, `find_model_root()` searches `/kaggle/input` (depth-limited
0..5) for a folder holding all three model files at full size, so any mount layout works.
If none is found, the notebook builds a writable `ComfyUI/models` root, reuses any valid
file found under `/kaggle/input` via symlink, and downloads the rest from
`Comfy-Org/Qwen-Image-2.1`. Every file is then checked against `MINIMUM_SIZES`. The chosen
path is recorded as `model_source` (and `model_fallback_details`) in
`interactive_test_status.json`.

## Startup diagnostics

The first two things printed by notebook `07`, in order:

```
BOOT: notebook code started
DATASET SCAN: done in <seconds> -> <root or None>
```

- `BOOT` appears before any filesystem work, so it proves the notebook actually started.
- `DATASET SCAN` reports the elapsed time of model discovery.
- No `BOOT` line at all ⇒ the delay is upstream (worker boot or dataset attach), not in
  the notebook. The Kaggle `/kaggle/input` mount can be slow, and attaching a ~15 GB model
  Dataset is the usual suspect.

`find_model_root()` / `find_model_file()` use depth-limited patterns (`*` … `*/*/*/*/*`,
0..5) rather than a full `rglob`, because an unbounded walk of `/kaggle/input` executes
before the first print and can stall silently.

## Multiple references

The official image-edit workflow has two `LoadImage` nodes (`image_1`, `image_2`) and eight
image slots in total; attach more images in the ComfyUI UI as needed.

## Kaggle CLI

```bash
kaggle kernels push -p kaggle/<folder> [-t <seconds>]
kaggle kernels status <user>/<slug>
kaggle kernels logs <user>/<slug>           # final log
kaggle kernels logs --follow <user>/<slug>  # live log
```

`-t` is the Kaggle safety timeout. The interactive notebooks additionally stop themselves
through `SESSION_LIMIT_MINUTES`.

## ComfyUI server flags

```bash
python main.py --listen 127.0.0.1 --port 8188 \
  --disable-auto-launch --lowvram --force-fp16 \
  --output-directory <dir>
```

## Session timer

`SESSION_LIMIT_MINUTES` in the first cell of notebook `07` controls how long the last cell
keeps ComfyUI and the tunnel alive. `0` disables it. The countdown starts when the hold cell
starts, so model downloads are not counted against it.

## Safety

- Do not commit `kaggle.json`, OAuth tokens, `.env`, or tunnel URLs.
- A Quick Tunnel URL is public; treat it as a temporary secret.
