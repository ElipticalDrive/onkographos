# Voxel and Grid-Based Prediction

Also called: Direct Multi-Plane Image Gaussian Optimization, DirectMPIGO

## Overview
Voxel and Grid-Based Prediction is a neural rendering method that combines voxel-based density representation with Multi-Plane Image (MPI) depth layers to perform efficient volume rendering [S1, unofficial]. It solves the problem of slow convergence and poor surface quality in implicit neural representations by using explicit grid-based features alongside approximate Signed Distance Function (SDF) supervision [S1, unofficial]. Unlike its sibling [Voxel and Grid-Based Prediction](15382-voxel-and-grid-based-prediction.md), which typically predicts Gaussian parameters or raw densities, DirectMPIGO explicitly predicts an SDF value via a feature grid to enable robust surface extraction and regularization [S1, unofficial]. It is significantly faster than implicit-only methods like NeuralRGB-D but requires larger memory footprints due to the explicit grid storage [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Camera rays (origin $\mathbf{o}$, direction $\mathbf{d}$), scene bounding box ($\mathbf{xyz\_min}, \mathbf{xyz\_max}$), MPI depth planes ($mpi\_depth$), and optional pre-computed mask cache.
- **Outputs:** Rendered RGB color and alpha (opacity) for each ray.

**Data Structures:**
1.  **Density Grid:** A `DenseGrid` (or `SparseGrid`) storing density values. Shape depends on resolution and `mpi_depth`.
2.  **Feature Grid ($k_0$):** Stores color/feature vectors. Channels = $rgbnet\_dim$ (or 3 if $rgbnet\_dim \le 0$).
3.  **Act Shift Grid:** A `DenseGrid` tensor of shape $[1, 1, mpi\_depth]$, `requires_grad=False`. Stores logit shifts to equalize initial alpha contributions across MPI depth layers.
4.  **Mask Cache:** A boolean tensor of shape `mask_cache_world_size`. Tracks occupied voxels to skip empty space during rendering.
5.  **Color MLP:** A multi-layer perceptron mapping features to RGB.

**Parameters:**
-   $\mathbf{xyz\_min}, \mathbf{xyz\_max}$: Bounding box of the scene.
-   $mpi\_depth$: Number of depth planes in the MPI representation.
-   $fast\_color\_thres$: Threshold for occupancy mask updates; voxels with alpha below this are considered empty.
-   $rgbnet\_dim$: Dimension of the color feature vector. If $\le 0$, uses raw RGB (dim=3) from $k_0$ grid; otherwise uses MLP output dimension.
-   $rgbnet\_depth$: Depth of the color MLP.
-   $rgbnet\_width$: Width of the color MLP.
-   $viewbase\_pe$: Number of positional encoding frequencies for view direction.
-   $voxel\_size\_ratio$: Constant ratio $256.0 / mpi\_depth$.

**Initialization Steps:**
1.  **Grid Creation:** Initialize density grid using `grid.create_grid` with `density_type`, `channels=1`, and world size derived from `_set_grid_resolution`. Initialize $k_0$ grid with `k0_type` and `channels` determined by $rgbnet\_dim$.
2.  **World Size Calculation:**
    $$
    \begin{aligned}
    r &= \sqrt{\frac{num\_voxels}{mpi\_depth \cdot \prod (\mathbf{xyz\_max} - \mathbf{xyz\_min})[:2]}} \\
    world\_size[:2] &= (\mathbf{xyz\_max} - \mathbf{xyz\_min})[:2] \cdot r \\
    world\_size[2] &= mpi\_depth
    \end{aligned}
    $$
3.  **Act Shift Initialization:**
    -   Initialize density values $g[i] = \frac{1.0}{mpi\_depth} - 10^{-6}$ for all $i \in [0, mpi\_depth)$. This ensures uniform initial transmittance.
    -   Compute conditional probability:
        $$
        p[i] = \begin{cases} 
        1 - g[0] & i = 0 \\
        \frac{1 - \sum_{j=0}^{i+1} g[j]}{1 - \sum_{j=0}^{i} g[j]} & i > 0
        \end{cases}
        $$
    -   Compute act shift value at layer $i$:
        $$
        act\_shift[i] = \log \left( \left( p[i] \right)^{-1 / voxel\_size\_ratio} - 1 \right)
        $$
4.  **Color MLP Initialization:**
    -   If $rgbnet\_dim \le 0$, set $k_0$ channels to 3 and disable the MLP.
    -   If $rgbnet\_dim > 0$, create $k_0$ with $rgbnet\_dim$ channels. Initialize MLP:
        -   Input dim: $dim_0 = (3 + 3 \cdot viewbase\_pe \cdot 2) + k_0\_dim$.
        -   Layers: Linear($dim_0$, $rgbnet\_width$), ReLU, $(rgbnet\_depth-2)$ layers of Linear($rgbnet\_width$, $rgbnet\_width$) + ReLU, final Linear($rgbnet\_width$, 3).
        -   Initialize final bias to 0.
    -   Initialize view direction positional encoding buffer with powers of 2: $[2^0, 2^1, \dots, 2^{(viewbase\_pe-1)}]$.
5.  **Mask Cache Initialization:** If `mask_cache_path` is provided, load it; otherwise, create an all-True boolean tensor of size `mask_cache_world_size`.

**Rendering Steps:**
1.  **Ray-Bbox Intersection:** For each ray, compute intersection $[t_{min}, t_{max}]$ with the bounding box defined by $\mathbf{xyz\_min}$ and $\mathbf{xyz\_max}$ [S1, unofficial].
    $$
    \begin{aligned}
    t_{min} &= \max(\min(\max(\min(ax, bx), \min(ay, by)), \min(az, bz)), near), far) \\
    t_{max} &= \max(\min(\min(\max(ax, bx), \max(ay, by)), \max(az, bz)), near), far)
    \end{aligned}
    $$
    Where $ax = (\mathbf{xyz\_max}_x - o_x) / d_x$, etc., with zero division protection.
2.  **Sample Points:** Calculate number of samples $N = \lceil (t_{max} - t_{min}) \cdot \|d\| / stepdist \rceil$. Sample points along the ray within $[t_{min}, t_{max}]$.
3.  **Grid Query:** Query density and features from the grids at sampled points.
4.  **Volume Rendering:** Composite color and alpha using the MPI depth layers and the act shift values to balance contributions.

**Occupancy Cache Updates:**
1.  **Update Occupancy Cache:** Query density at cache grid points. Activate voxels where alpha $> fast\_color\_thres$. Apply `max_pool3d(kernel=3, padding=1)` and update mask.
2.  **Update Occupancy Cache (Low Views):** For each ray batch, accumulate gradient counts. Update mask where count $\ge maskout\_lt\_nviews$.
3.  **Scale Volume Grid:** If new world size product $\le 256^3$, recompute mask cache by combining old mask with alpha $> fast\_color\_thres$ after `max_pool3d` smoothing.

**Total Variation Gradient Scaling:**
$$
\begin{aligned}
w_{xy} &= weight \cdot \frac{\max(world\_size[:2])}{128} \\
w_z &= weight \cdot \frac{mpi\_depth}{128}
\end{aligned}
$$

## Subtopics

### [GO-Surf Neural Feature Grid Optimization](15383-go-surf-neural-feature-grid-optimization.md)
GO-Surf is a differentiable volume rendering method that reconstructs high-fidelity 3D surfaces from RGB-D inputs by optimizing a multi-resolution neural feature grid and a lightweight geometry decoder. It solves the problem of slow convergence and poor surface quality in implicit neural representations by combining explicit grid-based features with approximate Signed Distance Function (SDF) supervision. Unlike its sibling [Voxel and Grid-Based Prediction](15382-voxel-and-grid-based-prediction.md), which typically predicts Gaussian parameters or raw densities, GO-Surf explicitly predicts an SDF value via a feature grid to enable robust surface extraction and regularization. It is significantly faster than implicit-only methods like NeuralRGB-D but requires larger memory footprints due to the explicit grid storage.

### Gaussian Splatting Attribute Prediction
A voxel transformer that predicts residuals added to initial Gaussian rendering attributes on voxels, rather than generating new attributes directly [S2]. The transformer takes fused voxel attributes $\mathbf{E}_j$ and offset $\mathbf{O}_j$ as input. The residual $\Delta \mathbf{G}_j$ is computed as:
$$
\Delta \mathbf{G}_j = \text{MLP}_{fine}([\mathbf{O}_j, \mathbf{E}_j])
$$
The final splat rendering parameter is:
$$
\mathbf{G}'_j = \mathbf{G}_j + \Delta \mathbf{G}_j
$$
where $\mathbf{G}_j$ is the initial rendering parameter [S2].

### 3D Gaussians Prediction from Voxels
Predicts a set of learnable Gaussian parameters for each voxel $v$, including center offset, opacity, covariance, and spherical harmonic color [S3]. The output dimension per Gaussian is 38, comprising offset (3), opacity (1), covariance (6), and spherical harmonic color (28) [S3]. The final 3D Gaussian center $\mu_j$ is computed by:
$$
\mu_j = r \cdot (\sigma(\bar{\mu}_j) - 0.5) + \text{Center}_j
$$
where $\sigma$ is sigmoid, $\bar{\mu}_j$ is the predicted offset, $\text{Center}_j$ is the voxel centroid, and $r$ is a scaling factor set to 3 times the voxel size [S3]. The sigmoid activation restricts the learnable offset to a localized neighborhood; the -0.5 shift facilitates symmetrical, bi-directional movement from the voxel center [S3].

### GS-LRM / Long-LRM
Predicts 3D Gaussians directly from input images, bypassing the triplane intermediate representation [S4, unofficial]. This approach results in faster downstream rendering compared to methods that convert triplanes to Gaussians [S4, unofficial].

## Operating characteristics
-   **Time Complexity:** Rendering time is proportional to the number of sampled points per ray and the number of MPI depth layers. Sparse grids reduce computation in empty regions.
-   **Memory Complexity:** Dominated by the explicit grid storage (density, features, act shift). Memory usage scales with $world\_size$ and $mpi\_depth$.
-   **Performance:** Significantly faster than implicit-only methods like NeuralRGB-D [S1, unofficial]. Supports sparse grids for efficiency [S1, unofficial].
-   **Strengths:** Efficient volume rendering with MPI layers; explicit grid features allow for faster convergence; supports sparse grids and occupancy caching.
-   **Limitations:** Larger memory footprint compared to implicit methods; requires careful tuning of grid resolution and MPI depth.
-   **When to Use:** When fast convergence and high-fidelity surface reconstruction are needed, and memory is not the primary constraint.
-   **When Not to Use:** When memory is extremely limited or when implicit representations are sufficient for the application.

## Background
DirectMPIGO was introduced by Kakao Brain as part of the Nerf-Factory repository [S1, unofficial]. It extends the Direct Voxel Grid Optimization (DVGO) method by incorporating Multi-Plane Image (MPI) depth layers, allowing for more efficient volume rendering and better handling of depth information [S1, unofficial]. Unlike DVGO, which uses dense voxel grids, DirectMPIGO supports sparse grids and MPI depth, improving both speed and quality [S1, unofficial].

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S3] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
[S4] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
