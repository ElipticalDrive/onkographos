# Coarse-to-Fine Optimization with Pruning

Also called: Plenoxel Grid with Trilinear Interpolation Optimization

## Overview
Coarse-to-Fine Optimization with Pruning is a hierarchical training strategy for 3D radiance fields represented as voxel grids [S1]. It solves the problem of efficiently initializing and refining high-resolution scene representations by starting with a coarse dense grid, pruning unoccupied regions, and iteratively subdividing the remaining voxels to higher resolutions [S1]. It sets itself apart from static grid methods by dynamically adapting the voxel structure to scene complexity during training, building on the general framework of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md) [S1].

## How it works
**Inputs and Outputs:**
- Input: A set of training images with known camera poses [S1].
- Output: An optimized voxel grid representation (density and color features) at the target resolution [S1].

**Data Structures:**
- **Voxel Grid:** A 3D array storing density ($\sigma$) and color features (e.g., spherical harmonic coefficients) [S1].
- **Pruning Mask:** A boolean indicator for each voxel to determine if it is occupied, derived from weight or density thresholds [S1].

**Steps:**
1. **Initialize:** Start with a dense voxel grid at a lower resolution (e.g., $256^3$) [S1].
2. **Optimize:** Train the grid parameters (opacities, spherical harmonic coefficients) using a loss function, typically Mean Squared Error (MSE) on rendered pixel colors plus Total Variation (TV) regularization [S1].
   $$ \mathcal{L} = \mathcal{L}_{recon} + \lambda_{TV} \mathcal{L}_{TV} $$
   where $\mathcal{L}_{recon}$ is the reconstruction loss and $\mathcal{L}_{TV}$ is the TV regularizer [S1].
3. **Prune:** Remove voxels deemed unoccupied. This is done by calculating a metric for each voxel over all training rays. The metric is the maximum weight:
   $$ w_i = T_i (1 - \exp(-\bar{\sigma}_i \bar{\delta}_i)) $$
   where $T_i$ is the transmittance, $\bar{\sigma}_i$ is the average density, and $\bar{\delta}_i$ is the average step size for voxel $i$ [S1]. Alternatively, a threshold on the density value itself can be used [S1].
   - **Pitfall & Solution:** Naive pruning can adversely impact color and density near surfaces because trilinear interpolation uses values from immediate exterior voxels. To solve this, perform a dilation operation: a voxel is only pruned if both itself and its neighbors are deemed unoccupied [S1].
4. **Refine:** Subdivide each remaining voxel in half in each dimension (e.g., from $256^3$ to $512^3$) [S1].
5. **Initialize New Values:** Use trilinear interpolation to initialize the grid values for the new, finer resolution based on the coarser grid [S1]. Trilinear interpolation at a lower resolution (e.g., $128^3$) can outperform nearest neighbor interpolation at a higher resolution (e.g., $256^3$) [S1].
6. **Repeat:** Continue optimizing at the new resolution. This process can be repeated to reach arbitrary target resolutions [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_optimization(training_rays, initial_res=256, target_res=512, prune_threshold=0.01):
    grid = create_voxel_grid(resolution=initial_res)
    current_res = initial_res

    while current_res < target_res:
        # Optimize grid parameters (opacities, spherical harmonic coefficients) using MSE + TV regularization [S1]
        # Note: Optimization loop details are not specified in sources [G]
        for step in range(num_optimization_steps):
            loss = compute_loss(grid, training_rays)
            update_parameters(grid, loss)

        # Prune unoccupied voxels using max weight threshold or density threshold [S1]
        # Calculate max weight for each voxel over all training rays: w_i = T_i * (1 - exp(-sigma_i * delta_i))
        voxel_weights = compute_max_weight_over_rays(grid, training_rays)
        voxels_to_prune = [w < prune_threshold for w in voxel_weights]

        # Dilation: voxel only pruned if itself and neighbors are unoccupied [S1]
        # Note: Dilation kernel size is not specified in sources [G]
        voxels_to_prune = apply_dilation(voxels_to_prune)

        grid.prune(voxels_to_prune)

        # Refine: Subdivide each remaining voxel in half in each dimension [S1]
        new_res = min(current_res * 2, target_res)
        grid.subdivide(new_res)

        # Initialize new values using trilinear interpolation from coarser grid [S1]
        grid.interpolate_from_coarser()

        current_res = new_res

    return grid
```

**Parameters:**
- `initial_res`: Starting resolution, typically $256^3$ [S1].
- `target_res`: Final resolution, e.g., $512^3$ [S1].
- `prune_threshold`: Threshold for voxel pruning, based on $T_i(1-\exp(-\bar{\sigma}_i \bar{\delta}_i))$ or density [S1].

**Edge Cases and Pitfalls:**
- **Surface Artifacts:** Naive pruning without dilation causes artifacts near surfaces due to trilinear interpolation pulling values from empty exterior voxels [S1].
- **Interpolation Quality:** Trilinear interpolation at lower resolutions can outperform nearest neighbor at higher resolutions, improving visual quality [S1].

## Subtopics

### [Progressive Sparse Voxels Optimization](2546-progressive-sparse-voxels-optimization.md)
Progressive Sparse Voxels Optimization is a coarse-to-fine training procedure for optimizing 3D radiance fields represented as sparse voxel grids. It solves the problem of initializing and maintaining an efficient voxel structure that adapts to scene complexity during training, preventing overfitting in unobserved regions while refining detail where needed. It sets itself apart from static grid methods by dynamically pruning low-visibility voxels and subdividing high-error voxels based on gradient signals, building on the general framework of [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md).

### Fine detail reconstruction
**Definition:** A second optimization stage that reconstructs surface details and view-dependent effects using a higher-resolution voxel grid and a hybrid explicit-implicit representation, freezing the coarse geometry grid [S2].

**Steps:**
1. **Freeze Coarse Grid:** The optimized coarse density grid $V^{(\text{density})(c)}$ is frozen [S2].
2. **Define Known Free Space:** A query point is in known free space if the post-activated alpha value from $V^{(\text{density})(c)}$ is less than a threshold $\tau^{(c)}$. Otherwise, it is in unknown space [S2].
3. **Fine Voxels Allocation:** Densely query $V^{(\text{density})(c)}$ to find a bounding box (BBox) tightly enclosing the unknown space. Derive voxel size $s^{(f)}$ and grid dimensions $N^{(f)}_x, N^{(f)}_y, N^{(f)}_z$ from the expected total number of voxels $M^{(f)}$ [S2].
4. **Fine-stage Points Sampling:** Filter out rays not intersecting known free space. Adjust near/far bounds $t^{(\text{near})}$ and $t^{(\text{far})}$ to ray-box intersection endpoints. Do not adjust $t^{(\text{near})}$ if the ray origin $x_0$ is already inside the BBox [S2].
5. **Progressive Scaling:** Start with half the expected voxels $\lfloor M^{(f)}/2 \rfloor$. At checkpoint steps, double the number of voxels until $M^{(f)}$ is reached. Resize voxel grids $V^{(\text{density})(f)}$ and $V^{(\text{feat})(f)}$ by trilinear interpolation at each checkpoint [S2].
6. **Free Space Skipping:** Skip sampled points in known free space by checking $V^{(\text{density})(c)}$ to improve efficiency [S2].

**Data Structures:**
- **Fine Density Grid:** $V^{(\text{density})(f)} \in \mathbb{R}^{1 \times N^{(f)}_x \times N^{(f)}_y \times N^{(f)}_z}$ with post-activated interpolation [S2].
- **Feature Grid:** $V^{(\text{feat})(f)} \in \mathbb{R}^{D \times N^{(f)}_x \times N^{(f)}_y \times N^{(f)}_z}$, where $D$ is the feature-space dimension [S2].
- **MLP:** A shallow MLP parameterized by $\Theta^{(f)}$ for view-dependent color [S2].

**Formulas:**
- Query density: $\vec{\phi}^{(f)} = \text{interp}(\mathbf{x}, V^{(\text{density})(f)})$ [S2].
- Query color: $\mathbf{c}^{(\mathbf{f})} = \text{MLP}_{\Theta^{(\mathbf{f})}}(\text{interp}(\mathbf{x}, V^{(\text{feat})(\mathbf{f})}), \mathbf{x}, \mathbf{d})$ [S2].

**Parameters:**
- $M^{(f)}$: Hyperparameter for the expected total number of voxels in the fine stage [S2].
- $D$: Hyperparameter for feature-space dimension in $V^{(\text{feat})(f)}$ [S2].
- $\tau^{(c)}$: Threshold for known free space [S2].

## Operating characteristics
- **Time Complexity:** The coarse-to-fine approach reduces training time compared to optimizing a high-resolution grid from scratch, as the initial coarse grid converges faster [S1]. However, the iterative refinement and pruning steps add overhead [S1]. The fine detail reconstruction stage adds additional computational cost due to the hybrid explicit-implicit representation and progressive scaling [S2].
- **Memory Complexity:** Pruning reduces memory usage by removing unoccupied voxels [S1]. The fine stage uses a higher-resolution grid but only in the unknown space, balancing memory and detail [S2].
- **Performance:** Trilinear interpolation at lower resolutions can outperform nearest neighbor at higher resolutions, improving visual quality [S1]. The method achieves super-fast convergence for radiance field reconstruction [S2].
- **Strengths:** Efficiently handles high-resolution scenes by focusing computation on relevant areas [S1]. Prevents overfitting in unobserved regions [S1].
- **Limitations:** Requires careful tuning of pruning thresholds and dilation to avoid artifacts [S1].
- **When to Use:** Suitable for high-quality novel view synthesis where computational resources are limited but high resolution is required [G].
- **When Not to Use:** Not ideal for real-time applications requiring immediate high-resolution output without iterative refinement [G].

## Background
Coarse-to-Fine Optimization with Pruning originates from the Plenoxels paper, which introduced radiance fields without neural networks using voxel grids [S1]. It builds on earlier work in hierarchical grid optimization and sparse voxel representations. The method has been further refined in Direct Voxel Grid Optimization, which adds fine detail reconstruction for surface details and view-dependent effects [S2].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
