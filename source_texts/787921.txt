# 3DGS Frame Selection Benchmark

Benchmark how frame-to-frame similarity filtering changes Gaussian splatting reconstructions. The first target is Nerfstudio Splatfacto on open-license videos, comparing raw all-frame input against NCC, MSE, and SSIM frame rejection.

This repo is the broader home for the frame-selection workload. The first NCC-only run lives in [DrStrangel0ve/3d-gaussian-splatting-ncc](https://github.com/DrStrangel0ve/3d-gaussian-splatting-ncc).

## Workload

- Extract every decoded video frame by default.
- Compare each candidate frame with the previous kept frame.
- Drop the candidate when the selected metric says the frames are too similar.
- Run `ns-process-data` and `ns-train splatfacto` for each threshold.
- Track PSNR, SSIM, LPIPS, registered COLMAP images, kept-frame count, train time, and exported `.ply` size.

## Starting Matrix

| Variant | Metric | Threshold meaning |
| --- | --- | --- |
| `keep_all` | none | keep every decoded frame |
| `ncc_0_99` | NCC | drop when NCC >= 0.99 |
| `ncc_0_95` | NCC | drop when NCC >= 0.95 |
| `ncc_0_90` | NCC | drop when NCC >= 0.90 |
| `ncc_0_75` | NCC | drop when NCC >= 0.75 |
| `ssim_*` | SSIM | drop when SSIM >= threshold |
| `mse_*` | MSE | drop when MSE <= threshold |

## Populated Contents

This repo is seeded with the open full-frame NCC run from the Nerfstudio project:

- `configs/open_videos_full.json`: three open-license source videos.
- `scripts/run_with_docker.sh`: WSL/Docker wrapper for Nerfstudio.
- `scripts/run_inside_nerfstudio.sh`: Splatfacto processing, training, eval, and export loop.
- `scripts/prepare_ncc_variants.py`: exact NCC variant builder used for the current results.
- `scripts/filter_frames.py`: standalone NCC, MSE, and SSIM frame filtering tool.
- `results/open_video_full_frame_summary.md`: current PSNR/SSIM/LPIPS table for full-frame inputs.
- `results/open_video_full_frame_report.md`: run notes and failure details.

## Current Open-Video Results

| Scene | Variant | Kept frames | Status | PSNR | SSIM | LPIPS |
| --- | --- | ---: | --- | ---: | ---: | ---: |
| `open_close_scanparayzo` | `ncc_0_90` | 31 | ok | 18.3222 | 0.900717 | 0.205718 |
| `open_mid_baios_statue` | `ncc_0_95` | 135 | ok | 20.7819 | 0.622283 | 0.704930 |
| `open_mid_baios_statue` | `ncc_0_90` | 97 | ok | 20.6879 | 0.596844 | 0.710817 |

The other open full-frame variants are retained in the summary table even when COLMAP or evaluation failed, because those failures are part of the workload.

## Quick Start

```bash
python scripts/filter_frames.py --input data/videos/example.mp4 --output frames/example/keep_all --metric keep_all
python scripts/filter_frames.py --input data/videos/example.mp4 --output frames/example/ncc_0_95 --metric ncc --threshold 0.95
python scripts/filter_frames.py --input data/videos/example.mp4 --output frames/example/ssim_0_95 --metric ssim --threshold 0.95
```

Then process and train with Nerfstudio, ideally from WSL or a CUDA-ready Linux environment:

```bash
ns-process-data images --data frames/example/ncc_0_95 --output-dir processed/example/ncc_0_95
ns-train splatfacto --data processed/example/ncc_0_95
```

For the populated open-video run:

```bash
python scripts/download_manifest_videos.py --manifest configs/open_videos_full.json
MANIFEST=configs/open_videos_full.json FPS=0 MAX_FRAMES=0 WIDTH=640 ./scripts/run_with_docker.sh
```

## Repository Shape

- `configs/workload_matrix.json` defines the intended threshold sweep.
- `scripts/filter_frames.py` is a metric-based frame keeper for full-frame video inputs.
- `scripts/download_manifest_videos.py` downloads open videos listed by the manifest.
- `results/` is reserved for summary tables and metric JSON files.
- `exports/` can hold generated Gaussian `.ply` files when an experiment is published.

## License

MIT for code and project scaffolding. Dataset and video licenses should be tracked per source.
