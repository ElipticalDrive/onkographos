# NeRF Volume Rendering (PyTorch)

A PyTorch implementation of Neural Radiance Fields (NeRF) trained on the Blender "lego" synthetic scene. The repository fills in the core of the NeRF rendering pipeline on top of an existing NeRF training framework: generating camera rays for every pixel, stratified sampling of points along each ray, and differentiable volume rendering that turns per-point density and color into pixel color, depth and opacity. Coarse-only and coarse-plus-fine (hierarchical sampling) configurations are provided, along with unit tests for the rendering math and a script that assembles training-progress figures.

## Implementation

Core rendering functions (`nerf_utils.py`, `run_nerf.py`):

- **`get_rays(H, W, K, c2w)`**: builds a pixel grid with `torch.meshgrid`, converts pixels to camera-space directions with the intrinsics `K` (OpenGL convention, camera looking down -z), rotates them into world space with the camera-to-world matrix, and returns per-pixel ray origins and (unnormalized) directions.
- **`sample_points(rays_o, rays_d, near, far, N_samples)`**: stratified sampling. Splits `[near, far]` into evenly spaced bins and draws one uniform random depth inside each bin, then returns the depths and 3D sample points.
- **`compute_bin_dists(z_vals, rays_d)`**: distances between consecutive samples, with a large last interval (1e10), scaled by the ray direction norm to get metric distances.
- **`compute_alpha(sigma, dists)`**: `alpha = 1 - exp(-relu(sigma) * delta)`.
- **`compute_weights(alpha)`**: transmittance as an exclusive cumulative product of `(1 - alpha)`, and weights `T_i * alpha_i`.
- **`volume_rendering(raw, z_vals, rays_d, white_bkgd)`**: applies a sigmoid to the predicted color, then computes the RGB map, expected depth and accumulated opacity as weighted sums, with optional white-background compositing.

Retained from the base framework: the 8-layer, 256-wide MLP with a skip connection and view-direction branch, sinusoidal positional encoding (10 frequencies for positions, 4 for directions), hierarchical importance sampling (`sample_pdf`), batched rendering, checkpointing, and loaders for Blender, LLFF, DeepVoxels and LINEMOD data.

## Training setup

| Config | Coarse samples | Fine samples | Notes |
|---|:---:|:---:|---|
| `configs/lego_coarse.txt` | 64 | 0 | single coarse network |
| `configs/lego_fine.txt` | 64 | 128 | coarse + fine networks with hierarchical sampling |

Both configs use the Blender `lego` scene at half resolution (`downscale_factor = 2`), a white background, 1,024 random rays per step, center cropping for the first 500 iterations, and an exponentially decaying learning rate starting at 5e-4. Training runs for 22,000 iterations. The loss is the photometric MSE (summed over the coarse and fine outputs when both exist).

During training, `run_nerf.py`:

- logs loss and train PSNR (coarse and fine separately) to TensorBoard,
- renders three fixed test views (positions 11, 17 and 22 of the test split after `testskip` subsampling) at iterations 500, 2,000, 5,000, 10,000 and 20,000, saving RGB and depth images and logging the mean test PSNR,
- saves a checkpoint every 5,000 iterations.

No trained weights or rendered images are committed; they are produced in `logs/<expname>/`.

## Repository structure

```
.
├── run_nerf.py         training / rendering entry point, sampling and volume rendering
├── nerf_utils.py       positional encoding, NeRF MLP, get_rays, NDC rays, sample_pdf
├── unit_test.py        tests for get_rays, compute_bin_dists, compute_alpha, compute_weights
├── image_gen.py        grid of test-view RGB and depth renders across training iterations
├── load_blender.py     Blender synthetic loader (with downscaling)
├── load_llff.py, load_deepvoxels.py, load_LINEMOD.py
├── configs/
│   ├── lego_coarse.txt
│   └── lego_fine.txt
├── requirements.txt
└── run.sh              SLURM job: run the unit tests, then train with lego_fine.txt
```

## Setup

PyTorch is not pinned in `requirements.txt` (the `torch` lines are commented out). Install a CUDA build of PyTorch for your system first, then:

```bash
pip install -r requirements.txt
```

A CUDA GPU is required: `run_nerf.py` sets the default tensor type to `torch.cuda.FloatTensor`.

## Data

Download the NeRF synthetic Blender dataset from the original NeRF release and place the lego scene at:

```
data/nerf_synthetic/lego/
├── transforms_train.json
├── transforms_val.json
├── transforms_test.json
├── train/  val/  test/
```

The path is set by `datadir` in the config files.

## Usage

Run the unit tests for the rendering functions:

```bash
python unit_test.py --test all
python unit_test.py --test get_rays    # or compute_bin_dists, compute_alpha, compute_weights
```

Train:

```bash
python run_nerf.py --config configs/lego_coarse.txt
python run_nerf.py --config configs/lego_fine.txt
```

Monitor training:

```bash
tensorboard --logdir logs/blender_lego_fine/tensorboard
```

Render the test views from the latest checkpoint (saves per-view RGB and depth images and a `video.mp4` under `logs/<expname>/`):

```bash
python run_nerf.py --config configs/lego_fine.txt --reload --render_only --render_test
```

Build a figure of RGB and depth renders for the test views across iterations:

```bash
python image_gen.py --base_path logs/blender_lego_fine \
    --iterations 500 2000 5000 10000 20000 --test_images 0 1 2 \
    --output rendering_comparison.png
```

## Acknowledgements

- NeRF: Mildenhall et al., "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis", ECCV 2020 ([bmild/nerf](https://github.com/bmild/nerf)).
- The training framework, network definition, hierarchical sampling and data loaders come from the PyTorch NeRF port [yenchenlin/nerf-pytorch](https://github.com/yenchenlin/nerf-pytorch). The code is distributed under the MIT License; see [LICENSE](LICENSE).
