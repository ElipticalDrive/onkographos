# Sparse Voxel Grid Optimization


## Overview
Sparse Voxel Grid Optimization is a differentiable volume rendering method that represents a scene as an adaptive octree of sparse voxels, optimizing density and color attributes directly in grid memory rather than through a neural network [S1]. It solves the high memory overhead and slow convergence of MLP-based NeRFs by combining sparse voxel rasterization with adaptive subdivision [S1]. Unlike PlenOctree Optimization, which uses octrees primarily for compression and hierarchical storage of pre-trained features, this method actively grows and prunes the voxel structure during training to match scene complexity, enabling real-time rendering speeds comparable to 3D Gaussian Splatting while maintaining higher geometric fidelity [S1].

## How it works
The method optimizes a sparse voxel grid using differentiable volume rendering [S1]. It builds on the shared principles of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md), specifically the evaluation of the volume rendering equation and gradient propagation [G].

### Inputs and Outputs
**Inputs:**
- A set of $M$ training images with known camera poses $\mathbf{P}_m$ and intrinsics [G].
- Initial scene bounds (for bounded scenes) or a foreground bounding box plus background shell parameters (for unbounded scenes) [S1].

**Outputs:**
- An optimized sparse voxel octree structure containing density $\sigma$ and color $\mathbf{c}$ (or spherical harmonics coefficients) for each active voxel [S1].
- A rendered novel-view image $\hat{C}(\mathbf{r})$ for any ray $\mathbf{r}$ [G].

### Data Structures
The scene is represented as an octree [S1]. Each node in the octree corresponds to a voxel in 3D space [G].
- **Voxel Node:** Contains attributes for the voxel volume [G].
    - **Density ($\sigma$):** A scalar value representing the extinction coefficient [G].
    - **Color ($\mathbf{c}$):** A vector (RGB) or spherical harmonics coefficients representing the emitted color [G].
    - **Children:** Pointers to 8 child nodes if the voxel is subdivided [G].
- **Occupancy Cache:** A mechanism to track active voxels to avoid processing empty space, particularly important for unbounded scenes [S1].

### Algorithm Steps

1.  **Initialization:**
    -   Start optimization from empty space [S1]. Set the raw density offset $h_{geo} = -10$ [S1].
    -   Initialize the octree at level $h_{lv} = 6$ (resulting in a $64^3$ base grid) for the foreground main region [S1].
    -   For unbounded scenes, add $h_{out} = 5$ background shell levels, with a voxel count ratio $h_{ratio} = 2$ times the number of foreground voxels [S1].

2.  **Ray Tracing and Rasterization:**
    -   For each pixel, cast a ray $\mathbf{r}$ through the scene [G].
    -   Traverse the sparse octree to find intersecting voxels [G].
    -   Sort intersecting voxels by depth [G].
    -   Sample points within each voxel [G]. Use $K=1$ sample point for novel-view synthesis and $K=3$ for mesh reconstruction [S1].
    -   Compute transmittance $T_i$ and color contribution using the standard emission-absorption model [G]:
        $$
        \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
        $$
        where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [G].
    -   Apply early ray stopping if transmittance $T < h_T = 10^{-4}$ [S1].
    -   Apply supersampling with scale $h_{ss} = 1.5$ to reduce aliasing [S1].

3.  **Loss Computation:**
    Compute the total loss $\mathcal{L}$ as a weighted sum of terms [S1]:
    $$
    \mathcal{L} = \lambda_{\text{ssim}} \mathcal{L}_{\text{ssim}} + \lambda_{\text{T}} \mathcal{L}_{\text{T}} + \lambda_{\text{dist}} \mathcal{L}_{\text{dist}} + \lambda_{\text{R}} \mathcal{L}_{\text{R}} + \lambda_{\text{tv}} \mathcal{L}_{\text{tv}}
    $$
    -   $\mathcal{L}_{\text{ssim}}$: Structural Similarity Index Loss [S1].
    -   $\mathcal{L}_{\text{T}}$: Transmittance loss, encouraging final ray transmittances to be 0 or 1 [S1].
    -   $\mathcal{L}_{\text{dist}}$: Distortion loss [S1].
    -   $\mathcal{L}_{\text{R}}$: Per-point RGB loss [S1].
    -   $\mathcal{L}_{\text{tv}}$: Total Variation loss on the sparse density grid to smooth noise [S1].
    
    For mesh extraction tasks, add depth-normal consistency losses [S1]:
    $$
    \mathcal{L}_{\text{mesh}} = \lambda_{\text{n-dmean}} \mathcal{L}_{\text{n-dmean}} + \lambda_{\text{n-dmed}} \mathcal{L}_{\text{n-dmed}}
    $$
    These losses align rendered normals with depth-derived normals from mean and median depth [S1].

4.  **Optimization Step:**
    -   Perform backpropagation to compute gradients with respect to voxel densities and colors [G].
    -   Update parameters using an optimizer (e.g., Adam) [G].

5.  **Adaptive Subdivision and Pruning:**
    -   Every $h_{every} = 1,000$ iterations, prune and subdivide voxels [S1].
    -   **Pruning:** Remove voxels with density below a threshold $h_{prune}$ [S1]. The threshold is linearly scaled from 0.0001 to 0.05 over the training duration [S1].
    -   **Subdivision:** Subdivide voxels with high density or high gradient magnitude [G]. The subdivision percentage is set to $h_{percent} = 5$ [S1].
    -   Repeat until the target number of iterations is reached (typically 20,000) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Tensor:
    def __init__(self, data):
        self.data = data
    def backward(self):
        pass  # Placeholder for autograd engine
    def zero_grad(self):
        pass

class Optimizer:
    def step(self): pass
    def zero_grad(self): pass

def sample_rays(images, poses):
    return [(ray_origin, ray_direction) for _ in range(100)]  # Batch of rays

def get_ground_truth_colors(rays):
    return [[0.5, 0.5, 0.5] for _ in rays]

def compute_ssim_loss(pred, gt): return 0.0
def compute_transmittance_loss(pred, rays): return 0.0
def compute_distortion_loss(pred, rays): return 0.0
def compute_rgb_loss(pred, gt): return 0.0
def compute_total_variation_loss(grid): return 0.0

def sparse_voxel_grid_optimization(training_images, camera_poses, num_iterations=20000):
    # 1. Initialization
    octree = Octree(initial_level=6)  # 64^3 base grid
    raw_density_offset = -10
    octree.set_raw_density_offset(raw_density_offset)
    octree.add_background_shells(num_levels=5, ratio=2.0)
    
    h_every = 1000
    h_T = 1e-4
    h_ss = 1.5
    current_prune_threshold = 0.0001
    max_prune_threshold = 0.05
    optimizer = Optimizer()
    
    # 2. Training Loop
    for iteration in range(1, num_iterations + 1):
        rays = sample_rays(training_images, camera_poses)
        rendered_colors = []
        
        for ray in rays:
            intersecting_voxels = octree.traverse_and_sort(ray)
            samples = sample_points(intersecting_voxels, K=1)
            
            T = 1.0
            color = [0.0, 0.0, 0.0]
            for sample in samples:
                if T < h_T:
                    break
                sigma = octree.get_density(sample.voxel)
                c = octree.get_color(sample.voxel)
                delta = sample.distance
                
                alpha = 1.0 - math.exp(-sigma * delta)
                weight = T * alpha
                color[0] += weight * c[0]
                color[1] += weight * c[1]
                color[2] += weight * c[2]
                T *= math.exp(-sigma * delta)
            rendered_colors.append(color)
        
        ground_truth = get_ground_truth_colors(rays)
        
        loss_ssim = compute_ssim_loss(rendered_colors, ground_truth)
        loss_T = compute_transmittance_loss(rendered_colors, rays)
        loss_dist = compute_distortion_loss(rendered_colors, rays)
        loss_R = compute_rgb_loss(rendered_colors, ground_truth)
        loss_tv = compute_total_variation_loss(octree.density_grid)
        
        lambda_ssim, lambda_T, lambda_dist, lambda_R, lambda_tv = 0.2, 1.0, 0.1, 1.0, 0.01
        total_loss = (lambda_ssim * loss_ssim + lambda_T * loss_T + 
                      lambda_dist * loss_dist + lambda_R * loss_R + 
                      lambda_tv * loss_tv)
        
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        if iteration % h_every == 0:
            progress = iteration / num_iterations
            current_prune_threshold = 0.0001 + (max_prune_threshold - 0.0001) * progress
            octree.prune(threshold=current_prune_threshold)
            octree.subdivide(percentage=0.05)
            
    return octree
```

### Edge Cases and Pitfalls
-   **Exposure Variation:** Scenes with severe exposure variation in training views can cause the method to produce clear brightness boundaries and "floaters" (spurious geometry), degrading PSNR and FPS [S1].

## Subtopics

### [Adaptive Voxel Size](2631-adaptive-voxel-size.md)
Adaptive Voxel Size is a hierarchical grid optimization technique where voxels are subdivided based on local detail requirements rather than using a uniform grid resolution. It solves the memory and performance bottlenecks of uniform high-resolution grids by allocating fine voxels only where necessary, achieving higher visual quality (LPIPS 0.200, PSNR 28.01) than uniform $512^3$ grids while maintaining competitive rendering speeds (171 FPS). Unlike uniform variants that subdivide all voxels to a fixed depth, this method dynamically adjusts granularity, outperforming fully-explicit voxel approaches like Plenoxels in both quality and speed. Choose it when balancing high-fidelity novel-view synthesis with real-time rendering constraints.

### [SVRecon](4359-svrecon.md)
SVRecon is a surface reconstruction method that generalizes SVRaster by parametrizing each voxel with Signed Distance Function (SDF) values at its corners, enabling implicit geometry regularization and the extraction of watertight meshes. It addresses the limitation of SVRaster’s density-based representation, which lacks explicit geometric priors and suffers from depth bias, by introducing SDF-based continuity enforcement. Unlike 3D Gaussian Splatting (3DGS) approaches that rely heavily on initial Structure-from-Motion (SfM) points, SVRecon leverages PI³ point-map priors aligned to ground-truth poses for initialization, producing sharper boundaries and smoother, hole-free surfaces. Choose it when explicit, watertight mesh extraction is required without SfM dependency.

### DirectVoxGO
DirectVoxGO is a differentiable volume rendering method that optimizes a single 3D voxel grid to represent a Neural Radiance Field (NeRF), storing density and color features directly in grid tensors [S1]. It solves the slow convergence and high memory overhead of MLP-based NeRFs by enabling direct optimization of grid values [S1]. It distinguishes itself from standard NeRF by replacing coordinate-to-feature MLP lookups with direct grid sampling, and from sparse voxel methods by using a dense grid with occupancy caching and near-camera masking to handle unbounded scenes efficiently [S1]. Choose it when fast training convergence is prioritized over memory efficiency in bounded or moderately unbounded scenes.

### [Fused Sparse Voxel Volume Rendering Backward Pass](15369-fused-sparse-voxel-volume-rendering-backward-pass.md)
The fused sparse voxel volume rendering backward pass computes gradients for spherical harmonics (SH) coefficients, SH basis values, density, and render accumulators during differentiable volume marching through a packed sparse grid. It natively supports gradient accumulation for beta loss and sparsity loss regularization terms. The method fuses ray marching, color interpolation, and gradient computation into a single kernel to minimize memory bandwidth overhead. Choose it when maximizing training throughput and minimizing memory bandwidth bottlenecks on GPU hardware.

### 4D Voxel Optimization Stage
The 4D Voxel Optimization Stage finetunes a dynamics module $f_\psi$ and 4D voxel grids $V_{t=1}$ obtained from previous stages [S2, unofficial]. It trains a volume slot attention mechanism and compositional renderer collaboratively to refine object-centric voxel grids [S2, unofficial]. The dynamic objective is defined as $\mathcal{L}_{\text{Dyn}} = \sum_{t=1}^T (\mathcal{L}_{\text{Render}} + \alpha_p \mathcal{L}_{\text{Point}} + \alpha_e \mathcal{L}_{\text{Ent}})$ [S2, unofficial]. Choose it when optimizing dynamic scenes with object-centric motion and appearance features.

## Operating characteristics
-   **Time Complexity:** Training time is significantly reduced compared to MLP-based NeRFs [S1]. Typical training takes ~15 minutes for 20,000 iterations on standard datasets [S1]. Fast-train variants can achieve >3x training speedup by scaling iteration-related hyperparameters by 0.3 [S1].
-   **Memory Complexity:** Memory usage scales with the number of active voxels, not the full grid resolution [S1]. This allows handling of high-resolution details that would cause OOM in uniform grids [S1]. However, it uses more voxel primitives than Gaussian splatting, resulting in larger model size and higher GPU memory usage than 3DGS in some scenes [S1].
-   **Rendering Performance:** Achieves real-time rendering speeds (FPS 121-125 on Mip-NeRF360, up to 366 on Deep Blending) comparable to 3D Gaussian Splatting [S1]. Fast-render variants with $h_{ss}=1.5$ and $h_{prune}=0.15$ achieve >2x FPS with moderate quality reduction [S1].
-   **Strengths:** High visual quality (low LPIPS), ability to extract watertight meshes via TSDF fusion and Marching Cubes, no dependency on SfM points for geometry, and efficient handling of unbounded scenes [S1].
-   **Limitations:** Struggles with scenes having severe exposure variation, leading to floaters and boundary artifacts [S1]. Performance can be scene-dependent due to the variable number of contributing primitives [S1].
-   **When to use:** When real-time rendering and high geometric fidelity (mesh extraction) are required, and the scene has moderate to high complexity where uniform grids are too large [S1].
-   **When not to use:** For scenes with extreme photometric variations where floaters become problematic, or when minimal GPU memory is available and the scene is very simple (where dense grids or Gaussians might be more efficient) [S1].

## Background
Sparse Voxel Grid Optimization, specifically in the form of SVRaster, was introduced in the paper "Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering" [S1]. It emerged as a response to the limitations of both neural field methods (slow training, high memory) and Gaussian splatting (lack of explicit geometry, dependency on SfM) [S1]. The method leverages the efficiency of sparse voxel structures for rasterization and the flexibility of adaptive octrees for representation [S1]. It is used in novel-view synthesis and 3D reconstruction tasks, particularly where mesh extraction is desired [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
[S2] DynaVol: Unsupervised Learning for Dynamic Scenes through Object-Centric Voxelization, https://doi.org/10.48550/arxiv.2305.00393, fetched 2026-10-26 (unofficial)
