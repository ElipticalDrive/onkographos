# VoxelTTO Training Configuration
Also called: VoxelTTO Training Details

## Overview
VoxelTTO training configures the optimization of a voxel-aligned feed-forward 3D Gaussian Splatting (3DGS) module initialized from a frozen Visual Foundation Model (VFM). It solves the problem of adapting large pre-trained vision models to specific 3D reconstruction tasks via test-time optimization, keeping the VFM parameters frozen while optimizing only the lightweight 3DGS module [S1, unofficial]. It differs from standard neural field training by leveraging pose supervision and a hybrid loss combining rendered depth/RGB with VFM-predicted depth alignment [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A sequence of 3–16 video frames resized to $224 \times 448$ pixels. Ground-truth depth maps and RGB images are required for supervision. Camera poses are assumed known for the test-time optimization phase [S1, unofficial].
- **Outputs:** Optimized parameters for the voxel-aligned feed-forward 3DGS module. The VFM weights remain frozen [S1, unofficial].

**Data Structures:**
- **Architecture:** DA3-GIANT with $L=40$ alternating-attention layers [S1, unofficial].
- **Parameters:** Total model size is 1.39B parameters. Only 75.3M parameters are trainable (the 3DGS module) [S1, unofficial].
- **Initialization:** The model is initialized with pretrained VFM weights [S1, unofficial].

**Loss Function:**
The total loss $L$ is a weighted sum of depth and RGB losses [S1, unofficial]:

$$
L = L_{depth} + L_{rgb} = \lambda_{\ell^1_d} L_{\ell^1_{depth}} + \lambda_{align} L_{align_{depth}} + \lambda_{\ell^1} L_{\ell^1_{rgb}} + \lambda_{ssim} L_{ssim_{rgb}}
$$

Where:
- $L_{\ell^1_{depth}}$: $\ell^1$ loss between rendered depth and ground-truth depth.
- $L_{align_{depth}}$: Alignment loss between rendered depth and VFM-predicted depth.
- $L_{\ell^1_{rgb}}$: $\ell^1$ loss between rendered RGB and ground-truth RGB.
- $L_{ssim_{rgb}}$: SSIM loss between rendered RGB and ground-truth RGB [S1, unofficial].

**Loss Weights:**
- $\lambda_{\ell^1_d} = 0.25$
- $\lambda_{align} = 0.25$
- $\lambda_{\ell^1} = 0.9$
- $\lambda_{ssim} = 1.0$ [S1, unofficial]

**Optimization Steps:**
1. **Data Sampling:** For each dataset, define a minimum and maximum frame gap. Randomly select an interval between the first and last frame within this range. Sample 3–16 frames from this interval [S1, unofficial].
2. **Preprocessing:** Resize input frames to $224 \times 448$ [S1, unofficial].
3. **Initialization:** Load pretrained DA3-GIANT weights. Freeze all VFM parameters. Initialize the voxel-aligned feed-forward 3DGS module [S1, unofficial].
4. **Optimizer Setup:** Use Adam optimizer. Set initial learning rate to $3 \times 10^{-6}$. Set minimum learning rate to $10^{-8}$ [S1, unofficial].
5. **Scheduling:** Apply cosine annealing schedule with a 2K warm-up period [S1, unofficial].
6. **Iteration:** Perform 30K optimization iterations [S1, unofficial].
7. **Loss Computation:** At each step, render depth and RGB from the current 3DGS representation. Compute $L_{depth}$ and $L_{rgb}$ using the defined weights. Backpropagate only through the trainable 3DGS parameters [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration
NUM_ITERATIONS = 30000
WARMUP_ITERATIONS = 2000
INITIAL_LR = 3e-6
MIN_LR = 1e-8
LAMBDA_L1_D = 0.25
LAMBDA_ALIGN = 0.25
LAMBDA_L1_RGB = 0.9
LAMBDA_SSIM = 1.0
FRAME_COUNT_MIN = 3
FRAME_COUNT_MAX = 16
RESIZE_WIDTH = 224
RESIZE_HEIGHT = 448

# Initialization
model = DA3_GIANT(L=40)
model.load_pretrained_weights()
freeze_vfm_parameters(model)
optimizer = Adam(model.trainable_parameters, lr=INITIAL_LR)

# Training Loop
for iteration in range(NUM_ITERATIONS):
    # 1. Data Sampling
    # Randomly select frame gap interval based on dataset min/max
    frame_gap = random_sample_gap(dataset_min_gap, dataset_max_gap)
    # Sample 3-16 frames from the interval
    frames = sample_frames(frame_gap, count=random_range(FRAME_COUNT_MIN, FRAME_COUNT_MAX))
    
    # 2. Preprocessing
    frames = resize(frames, RESIZE_WIDTH, RESIZE_HEIGHT)
    gt_depth, gt_rgb = get_ground_truth(frames)
    
    # 3. Forward Pass
    rendered_depth, rendered_rgb = model.render(frames)
    
    # 4. Loss Computation
    # L_depth components
    l1_depth_loss = l1_loss(rendered_depth, gt_depth)
    vfm_depth = model.vfm_predict_depth(frames) # VFM is frozen, no grad
    align_depth_loss = alignment_loss(rendered_depth, vfm_depth)
    
    # L_rgb components
    l1_rgb_loss = l1_loss(rendered_rgb, gt_rgb)
    ssim_rgb_loss = ssim_loss(rendered_rgb, gt_rgb)
    
    # Total Loss
    loss = (LAMBDA_L1_D * l1_depth_loss + 
            LAMBDA_ALIGN * align_depth_loss + 
            LAMBDA_L1_RGB * l1_rgb_loss + 
            LAMBDA_SSIM * ssim_rgb_loss)
    
    # 5. Optimization Step
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # 6. Learning Rate Scheduling
    # Cosine annealing with warm-up
    progress = iteration / NUM_ITERATIONS
    if iteration < WARMUP_ITERATIONS:
        lr = INITIAL_LR * (iteration / WARMUP_ITERATIONS)
    else:
        # Cosine annealing from INITIAL_LR to MIN_LR
        lr = MIN_LR + 0.5 * (INITIAL_LR - MIN_LR) * (1 + cos(pi * (progress - WARMUP_ITERATIONS/NUM_ITERATIONS) / (1 - WARMUP_ITERATIONS/NUM_ITERATIONS)))
    optimizer.set_lr(lr)
```

## Subtopics

### [Test-Time Optimization](17445-test-time-optimization.md)
A technique to exploit known camera parameters while keeping the pretrained visual foundation model (VFM) parameters frozen. It adapts lightweight LoRA modules using pose supervision.

## Operating characteristics
- **Time Complexity:** Training takes approximately 40 hours [S1, unofficial].
- **Hardware:** 2 RTX PRO 6000 GPUs [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Scalability:** The method relies on a fixed architecture size (1.39B total, 75.3M trainable). Performance scales with the number of GPUs and the resolution of input frames ($224 \times 448$) [S1, unofficial].
- **Strengths:** Efficient adaptation of large VFMs by freezing most parameters. Combines ground-truth supervision with VFM priors for robust depth alignment [S1, unofficial].
- **Limitations:** Requires known camera poses for the test-time optimization phase. Limited to sequential frame inputs [S1, unofficial].

## Background
VoxelTTO was introduced in the paper "VoxelTTO: Voxel-Aligned Feed-Forward 3D Gaussian Splatting with Test-Time Optimization" by Lin et al. (2025) [S1, unofficial]. It builds on the DA3-GIANT architecture and aims to improve 3D reconstruction accuracy by leveraging test-time optimization with frozen visual foundation models [S1, unofficial].

## Sources
[S1] VoxelTTO: Voxel-Aligned Feed-Forward 3D Gaussian Splatting with Test-Time Optimization, https://doi.org/10.48550/arxiv.2609.21498, fetched 2026-10-26 (unofficial)
