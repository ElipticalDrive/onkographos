# HashNeRF Parameters

Also called: HashNeRF Hyperparameters

## Overview
HashNeRF Parameters define the configuration for training Neural Radiance Fields using HashGrid encoding, balancing rendering quality and training speed on consumer hardware. It differs from standard NeRF by using a sparse, multi-resolution hash grid for feature storage [S1, unofficial], and from dense voxel methods by its spatial hashing mechanism [S1, unofficial]. The configuration is heavily influenced by Instant Neural Graphics Primitives (i-NGP) and ZipNeRF, optimizing for fast convergence with limited memory [S2, unofficial].

## How it works
The method configures the data structures, training loop, and sampling strategy for a HashGrid-based NeRF.

### Inputs and Outputs
**Inputs:**
- Dataset path and type (`dataset_type`: 'llff', 'blender', 'deepvoxels') [S1, unofficial].
- Image resolution settings (e.g., `half_res` for 400x400 vs 800x800) [S1, unofficial].
- Camera pose data (normalized device coordinates or world space) [S1, unofficial].
- Target resolution and scene bounds [S1, unofficial].

**Outputs:**
- Trained neural field model (HashGrid embeddings + MLP) [S1, unofficial].
- Rendered images and videos [S1, unofficial].
- Checkpoint weights [S1, unofficial].

### Data Structures
**HashGrid Pyramid:**
- Consists of $L$ levels (typically 10) [S2, unofficial].
- Resolutions range from $R_{min}=16$ to $R_{max}=8192$ with a growth factor of 2 [S2, unofficial].
- Each level has a hashmap of size $H = 2^{19}$ (default) or $2^{21}$ (ZipNeRF/RING-NeRF configuration) [S1, unofficial][S2, unofficial].
- Feature vector size per entry: 8 dimensions (default in RING-NeRF/ZipNeRF context) or 4 (original HashNeRF) [S1, unofficial][S2, unofficial].
- Hashmap size per level: $128^3 = 2^{21}$ entries in the referenced baseline configuration [S2, unofficial].

**Proposal Samplers:**
- Two rounds of sampling using grid-based proposal samplers [S2, unofficial].
- Samplers use max resolutions of 512 and 2048 [S2, unofficial].
- Feature size for samplers: 1 (density-only) [S2, unofficial].
- Architecture: Concatenation model based on i-NGP [S2, unofficial].

### Numbered Steps
1. **Data Loading:**
   - Load images and poses based on `dataset_type` [S1, unofficial].
   - Apply downsampling factor (`factor`) if specified (e.g., LLFF default 8) [S1, unofficial].
   - If `half_res` is true, resize Blender synthetic data to 400x400 [S1, unofficial].
   - If `white_bkgd` is true, composite alpha channel onto white background [S1, unofficial].
   - If `no_ndc` is true, do not convert to normalized device coordinates [S1, unofficial].
   - If `lindisp` is true, sample linearly in disparity rather than depth [S1, unofficial].
   - If `spherify` is true, recenter poses for spherical 360 scenes [S1, unofficial].
   - Split dataset into train/test using `llffhold` (every $N$-th image, default 8) [S1, unofficial].

2. **Initialization:**
   - Initialize HashGrid pyramid with $L=10$ levels, resolutions $16 \dots 8192$ [S2, unofficial].
   - Initialize hashmap size $H=2^{19}$ (default) or $2^{21}$ [S1, unofficial][S2, unofficial].
   - Set `raw_noise_std` to $1.0$ to regularize sigma output [S1, unofficial].

3. **Training Loop (25k iterations):** [S2, unofficial]
   - **Batching:** Sample $2^{16}$ rays per batch [S3].
   - **Proposal Sampling:**
     - Perform 2 rounds of sampling using proposal samplers with max resolutions 512 and 2048 [S2, unofficial].
     - Use features of size 1 for density estimation [S2, unofficial].
   - **Forward Pass:**
     - Encode sampled points using HashGrid encoding [S1, unofficial].
     - Pass through MLP to predict color and density [S1, unofficial].
     - Add noise to density with std `raw_noise_std` [S1, unofficial].
   - **Loss Calculation:**
     - Compute MSE loss between rendered and ground truth images [S1, unofficial].
     - Apply TV loss with weight `tv-loss-weight` ($10^{-6}$) [S1, unofficial].
     - Apply sparse loss with weight `sparse-loss-weight` ($10^{-10}$) [S1, unofficial].
     - Apply distortion loss (as implemented in NerfStudio) [S3].
   - **Optimization:**
     - Use RAdam optimizer with initial learning rate $10^{-2}$ [S3].
     - Apply cosine decay scheduler to final learning rate $10^{-3}$ [S3].
   - **Logging/Saving:**
     - Print metrics every `i_print` steps (default 100) [S1, unofficial].
     - Log images to TensorBoard every `i_img` steps (default 500) [S1, unofficial].
     - Save weights every `i_weights` steps (default 10000) [S1, unofficial].
     - Save testset renders every `i_testset` steps (default 1000) [S1, unofficial].
     - Save video renders every `i_video` steps (default 5000) [S1, unofficial].

4. **Pre-cropping (Optional):**
   - If `precrop_iters > 0`, train on central crops of fraction `precrop_frac` (default 0.5) for the first `precrop_iters` steps [S1, unofficial].

### Formulas
- **Hash Function:** $h(k) = k \mod H$, where $k$ is the hashed coordinate and $H$ is hashmap size [S1, unofficial].
- **Trilinear Interpolation:** Features are interpolated from the 8 neighboring grid cells in the active resolution level [S1, unofficial].
- **Volume Rendering:** $C(r) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) c_i$, where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [G].
- **Learning Rate Schedule:** $\eta(t) = \eta_{min} + \frac{1}{2}(\eta_{max} - \eta_{min})(1 + \cos(\frac{t}{T}\pi))$ [S3].
- **Scene Contraction (L∞ norm):** $x' = \frac{x}{\max(1, \|x\|_\infty)}$ to map unbounded space to a bounded unit cube [S4].

### Parameters
- `finest_res`: 512 (default) [S1, unofficial].
- `log2_hashmap_size`: 19 (default) [S1, unofficial].
- `sparse-loss-weight`: $10^{-10}$ (default) [S1, unofficial].
- `tv-loss-weight`: $10^{-6}$ (default) [S1, unofficial].
- `raw_noise_std`: 1.0 (recommended) [S1, unofficial].
- `render_factor`: 4 or 8 (for fast preview) [S1, unofficial].
- `precrop_iters`: 0 (default) [S1, unofficial].
- `precrop_frac`: 0.5 (default) [S1, unofficial].
- `dataset_type`: 'llff' (default) [S1, unofficial].
- `testskip`: 8 (default) [S1, unofficial].
- `shape`: 'greek' (default) [S1, unofficial].
- `white_bkgd`: Not given in the sources.
- `half_res`: Not given in the sources.
- `scannet_sceneID`: 'scene0000_00' (default) [S1, unofficial].
- `factor`: 8 (default) [S1, unofficial].
- `no_ndc`: Not given in the sources.
- `lindisp`: Not given in the sources.
- `spherify`: Not given in the sources.
- `llffhold`: 8 (default) [S1, unofficial].
- `i_print`: 100 (default) [S1, unofficial].
- `i_img`: 500 (default) [S1, unofficial].
- `i_weights`: 10000 (default) [S1, unofficial].
- `i_testset`: 1000 (default) [S1, unofficial].
- `i_video`: 5000 (default) [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision:** Hash collisions may occur if hashmap size is too small for the resolution [S1, unofficial].
- **Boundaries:** Scene contraction is used to handle unbounded scenes [S2, unofficial].
- **Degenerate Input:** Mismatch between image count and pose count causes failure [S1, unofficial].
- **Memory:** Large hashmap sizes ($2^{21}$) require significant GPU memory [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def load_dataset(dataset_type, factor, half_res):
    # Standard data loading logic based on dataset_type and resolution scaling
    pass

def split_dataset(dataset, llffhold):
    # Splits dataset into train/test by taking every llffhold-th image for test
    test_indices = list(range(0, len(dataset), llffhold))
    train_indices = [i for i in range(len(dataset)) if i not in test_indices]
    return train_indices, test_indices

class HashGridPyramid:
    def __init__(self, levels, resolutions, hashmap_size, feature_size):
        self.levels = levels
        self.resolutions = resolutions
        self.hashmap_size = hashmap_size
        self.feature_size = feature_size
        self.grids = [np.random.randn(hashmap_size, feature_size) for _ in levels]
        
    def encode(self, points):
        # Hash coordinates to grid indices and perform trilinear interpolation
        features = []
        for grid in self.grids:
            indices = np.floor(points * self.resolutions[0]).astype(int) % self.hashmap_size
            features.append(grid[indices])
        return np.mean(features, axis=0)

class MLP:
    def __init__(self, in_features=8, out_features=4):
        self.weights = np.random.randn(in_features, out_features)
        
    def forward(self, features):
        return features @ self.weights

def sample_rays(train_loader, batch_size):
    # Samples random rays from the training dataset
    return np.random.randn(batch_size, 3)

def proposal_sampler_sample(rays, hash_grid, max_resolutions, feature_size):
    # Performs 2 rounds of sampling using grid-based proposal samplers
    points = []
    for res in max_resolutions:
        points.append(rays * res)
    return np.concatenate(points, axis=0)

def noise(std):
    return np.random.normal(0, std)

def volume_render(rays, density, color):
    # Standard alpha compositing for volume rendering
    deltas = 0.1
    transmittance = np.exp(-np.cumsum(density * deltas, axis=0))
    weights = (1 - np.exp(-density * deltas)) * transmittance
    return np.sum(weights[..., None] * color, axis=0)

def mse(rendered, target):
    return np.mean((rendered - target) ** 2)

def total_variation(grid):
    # Computes total variation loss on grid features
    return np.mean(np.abs(grid[1:] - grid[:-1]))

def sparsity_loss(grid):
    # Computes sparsity loss to encourage empty space
    return np.mean(np.abs(grid))

def compute_distortion_loss(density, rays):
    # Distortion loss as implemented in NerfStudio
    return np.mean(density ** 2)

def train_hashnerf(config):
    # 1. Data Loading
    dataset = load_dataset(config.dataset_type, config.factor, config.half_res)
    train_indices, test_indices = split_dataset(dataset, config.llffhold)
    
    # 2. Initialization
    hash_grid = HashGridPyramid(
        levels=10,
        resolutions=generate_resolutions(16, 8192, factor=2),
        hashmap_size=2**config.log2_hashmap_size,
        feature_size=8
    )
    mlp = MLP()
    
    # 3. Training Loop
    for step in range(25000):
        rays = sample_rays(train_indices, batch_size=2**16)
        
        # Proposal Sampling
        proposal_points = proposal_sampler_sample(
            rays, 
            hash_grid, 
            max_resolutions=[512, 2048], 
            feature_size=1
        )
        
        # Forward Pass
        features = hash_grid.encode(proposal_points)
        color, density = mlp.forward(features)
        density += noise(std=config.raw_noise_std)
        
        # Rendering
        rendered_color = volume_render(rays, density, color)
        target_color = rays.target
        
        # Loss Calculation
        mse_loss = mse(rendered_color, target_color)
        tv_loss = total_variation(hash_grid.grids[0])
        sparse_loss = sparsity_loss(hash_grid.grids[0])
        distortion_loss = compute_distortion_loss(density, rays)
        
        total_loss = (mse_loss + 
                      config.tv_loss_weight * tv_loss + 
                      config.sparse_loss_weight * sparse_loss + 
                      distortion_loss)
        
        # Optimization (RAdam + Cosine Decay)
        lr = config.min_lr + 0.5 * (config.max_lr - config.min_lr) * (1 + math.cos(step / 25000 * math.pi))
        # Update weights using gradient descent with computed lr
        optimizer_step(hash_grid, mlp, lr)
        
        # Logging
        if step % config.i_print == 0:
            log_metrics(step, total_loss)
        if step % config.i_img == 0:
            log_image(step, rendered_color)
        if step % config.i_weights == 0:
            save_checkpoint(step, hash_grid, mlp)
        if step % config.i_testset == 0:
            save_testset(step, test_indices, hash_grid, mlp)
        if step % config.i_video == 0:
            save_video(step, render_poses, hash_grid, mlp)
```

## Subtopics

### [Hashed Embedding Table Initialization](3188-hashed-embedding-table-initialization.md)
Defines the configuration and initialization of the hashed embedding tables used in HashGrid-based neural fields, specifically within the HashNeRF framework. It is distinguished from standard positional encoding by its use of learnable hash tables with trilinear interpolation, and from dense voxel grids by its sparse, hash-based memory access pattern.

### L∞ norm scene contraction
A spatial compression technique that shrinks an unbounded scene into a fixed-size bounded space. It replaces the NDC method in the original NeRF pipeline and aligns the scene model more effectively with the hash code to improve reconstruction efficiency [S4].

## Operating characteristics
- **Iterations:** Training typically runs for 25k iterations [S2, unofficial][S3].
- **Batch Size:** $2^{16}$ rays per batch [S3].
- **Optimizer:** RAdam with initial learning rate $10^{-2}$, cosine-decayed to $10^{-3}$ [S3].
- **Grid Configuration:** 10 grids with a growth factor of 2, resolutions ranging from 16 to 8192 [S3].
- **Hashmap Size:** $2^{21}$ entries per level [S3].
- **Proposal Samplers:** 2 rounds of sampling via grid-based proposal samplers with max resolutions of 512 and 2048, using features of size 1 [S3].
- **Loss:** Distortion loss implemented in NerfStudio [S3].
- **Performance:** Comparable to original NeRF in 5 seconds on NVIDIA V100 [S1, unofficial].

## Background
HashNeRF was introduced by Müller et al. in "Instant Neural Graphics Primitives with a Hash Encoding" [S1, unofficial]. It was developed to enable real-time training and rendering of NeRFs on consumer GPUs. The method uses spatial hashing to map 3D coordinates to a sparse feature grid, allowing for efficient memory access and fast training [S1, unofficial].

## Sources
[S1] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
[S2] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S3] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
[S4] Neural Radiance Fields-Based 3D Reconstruction of Power Transmission Lines Using Progressive Motion Sequence Images., https://doi.org/10.3390/s23239537, fetched 2026-10-26
