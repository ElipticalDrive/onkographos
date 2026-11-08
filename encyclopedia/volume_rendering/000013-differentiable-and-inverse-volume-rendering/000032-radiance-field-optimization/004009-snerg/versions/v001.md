# SNeRG
Also called: Deferred Shading NeRF, SNeRG (Deferred Shading NeRF), Sparse Neural Radiance Fields with View-dependent Effects

## Overview
SNeRG reduces the computational cost of Neural Radiance Fields (NeRF) by decomposing the radiance field into a 3D field of densities, diffuse RGB colors, and feature vectors, using a deferred shading model to limit MLP evaluations to one per ray [S1, unofficial]. Unlike standard NeRF which queries a single MLP for both density and view-dependent color at every sample point, SNeRG separates these concerns, caching diffuse color and using a small network only for view-dependent effects [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Camera poses, intrinsic parameters, training images, and initial scene geometry (often from a coarse reconstruction or implicit surface).
- **Outputs:** A cached sparse 3D grid for density/geometry, a 3D texture atlas for diffuse colors, and a small neural network for view-dependent effects.

**Data Structures:**
1.  **Sparse 3D Grid:** A low-resolution 3D grid storing density values $\sigma$. It is used to quickly locate the surface along a ray.
2.  **3D Texture Atlas:** Stores the baked diffuse RGB colors $\mathbf{c}_d$ for occupied voxels.
3.  **Small Network:** A lightweight MLP that takes the diffuse color and view direction as input to predict the final view-dependent color.

**Steps:**
1.  **Ray Marching for Geometry:** For a given ray, query the sparse 3D grid to find the surface intersection or density distribution. This step avoids MLP evaluation for geometry.
2.  **Color Baking:** During training, the diffuse component of the color is baked into the 3D texture atlas. This involves evaluating the radiance field to separate diffuse and specular components, then storing the diffuse part in the grid.
3.  **Deferred Shading for View-Dependent Effects:** At render time, for each sample point along the ray:
    - Retrieve the baked diffuse color $\mathbf{c}_d$ from the 3D texture atlas.
    - Retrieve the view direction $\mathbf{v}$.
    - Pass $\mathbf{c}_d$ and $\mathbf{v}$ through the small network $f_{\theta}$ to get the view-dependent color $\mathbf{c}_{vd}$:
      $$ \mathbf{c}_{vd} = f_{\theta}(\mathbf{c}_d, \mathbf{v}) $$
    - The final color contribution is based on $\mathbf{c}_{vd}$.
4.  **Volume Rendering:** Compute the final pixel color using the standard volume rendering equation, summing the contributions from all samples along the ray, weighted by transmittance and density.

**Formulas:**
The final color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is:
$$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_{vd,i} $$
where $\mathbf{c}_{vd,i} = f_{\theta}(\mathbf{c}_{d,i}, \mathbf{v})$, $\mathbf{c}_{d,i}$ is the baked diffuse color, and $\mathbf{v}$ is the view direction [S1, unofficial].

**Parameters:**
- Grid resolution: Not given in the sources.
- Small network size: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Sparsity:** The method relies on a sparse grid. Empty space must be handled efficiently to save VRAM. A sparsity loss is used during training to reduce the VRAM usage of sparse voxels [S2, unofficial].
- **View-Dependent Effects:** The small network must be capable of capturing significant view-dependent effects (e.g., specularities) that are not captured by the baked diffuse color.

## Operating characteristics
- **Time Complexity:** Reduces the number of MLP evaluations from $O(N)$ per ray (where $N$ is the number of samples) to $O(1)$ per ray for the main radiance field, with only a small network evaluation for view-dependent effects. This significantly speeds up rendering.
- **Memory Complexity:** Uses a sparse 3D grid and a 3D texture atlas. The sparsity loss helps minimize memory usage for empty regions [S2, unofficial].
- **Strengths:** Real-time view synthesis capabilities due to reduced computational cost. Efficient memory usage for large scenes.
- **Limitations:** May struggle with complex view-dependent effects if the small network is too simple. Requires a baking step during training.

## Background
SNeRG was introduced by Hedman et al. in "Baking Neural Radiance Fields for Real-Time View Synthesis" (ICCV 2021) [S1, unofficial]. It builds on the NeRF framework by addressing its high computational cost through deferred shading and caching techniques.

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
