# Coarse-to-Fine Optimization with Pruning

Also called: Plenoxel Grid with Trilinear Interpolation Optimization

## Overview
Coarse-to-Fine Optimization with Pruning is a hierarchical training strategy for optimizing 3D radiance fields represented as sparse voxel grids. It solves the problem of initializing and maintaining an efficient voxel structure that adapts to scene complexity during training, preventing overfitting in unobserved regions while refining detail where needed [S1]. It sets itself apart from static grid methods by dynamically pruning low-visibility voxels and subdividing high-error voxels based on gradient signals [S1], building on the general framework of [Plenoxels](7440-plenoxels.md).

## How it works
**Inputs and Outputs:**
- Input: A scene with training views, a target final resolution (e.g., $512^3$), and initial coarse resolution (e.g., $256^3$).
- Output: An optimized voxel grid containing density and spherical harmonic coefficients for the radiance field.

**Data Structures:**
- **Voxel Grid:** A 3D array storing per-voxel attributes. In the base Plenoxels implementation, this includes density ($\sigma$) and spherical harmonic (SH) coefficients for color.
- **Pruning Mask:** A boolean or weight-based metric indicating voxel occupancy.

**Parameters:**
- $T_{init}$: Initial resolution (e.g., $256^3$) [S1].
- $T_{final}$: Final resolution (e.g., $512^3$) [S1].
- $\tau_{prune}$: Threshold for pruning unoccupied voxels [S1].
- $\lambda_{TV}$: Weight for Total Variation regularization [S1].

**Steps:**

1. **Initialize:** Start with a dense voxel grid at the lower resolution $T_{init}$ (e.g., $256^3$). Initialize density and SH coefficients [S1].

2. **Optimize:** Train the grid parameters using the loss function defined in [S1]:
   $$ \mathcal{L} = \mathcal{L}_{recon} + \lambda_{TV} \mathcal{L}_{TV} $$
   where $\mathcal{L}_{recon}$ is the Mean Squared Error (MSE) over rendered pixel colors and $\mathcal{L}_{TV}$ is the Total Variation regularizer [S1].

3. **Prune:** Remove voxels deemed unoccupied. A voxel $i$ is pruned if its maximum weight over all training rays falls below a threshold. The weight is calculated as:
   $$ w_i = T_i(1-\exp(-\bar{\sigma}_i \bar{\delta}_i)) $$
   where $T_i$ is the transmittance, $\bar{\sigma}_i$ is the average density, and $\bar{\delta}_i$ is the average step size along rays passing through voxel $i$ [S1]. Alternatively, pruning can be based directly on the density value [S1].
   
   *Pitfall:* Naive pruning can adversely impact color and density near surfaces because trilinear interpolation uses values from immediate exterior voxels [S1].
   
   *Solution:* Perform a dilation operation. A voxel is only pruned if both itself and its neighbors are deemed unoccupied [S1].

4. **Refine:** Subdivide each remaining voxel in half in each dimension (e.g., from $256^3$ to $512^3$) [S1].

5. **Initialize New Values:** Use trilinear interpolation to initialize the grid values for the new, finer resolution based on the coarser grid [S1]. Trilinear interpolation at resolution $128^3$ has been observed to outperform nearest neighbor interpolation at $256^3$ [S1].

6. **Repeat:** Continue optimizing at the new resolution. Repeat steps 2-5 until the target resolution $T_{final}$ is reached [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_optimization(scene_data, T_init, T_final, tau_prune):
    # Step 1: Initialize
    grid = VoxelGrid(resolution=T_init)
    grid.initialize_parameters(scene_data)
    
    current_res = T_init
    
    while current_res < T_final:
        # Step 2: Optimize
        optimize_grid(grid, scene_data, lambda_tv=0.01) # lambda_tv is a typical value
        
        # Step 3: Prune
        # Calculate max weight for each voxel over all training rays
        voxel_weights = compute_max_voxel_weights(grid, scene_data.rays)
        
        # Apply dilation to prevent boundary artifacts
        is_unoccupied = voxel_weights < tau_prune
        dilated_unoccupied = apply_dilation(is_unoccupied, grid.shape)
        
        # Remove unoccupied voxels
        grid.prune(dilated_unoccupied)
        
        # Step 4: Refine
        new_res = current_res * 2
        grid.subdivide()
        
        # Step 5: Initialize New Values
        # Trilinear interpolation from old grid to new finer grid
        grid.interpolate_from_coarser()
        
        current_res = new_res
        
    return grid
```

## Subtopics

### [Progressive Sparse Voxels Optimization](2546-progressive-sparse-voxels-optimization.md)
Progressive Sparse Voxels Optimization is a coarse-to-fine training procedure for optimizing 3D radiance fields represented as sparse voxel grids. It solves the problem of initializing and maintaining an efficient voxel structure that adapts to scene complexity during training, preventing overfitting in unobserved regions while refining detail where needed. It sets itself apart from static grid methods by dynamically pruning low-visibility voxels and subdividing high-error voxels based on gradient signals, building on the general framework of [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md).

### Fine detail reconstruction
A second optimization stage that reconstructs surface details and view-dependent effects using a higher-resolution voxel grid and a hybrid explicit-implicit representation, freezing the coarse geometry grid [S2].

**Definition:**
This stage focuses on a smaller subspace to reconstruct surface details and view-dependent effects, freezing the optimized coarse geometry grid $V^{(density)(c)}$ [S2].

**Steps:**
1. **Freeze Coarse Grid:** The optimized $V^{(density)(c)}$ from the coarse stage is frozen [S2].
2. **Define Known Free Space:** A query point is in known free space if the post-activated alpha value from $V^{(density)(c)}$ is less than threshold $\tau^{(c)}$. Otherwise, it is in unknown space [S2].
3. **Fine Voxels Allocation:** Densely query $V^{(density)(c)}$ to find a BBox tightly enclosing the unknown space, with lengths $L_x^{(f)}, L_y^{(f)}, L_z^{(f)}$. Derive voxel size $s^{(f)}$ and grid dimensions $N_x^{(f)}, N_y^{(f)}, N_z^{(f)}$ from expected total voxels $M^{(f)}$ [S2].
4. **Fine-stage Points Sampling:** Filter out rays not intersecting known free space. Adjust near/far bounds $t^{(near)}$ and $t^{(far)}$ to ray-box intersection endpoints. Do not adjust $t^{(near)}$ if $x_0$ is already inside the BBox [S2].

**Parameters:**
- $M^{(f)}$: Hyperparameter for the expected total number of voxels in the fine stage [S2].
- $D$: Hyperparameter for feature-space dimension in the feature voxel grid $V^{(feat)(f)}$ [S2].

**Representation:**
The fine stage uses a higher-resolution density voxel grid $V^{(density)(f)} \in \mathbb{R}^{1 \times N_x^{(f)} \times N_y^{(f)} \times N_z^{(f)}}$ with post-activated interpolation [S2]. For view-dependent color, a hybrid representation is used:
- A feature voxel grid $V^{(feat)(f)} \in \mathbb{R}^{D \times N_x^{(f)} \times N_y^{(f)} \times N_z^{(f)}}$.
- A shallow MLP parameterized by $\Theta^{(f)}$.

Queries for 3D points $x$ and viewing direction $d$ are performed by:
$$ \vec{\phi}^{(f)} = \text{interp} \left( \boldsymbol{x}, \boldsymbol{V}^{(\text{density})(f)} \right) $$
$$ \mathbf{c}^{(\mathbf{f})} = \text{MLP}_{\Theta^{(\mathbf{f})}}(\text{interp}(\mathbf{x}, \mathbf{V}^{(\text{feat})(\mathbf{f})}), \mathbf{x}, \mathbf{d}) $$
where $\mathbf{c}^{(f)} \in \mathbb{R}^3$ is the view-dependent color emission and $\sigma^{(f)} \in \mathbb{R}$ is the raw volume density [S2]. Positional embedding is applied on $x, d$ for the MLP [S2].

**Progressive Scaling:**
Inspired by NSVF, the voxel grids $V^{(density)(f)}$ and $V^{(feat)(f)}$ are progressively scaled. Let $p_{g\_ckpt}$ be the set of checkpoint steps. The initial number of voxels is set to $\lfloor M^{(f)} / 2^{|p_{g\_ckpt}|} \rfloor$. When reaching a training step in $p_{g\_ckpt}$, the number of voxels is doubled such that the final number is $M^{(f)}$. At each checkpoint, the voxel grids are resized by trilinear interpolation [S2].

**Efficiency:**
Free space skipping is used. Points in known free space (checked via $V^{(density)(c)}$) are skipped during training and testing to improve efficiency, as querying $V^{(density)(c)}$ is faster than querying $V^{(density)(f)}$ or computing view-dependent colors [S2].

## Operating characteristics
- **Time Complexity:** The coarse-to-fine approach reduces training time compared to optimizing a high-resolution grid from scratch, as the initial coarse grid converges faster. However, the subdivision and interpolation steps add overhead. The fine detail reconstruction stage adds additional computational cost due to the hybrid explicit-implicit representation and MLP queries [S2].
- **Memory Complexity:** Pruning reduces memory usage by removing unoccupied voxels. The fine stage requires additional memory for the higher-resolution grid and feature grid, but the BBox allocation limits this to the unknown space [S2].
- **Performance:** Trilinear interpolation at lower resolutions (e.g., $128^3$) can outperform nearest neighbor at higher resolutions (e.g., $256^3$) [S1]. The method achieves fast convergence for radiance field reconstruction [S2].
- **Strengths:** Adapts to scene complexity, prevents overfitting in unobserved regions, and enables high-resolution detail reconstruction [S1, S2].
- **Limitations:** Naive pruning can cause artifacts near surfaces if not handled with dilation [S1]. The fine stage requires careful tuning of hyperparameters like $M^{(f)}$ and $\tau^{(c)}$ [S2].

## Background
Coarse-to-Fine Optimization with Pruning originates from the Plenoxels paper, which introduced a novel approach for fast radiance field reconstruction from sparse views using voxel grids without neural networks as the radiance field representation [S1]. The method builds on the concept of PlenOctrees for voxel pruning [S1]. The fine detail reconstruction variant was introduced in the Direct Voxel Grid Optimization paper to further enhance surface details and view-dependent effects [S2].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
