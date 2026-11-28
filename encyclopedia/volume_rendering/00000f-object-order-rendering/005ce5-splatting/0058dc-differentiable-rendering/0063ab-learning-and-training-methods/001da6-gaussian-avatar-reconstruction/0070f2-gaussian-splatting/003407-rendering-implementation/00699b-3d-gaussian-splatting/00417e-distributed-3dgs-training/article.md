# Distributed 3DGS Training

Also called: Grendel Distributed Training, Grendel-GS Training Pipeline, HPC-scale distributed 3DGS

## Overview
Distributed 3DGS Training partitions 3D Gaussian Splatting parameters and datasets across multiple GPUs or HPC nodes to overcome single-GPU memory limits, enabling the training of high-resolution and large-scale scenes (e.g., >100M Gaussians) [S1][S2, S3, unofficial]. It sets itself apart from single-GPU baselines by preserving the original 3DGS algorithm and convergence characteristics while using spatial partitioning with ghost cells for boundary consistency and sparse all-to-all communication to distribute the rasterization workload [S1][S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A COLMAP dataset (images, camera poses) partitioned across nodes [S3, unofficial].
- **Output:** A global 3D Gaussian model (positions, covariances, opacities, SH coefficients) merged from local node states [S1][S2, S3, unofficial].

**Data Structures:**
- **Local Gaussians:** A subset of the global Gaussian set assigned to each node [S2, S3, unofficial].
- **Ghost Cells:** Boundary regions where Gaussians from neighboring nodes are replicated to ensure smooth transitions during local rasterization [S3, unofficial].
- **Background Masks:** Binary masks applied during local rendering to eliminate artifacts at the boundaries of the partition [S3, unofficial].

**Numbered Steps:**
1. Partition datasets across nodes [S3, unofficial].
2. Partition 3DGS parameters across multiple GPUs [S2, unofficial].
3. Use ghost cells at boundaries for smooth transitions and background masks to eliminate rendering artifacts [S3, unofficial].
4. Train Gaussian splats independently per node [S3, unofficial].
5. Perform sparse all-to-all communication to transfer each required Gaussian to a pixel partition [S2, unofficial].
6. Merge for global rendering [S3, unofficial].
7. Support batched training using multiple views [S2, unofficial].
8. Evaluate scaling rules by starting with a batch size of 1, resetting Adam optimizer states, and continuing training with different batch sizes [S1].

**Formulas:**
- **Learning Rate Scaling:** The learning rate scales by the square root of the batch size to maintain update cosine similarity and magnitude across different training batch sizes [S1].
  $$ \eta_{new} = \eta_{base} \times \sqrt{\frac{BS_{new}}{BS_{base}}} $$
  Where $BS$ is the batch size (number of camera views) [S1].
- **Loss Function:** Standard 3DGS loss combining L1 and SSIM, where $\lambda$ controls SSIM influence [S1][S2, unofficial].
  $$ L = (1 - \lambda) L_1 + \lambda L_{SSIM} $$
  Where $\lambda$ defaults to 0.2 [S1][S2, unofficial].

**Parameters:**
- `--bsz`: Batch size (number of camera views) in single step training. Default: 1 [S2, unofficial].
- `--lr_scale_mode`: Mode of scaling learning rate given larger batch size. Default: `sqrt` [S2, unofficial].
- `--iterations`: Number of total iterations to train for. Default: 30,000 [S2, unofficial].
- `--preload_dataset_to_gpu`: Save all ground truth images from the dataset in GPU memory rather than loading each image on-the-fly. Warning: May cause OOM on large datasets [S2, unofficial].
- `--sh_degree`: Order of spherical harmonics to be used. Default: 3 [S2, unofficial].
- `--feature_lr`: Spherical harmonics features learning rate. Default: 0.0025 [S2, unofficial].
- `--position_lr_init`: Initial 3D position learning rate. Default: 0.00016 [S2, unofficial].
- `--position_lr_final`: Final 3D position learning rate. Default: 0.0000016 [S2, unofficial].
- `--densify_from_iter`: Iteration where densification starts. Default: 500 [S2, unofficial].
- `--densify_until_iter`: Iteration where densification stops. Default: 15,000 [S2, unofficial].
- `--densify_grad_threshold`: Gradient norm threshold to initiate densification. Default: 0.0002 [S2, unofficial].
- `--densification_interval`: How frequently to densify. Default: 100 [S2, unofficial].
- `--opacity_reset_interval`: How frequently to reset opacity. Default: 3,000 [S2, unofficial].
- `--lambda_dssim`: Influence of SSIM on total loss from 0 to 1. Default: 0.2 [S2, unofficial].
- `--white_background`: Use white background instead of black. Default: False [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.distributed as dist

def distributed_train(args):
    # Initialize distributed process group
    dist.init_process_group(backend=args.backend)
    local_rank = dist.get_rank()
    world_size = dist.get_world_size()

    # Initialize Gaussian parameters and optimizer
    # Standard 3DGS parameter initialization (positions, covariances, opacities, SH coefficients)
    gaussians = torch.nn.ParameterDict({
        'means': torch.nn.Parameter(torch.zeros(3, dtype=torch.float32)),
        'log_scales': torch.nn.Parameter(torch.zeros(3, dtype=torch.float32)),
        'opacities': torch.nn.Parameter(torch.zeros(1, dtype=torch.float32)),
        'sh_coeffs': torch.nn.Parameter(torch.zeros(3, 16, dtype=torch.float32))
    })
    optimizer = torch.optim.Adam(gaussians.parameters(), lr=args.position_lr_init)

    # Determine background color
    bg_color = torch.tensor([1.0, 1.0, 1.0], dtype=torch.float32, device="cuda") if args.white_background else torch.tensor([0.0, 0.0, 0.0], dtype=torch.float32, device="cuda")

    # Training loop
    for iteration in range(args.iterations + 1):
        # 1. Load batch of images assigned to this node
        cameras = scene.getTrainCameras()
        local_cameras = [c for i, c in enumerate(cameras) if i % world_size == local_rank]
        if not local_cameras:
            continue

        # 2. Render local Gaussians with background mask
        # Standard differentiable Gaussian rasterization with boundary masks
        rendered_image = rasterize_local(gaussians, local_cameras, bg_color, args.sh_degree)

        # 3. Compute Loss (L1 + SSIM)
        gt_image = local_cameras[0].original_image
        loss = compute_l1_ssim(rendered_image, gt_image, args.lambda_dssim)

        # 4. Backpropagate
        loss.backward()

        # 5. Aggregate Gradients across nodes
        for param in gaussians.parameters():
            if param.grad is not None:
                dist.all_reduce(param.grad, op=dist.ReduceOp.SUM, group=dist.group.WORLD)
                param.grad /= world_size

        # 6. Update Parameters
        optimizer.step()
        optimizer.zero_grad()

        # 7. Densification/Pruning (Local)
        # Standard 3DGS densification based on gradient thresholds and opacity
        if iteration % args.densification_interval == 0:
            densify_and_prune_local(gaussians, iteration, args)

        # 8. Checkpoint and Evaluation
        if iteration in args.save_iterations:
            save_model_checkpoint(gaussians, args.model_path, iteration)
        if iteration in args.test_iterations:
            evaluate_metrics(gaussians, scene.getTestCameras())

    dist.destroy_process_group()
```

**Edge Cases and Pitfalls:**
- **Single-GPU OOM:** Large scenes like Rubble cannot be rendered with a single GPU regardless of batch size due to memory requirements; at least 4 GPUs are needed to avoid OOM [S1].
- **Preload OOM:** Setting `--preload_dataset_to_gpu` on large datasets can cause Out-Of-Memory errors [S2, unofficial].
- **Load Balancing:** Uneven distribution of scene complexity or camera views can lead to load imbalance, and moderate-sized datasets show limited scaling gains when insufficient to fully utilize additional nodes [S3, unofficial].
- **Boundary Artifacts:** Without ghost cells and background masks, rendering artifacts appear at partition boundaries [S3, unofficial].

## Subtopics

### RetinaGS
A distributed training strategy for 3DGS that retains the original algorithm but uses a distinct parallelization approach where each GPU renders the entire image using its local partition of Gaussians, with outputs merged subsequently [S1]. Redundant computations occur because many GPUs render pixels beyond the opacity saturation depth unnecessarily [S1].

## Operating characteristics
- **Time Complexity:** Scales with the number of nodes and Gaussians, but communication overhead and load imbalance reduce efficiency [S2, S3, unofficial].
- **Memory Complexity:** Memory per GPU is reduced by a factor of $N$ (number of nodes), allowing training of scenes that exceed single-GPU memory limits (e.g., ~11.2M Gaussians on A100) [S2, S3, unofficial].
- **Measured Performance:**
  - Rubble scene (35M Gaussians): Throughput increased from 5.55 images per second (4 GPUs, batch size 1) to 38.03 images per second (32 GPUs, batch size 64) [S1].
  - Train scene (980×545 resolution): A 16-GPU setup with batch size 32 completed training on 30K images in 2 minutes and 42.97 seconds [S1].
  - Mip-NeRF 360, Tanks & Temple, Deep Blending: Using 4 GPUs and a batch size of 4 achieved a 3x to 4x speedup without PSNR degradation [S1].
  - Richtmyer-Meshkov dataset (106.7M Gaussians, 5.3 GB, 2048×2048): 3.1× speedup scaling from 4 to 8 nodes, PSNR 30.04, SSIM 0.97, LPIPS 0.019 [S3, unofficial].
  - Rayleigh-Taylor dataset (18.2M Gaussians): 1.4× speedup from 2 to 4 nodes [S3, unofficial].
- **Strengths:** Enables training of very large scenes; reduces per-GPU memory usage; maintains reconstruction quality and convergence characteristics of the original algorithm [S1][S2, S3, unofficial].
- **Limitations:** Scaling gains are limited for moderate-sized datasets; load balancing for uneven distributions remains a challenge; communication overhead increases with node count [S2, S3, unofficial].
- **When to Use:** Large-scale scenes (e.g., scientific volumes, high-resolution urban scans) that exceed single-GPU memory constraints [S1][S3, unofficial].
- **When Not to Use:** Small to moderate scenes where single-GPU training is sufficient and faster due to lower communication overhead [S3, unofficial].

## Background
Originated from the need to scale 3D Gaussian Splatting to large scientific volumes and high-resolution scenes that exceed single-GPU memory constraints [S2, S3, unofficial]. The method was introduced by Zhao et al. in "On Scaling Up 3D Gaussian Splatting Training" (2024) [S1]. It builds on the original 3DGS framework [S1] and integrates distributed computing techniques from HPC, preserving the original algorithm's convergence characteristics unlike ADMM-based alternatives [S1].

## Sources
[S1] On Scaling Up 3D Gaussian Splatting Training, https://doi.org/10.1007/978-3-031-91989-3_2, fetched 2026-10-26
[S2] nyu-systems/Grendel-GS, https://github.com/nyu-systems/Grendel-GS, fetched 2026-10-27 (unofficial)
[S3] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
