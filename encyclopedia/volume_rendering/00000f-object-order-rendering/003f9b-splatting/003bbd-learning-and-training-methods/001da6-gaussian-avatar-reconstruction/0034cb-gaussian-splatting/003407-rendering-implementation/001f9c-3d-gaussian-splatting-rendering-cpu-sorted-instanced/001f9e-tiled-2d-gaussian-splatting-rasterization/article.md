# Tiled 2D Gaussian Splatting Rasterization

Also called: 3D Gaussian Rasterization (Forward Pass), 3D Gaussian Splatting Forward Rendering Pipeline, 3DGS Rasterization, CUDA Gaussian Rasterizer, CudaRasterizer::Rasterizer::forward, Gaussian Rasterization (GPU), Tile-based Gaussian Rasterization, Tiled Rendering of 3DGS

## Overview
Tiled 2D Gaussian Splatting Rasterization is a GPU-optimized, tile-based rendering method for 3D Gaussian Splatting (3DGS) that partitions the image plane into macro-tiles (typically 16×16 or 32×32 pixels) to efficiently composite sorted Gaussian primitives via iterative alpha-blending [S1, S2, S3, S4, S5, S6, unofficial]. It solves the performance bottleneck of evaluating millions of splats against every pixel by binning Gaussians to tiles, sorting them by depth, and processing each tile in parallel using CUDA thread blocks with shared memory synchronization [S1, S2, S4, S5, unofficial]. Unlike its sibling [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) which relies on standard graphics pipeline alpha blending, or [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) which iterates over Gaussians, this method explicitly manages memory and computation per tile to enable real-time training and rendering [S1, S2, S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian parameters (means $\mathbf{x}$, scales $\mathbf{s}$, rotations $\mathbf{r}$, opacities $\alpha_{logit}$, spherical harmonics coefficients $\mathbf{c}_{SH}$), camera parameters (view matrix $\mathbf{V}$, projection matrix $\mathbf{P}$, focal lengths $f_x, f_y$, image dimensions $W, H$), and background color $\mathbf{b}$ [S1, S4, S5, S6, unofficial].
*   **Outputs:** Rendered color image $\mathbf{C}$ (shape $[3, H, W]$), accumulated alpha $\mathbf{A}$ (shape $[H, W]$), and auxiliary buffers for gradients (depth, surface distance, etc.) [S4, S5, S6, unofficial].

**Data Structures:**
The method relies on three primary state buffers allocated on the GPU [S4, S5, unofficial]:
1.  **GeometryState** (per-Gaussian, size $P$):
    *   `depths`: float, view-space depth of Gaussian mean.
    *   `means2D`: float2, projected 2D center $(u, v)$.
    *   `conic_opacity`: float4, inverse 2D covariance matrix components $(a, b, c)$ and pre-computed opacity term. Specifically, the conic matrix $\mathbf{C}^{-1} = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$ [S4, S5, S6, unofficial].
    *   `rgb`: float3, pre-computed color from SH evaluation.
    *   `tiles_touched`: uint32_t, count of tiles the Gaussian overlaps.
    *   `point_offsets`: uint32_t, starting index in the flattened binning list.
2.  **BinningState** (sorting buffers):
    *   `point_list_keys_unsorted`: uint64_t, 64-bit key encoding tile ID and Gaussian index.
    *   `point_list_unsorted`: uint32_t, Gaussian index.
    *   `point_list_keys`: uint64_t, sorted keys.
    *   `point_list`: uint32_t, sorted Gaussian indices.
    *   `list_sorting_space`: char*, temporary space for radix sort.
3.  **ImageState** (per-pixel/tile):
    *   `accum_alpha`: float, accumulated transmittance (initially 1.0).
    *   `n_contrib`: uint32_t, number of Gaussians contributing to the pixel.
    *   `ranges`: uint2, start and end indices in `point_list` for each tile.

**Numbered Steps:**

1.  **Preprocessing and Projection:**
    For each Gaussian $i$ with $P$ total Gaussians:
    a. Transform mean $\mathbf{x}_i$ to camera space: $\mathbf{x}_i^c = \mathbf{V} \mathbf{x}_i$.
    b. Rotate 3D covariance $\Sigma_i$ to camera space: $\Sigma_i^c = \mathbf{R}_V \Sigma_i \mathbf{R}_V^T$.
    c. Project to 2D screen space using the Jacobian of the perspective projection to compute the 2D covariance $\Sigma_i^{2D}$ [S4, S6, unofficial].
    d. Compute the 2D conic (inverse covariance) $\mathbf{C}_i^{-1} = (\Sigma_i^{2D})^{-1} = \begin{bmatrix} a_i & b_i \\ b_i & c_i \end{bmatrix}$ [S4, S5, S6, unofficial].
    e. Evaluate Spherical Harmonics to get color $\mathbf{c}_i$ and apply opacity sigmoid: $\alpha_i = \sigma(\alpha_{logit, i})$ [S4, S6, unofficial].
    f. Compute axis-aligned bounding box (AABB) or bounding radius to determine which tiles the Gaussian overlaps. A Gaussian is assigned to a tile if its ellipse intersects the tile [S1, S4, S6, unofficial].
    g. Generate sorting keys: For each overlapping tile $t$, create a 64-bit key combining tile ID and depth (for sorting), and duplicate the Gaussian index [S4, S5, unofficial].

2.  **Binning and Sorting:**
    a. Collect all (key, value) pairs from all Gaussians into unsorted arrays `point_list_keys_unsorted` and `point_list_unsorted` [S4, S5, unofficial].
    b. Perform a device-side radix sort on `point_list_keys_unsorted` to group Gaussians by tile and sort them by depth within each tile [S4, S5, unofficial].
    c. Identify tile ranges: Scan the sorted keys to find the start and end indices for each tile in the sorted `point_list` array, storing these in `ImageState.ranges` [S4, S5, unofficial].

3.  **Tile-Based Rasterization (Forward Pass):**
    a. Launch a CUDA kernel where each thread block is assigned to one tile [S1, S2, S4, unofficial].
    b. Each thread block loads the sorted Gaussians for its tile into shared memory `Shmem` [S2, S4, unofficial].
    c. For each pixel $(u, v)$ in the tile (mapped to a thread):
        i. Initialize accumulated transmittance $T = 1.0$ and color $\mathbf{C} = \mathbf{b}$ [S4, S6, unofficial].
        ii. Iterate through the sorted Gaussians for this tile. For each Gaussian $i$:
            - Compute squared Mahalanobis distance: $dist\_sq = a_i (u - u_i)^2 + 2 b_i (u - u_i)(v - v_i) + c_i (v - v_i)^2$ [S6, unofficial].
            - Compute opacity contribution: $\alpha_i^{pixel} = \alpha_i \exp(-\frac{1}{2} dist\_sq)$ [S7, unofficial].
            - Update color and transmittance:
              $$\mathbf{C} \leftarrow \mathbf{C} + \mathbf{c}_i \alpha_i^{pixel} T$$
              $$T \leftarrow T (1 - \alpha_i^{pixel})$$
            - Early termination: If $T < \epsilon$ (e.g., 0.01), break the loop [S1, S6, unofficial].
        iii. Store final $\mathbf{C}$ and $1-T$ (alpha) to image buffers [S4, S6, unofficial].
    d. Synchronize threads within the block to ensure shared memory consistency [S2, unofficial].

4.  **Background Compositing:**
    Add the background color weighted by the final transmittance: $\mathbf{C}_{final} = \mathbf{C} + \mathbf{b} (1 - \mathbf{A})$ [S6, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def tiled_gaussian_rasterization(
    means2D: torch.Tensor,      # [P, 2]
    conics: torch.Tensor,       # [P, 3] (a, b, c)
    opacities: torch.Tensor,    # [P]
    colors: torch.Tensor,       # [P, 3]
    depths: torch.Tensor,       # [P]
    tile_ranges: torch.Tensor,  # [num_tiles, 2] (start, end indices in sorted list)
    sorted_gaussian_indices: torch.Tensor, # [num_rendered]
    image_width: int,
    image_height: int,
    tile_size: int = 16,
    bg_color: torch.Tensor = torch.tensor([0.0, 0.0, 0.0])
):
    """
    Simulates the tiled rasterization logic for a single tile.
    In practice, this is executed in parallel by CUDA threads.
    """
    num_tiles_x = math.ceil(image_width / tile_size)
    num_tiles_y = math.ceil(image_height / tile_size)
    
    # Initialize output buffers
    out_color = torch.zeros((3, image_height, image_width), dtype=torch.float32)
    out_alpha = torch.zeros((image_height, image_width), dtype=torch.float32)
    
    # Iterate over tiles (in reality, each tile is a CUDA block)
    for tile_y in range(num_tiles_y):
        for tile_x in range(num_tiles_x):
            tile_idx = tile_y * num_tiles_x + tile_x
            
            # Get range of Gaussians for this tile from the sorted list
            start_idx = tile_ranges[tile_idx, 0].item()
            end_idx = tile_ranges[tile_idx, 1].item()
            
            if start_idx >= end_idx:
                continue
                
            # Extract Gaussians for this tile
            tile_gaussian_indices = sorted_gaussian_indices[start_idx:end_idx]
            tile_means2D = means2D[tile_gaussian_indices]
            tile_conics = conics[tile_gaussian_indices]
            tile_opacities = opacities[tile_gaussian_indices]
            tile_colors = colors[tile_gaussian_indices]
            
            # Pixel bounds for this tile
            u_start = tile_x * tile_size
            v_start = tile_y * tile_size
            u_end = min(u_start + tile_size, image_width)
            v_end = min(v_start + tile_size, image_height)
            
            # Process pixels in this tile
            for v_pix in range(v_start, v_end):
                for u_pix in range(u_start, u_end):
                    accumulated_color = bg_color.clone()
                    accumulated_transmittance = 1.0
                    
                    # Iterate over Gaussians sorted by depth
                    for i in range(len(tile_gaussian_indices)):
                        # Early termination if transmittance is low
                        if accumulated_transmittance < 0.01:
                            break
                            
                        u_mean = tile_means2D[i, 0]
                        v_mean = tile_means2D[i, 1]
                        a, b, c = tile_conics[i, 0], tile_conics[i, 1], tile_conics[i, 2]
                        alpha_logit = tile_opacities[i]
                        color = tile_colors[i]
                        
                        # Compute opacity at pixel
                        du = u_pix - u_mean
                        dv = v_pix - v_mean
                        dist_sq = a * du**2 + 2 * b * du * dv + c * dv**2
                        
                        # Sigmoid opacity and Gaussian falloff
                        alpha_pixel = torch.sigmoid(alpha_logit) * torch.exp(-0.5 * dist_sq)
                        
                        # Alpha blending
                        accumulated_color = accumulated_color + color * alpha_pixel * accumulated_transmittance
                        accumulated_transmittance = accumulated_transmittance * (1 - alpha_pixel)
                    
                    # Store results
                    out_color[:, v_pix, u_pix] = accumulated_color
                    out_alpha[v_pix, u_pix] = 1 - accumulated_transmittance
                    
    return out_color, out_alpha
```

**Parameters:**
*   `tile_size`: Default 16×16 or 32×32 pixels [S1, S6, unofficial].
*   `sigma_multiplier`: Multiplier for Gaussian extent (e.g., 3.0) to determine bounding box size [S6, unofficial].
*   `epsilon`: Transmittance threshold for early termination (e.g., 0.01) [S1, S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-RGB Channels:** If the scene has non-RGB channels (`NUM_CHANNELS != 3`) and `colors_precomp` is not provided, the method throws a runtime error [S4, S5, unofficial].
*   **Invisible Gaussians:** Gaussians with `tiles_touched == 0` must be skipped during key generation to avoid invalid memory writes [S5, unofficial].
*   **Precision:** Floating-point precision in the Mahalanobis distance calculation can affect small Gaussians [S6, unofficial].
*   **Camera Model:** Restricted to pinhole cameras; cannot compute arbitrary light paths [S8].

## Subtopics

### [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md)
A rasterization method that iterates over Gaussians rather than tiles, using axis-aligned bounding boxes to limit pixel iteration and transmittance-based early ray termination for performance optimization. It produces visually identical results to tiled rasterization in most cases but is significantly faster due to early stopping and better culling.

### [Gaussian Splat Projection and Sorting](3369-gaussian-splat-projection-and-sorting.md)
Gaussian Splat Projection and Sorting is a GPU compute shader pass that projects 3D Gaussian splats into 2D screen space, culls invisible splats, computes view-dependent lighting, and prepares data for tile-based sorting. It differs from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by handling the geometric projection, visibility culling, and sorting key generation, whereas the parent handles the actual pixel-level rasterization within tiles. It differs from [Gaussian Splat View Data Calculation](10400-gaussian-splat-view-data-calculation.md) by focusing on the projection and sorting preparation rather than the evaluation of appearance (SH) and 4D state resolution, although both may coexist in different pipeline implementations.

### [Splat Batching Strategy](4296-splat-batching-strategy.md)
Splat Batching Strategy is a GPU optimization technique for 3D Gaussian Splatting that reduces draw call overhead by packing multiple Gaussian splats into a single instanced rendering command. It addresses the performance bottleneck of issuing individual draw calls for each Gaussian, which is inefficient when rendering scenes with tens or hundreds of thousands of splats. Unlike non-tiled approaches that may process all splats in one massive batch or tile-based methods that split work spatially, batching focuses on temporal efficiency by grouping splats into fixed-size chunks for instanced drawing.

### [Spirula Renderer](6909-spirula-renderer.md)
The Spirula Renderer is a GPU-accelerated Gaussian Splatting renderer implemented as a native Vulkan bridge (`librosplat_spirula.so`) called via Python's `ctypes`. It handles tile sorting, Gaussian rasterization, and rendering to Vulkan textures, explicitly excluding CUDA, PyTorch, `gsplat`, or OpenGL renderer paths. It builds on [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by performing the tile sorting and rasterization internally during the render call.

### [Depth Rendering with Constant Opacity](7462-depth-rendering-with-constant-opacity.md)
Depth Rendering with Constant Opacity is a specialized rendering pass designed to extract clean depth maps from 3D Gaussian Splatting models by forcing uniform opacity across all Gaussians. Unlike standard rendering which weights depth accumulation by per-Gaussian opacity, this method ensures that the rasterizer's depth accumulation is driven strictly by geometric distance, removing shading and opacity variations that can distort depth estimation. It serves as the foundation for depth-only passes in Gaussian Avatar Reconstruction pipelines, enabling accurate depth extraction before further processing like normalization or compositing.

### [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md)
Tile-based rasterization with radix sort is a GPU-parallel rendering method for 3D Gaussian Splatting that replaces per-pixel sorting with a single global radix sort to establish depth ordering per tile. It solves the performance bottleneck of evaluating millions of splats against every pixel by dividing the image into tiles (typically $16 \times 16$ or $32 \times 32$ pixels) and assigning each Gaussian to the tiles it overlaps, creating one work item per (Gaussian, tile) pair. Unlike [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) which processes pixels individually, or [Gaussian Splat Projection and Sorting](3369-gaussian-splat-projection-and-sorting.md) which may use less efficient sorting strategies, this method bins splats by tile ID and sorts them by depth in a single pass, enabling real-time performance for millions of splats by allowing one GPU thread block to process each tile's sorted list.

### [Gaussian Splatting Rendering with Camera Pose Override](7971-gaussian-splatting-rendering-with-camera-pose-override.md)
This method modifies the standard 3D Gaussian Splatting rasterization pipeline to decouple the rendering camera pose from the scene's stored camera parameters. It allows a pre-trained Gaussian model to be rendered from arbitrary viewpoints by injecting a custom world-to-camera transformation matrix (`camera_pose`) into the rasterization step, while keeping the Gaussian primitives fixed in their original world-space coordinates. This is distinct from [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md), which handles the tile-based memory management, by specifically addressing the coordinate frame transformation required for novel view synthesis or pose correction.

### [Half-cosine Squared Splatting](9915-half-cosine-squared-splatting.md)
Half-cosine Squared Splatting is a radial basis function (RBF) technique for 3D Gaussian Splatting that replaces the standard Gaussian decay with a half-cosine squared function. It solves the problem of slow convergence and high memory usage in standard splatting by providing higher opacity values per primitive, allowing fewer primitives to achieve the same accumulated opacity. It sets itself apart from standard Gaussian Splatting by using a bounded, flatter decay profile that accelerates training and reduces memory footprint, while differing from its sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) by fundamentally altering the splat shape rather than just the culling strategy.

### [HSPT Cut and Gaussian Loading Optimization](10300-hspt-cut-and-gaussian-loading-optimization.md)
This method optimizes the data transfer and memory management overhead inherent in hierarchical 3D Gaussian Splatting rendering. It solves the problem of high latency caused by frequent, fragmented RAM-to-GPU transfers required to load visible Gaussians for each frame. By storing Spherical Primitive Tree (SPT) properties in a continuous GPU buffer and transferring Gaussian properties via a single bulk copy operation followed by GPU-side reorganization, it minimizes PCIe bus traffic and kernel launch overhead. It differs from its siblings by explicitly decoupling the hierarchical cut logic (handled via cached SPTs) from the rasterization-ready data layout (SoA), whereas methods like [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) focus on pixel-level culling and [Opacity-Thresholded Tile Intersection Filtering](2109-opacity-thresholded-tile-intersection-filtering.md) focus on tile-level geometric filtering.

### [DP-GES Rendering](12576-dp-ges-rendering.md)
DP-GES is a sort-free rendering method that combines 2D surfels with semi-transparent boundaries and 3D Gaussians to achieve high-fidelity novel view synthesis, eliminating aliasing and popping artifacts while enabling correct transmittance modulation. It sets itself apart from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) and sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) by introducing a two-stage pipeline: Stage 1 uses z-buffer-based depth peeling to extract the nearest three layers of depth, color, and opacity from surfels, and Stage 2 splats 3D Gaussians without sorting, modulating their weights by the transmittance from the peeled surfel layers. Unlike GES, which uses fully opaque surfels, DP-GES uses semi-transparent boundaries to mitigate aliasing from depth testing and allow differentiable joint optimization.

### [Per-fragment Voxel-Cell Mask Injection](12613-per-fragment-voxel-cell-mask-injection.md)
Per-fragment Voxel-Cell Mask Injection is a technique for carving 3D Gaussian Splatting scenes by injecting a per-fragment Signed Distance Field (SDF) evaluation into a compiled fragment shader. It solves the problem of creating clean, cube-shaped holes in a splat cloud without forking the underlying renderer or deleting whole splats. It differs from its sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) and the baseline [H2](2061-tiled-2d-gaussian-splatting-rasterization.md) approach by operating at the fragment level using a 3D texture lookup, producing sharp cube edges instead of the rough masking artifacts caused by per-splat deletion.

### [Thread-efficient Partial Rendering](13363-thread-efficient-partial-rendering.md)
Thread-efficient partial rendering is a GPU rasterization optimization for 3D Gaussian Splatting (3DGS) designed to reduce the computational overhead of multi-view training. It solves the problem of low GPU utilization caused by naive binary masking in tiled rendering, where idle threads wait for active threads to finish. Unlike naive multi-view training which renders all pixels for all views, or naive partial rendering with masks which leaves most threads in a block idle, this method maintains the same number of training pixels as single-view training while maximizing parallelization by assigning multiple GPU blocks to a single tile.

### [3D Gaussian Splatting Rasterization (GPU)](13469-3d-gaussian-splatting-rasterization-gpu.md)
A torch.autograd.Function implementing the forward and backward passes for Gaussian rasterization, handling argument restructuring and buffer management.

### [4D Gaussian Rasterization](13946-4d-gaussian-rasterization.md)
Renders a scene represented by 4D Gaussians (3D spatial + 1D temporal) into a 2D image for a specific viewpoint and timestamp. It handles time-dependent opacity, position offsets, and spherical harmonics evaluation in 4D.

### [vtkPointGaussianMapper](14148-vtkpointgaussianmapper.md)
A mapper that renders point clouds as simple points or anisotropic 3D Gaussian splats without requiring verts cells, supporting per-point scale, rotation, opacity, and emissive properties.

### Mobile-GS
A memory-centric rendering design for smartphones that loads only the Gaussians visible in the current view frustum from flash storage, uses a quantized integer representation for positions and colors, and performs the blending in a lower-precision shader [S1, unofficial].

### 4D Gaussian Splatting Motion Evaluation
A technique for rendering time-varying 3D captures (volumetric video) by storing static per-splat attributes and per-splat motion coefficients in extra textures, evaluated in the vertex shader from a time uniform, avoiding per-frame CPU sorting and texture uploads [S9, unofficial]. It stores static per-splat attributes (position, scale, rotation, color) in standard splat buffers, stores per-splat motion coefficients (linear or polynomial trajectory parameters) and temporal opacity window parameters in extra textures, passes the current playback time as a uniform to the vertex shader, evaluates the motion coefficients against the time uniform to compute the current position and opacity for each splat, and performs sorting only occasionally, as motions are assumed to be small enough to not require per-frame sorting for visual correctness [S9, unofficial].

### AABB Culling for Gaussian Splatting
A culling technique that uses an axis-aligned bounding box of the projected Gaussian ellipse to reduce ineffective rendering expenses, specifically addressing the inability of bounding circles to cull low-opacity pairs in the minor axis direction [S10]. It sets the partial derivatives to 0 to find extrema, substitutes the coordinate relationships into the ellipse function to solve for the maximum coordinate values $x_{max}$ and $y_{max}$, and culls tiles based on the axis-aligned bounding box constructed by $x_{max}$ and $y_{max}$, which correspond to the half-width $w$ and half-height $h$ of the bounding box [S10].

### Load Balancing for Pixel-parallel Splatting
A strategy to address uneven computational load across pixel threads in Gaussian rasterization by reducing the number of Gaussians rendered for heavy-load pixels and increasing it for light-load pixels [S10]. The sum of weights for remaining losses ($\lambda_{L1} + \lambda_{ssim}$) typically has a value of 0.55 [S10].

### Tight Quad Rasterization with Eigenvalue Decomposition
Represents each 3D Gaussian as a quad in Gaussian rasterization, decomposes eigenvalues of anisotropic 3D Gaussians into rotation and scale components (2D Eigenvalues in screenspace), and computes a non-axis-aligned quad and incorporates splat alpha value into the bound rectangle calculation for tighter fits [S11].

### Adaptive Region Selection
A rendering strategy for novel viewpoints that prioritizes local Gaussians with higher contributions to avoid floating artifacts and holes [S12]. It trains a coarse global Gaussian representation by downsampling point clouds and images and avoiding densification, renders novel viewpoint using local and global Gaussians separately, calculates matching score $S$ for each local-global Gaussian pair: $S = (1 - \lambda)L1 + \lambda LSSIM$, sorts local Gaussians in descending order based on matching scores, calculates projection opacity $o_i(p)$ for each pixel $p$ and Gaussian $G_i$, and applies the formula $w_i = o_i(p) \prod_{j=1}^{i-1} (1 - o_j(p))$ if $\sum_{j=1}^{i-1} o_j(p) < \beta$; $w_i = 0$ if $\sum_{j=1}^{i-1} o_j(p) \ge \beta$, where $\beta$ is a hyperparameter threshold for overlapped opacity [S12].

## Operating characteristics
*   **Time Complexity:** $O(P \log P)$ for sorting, where $P$ is the number of Gaussians, plus $O(T \cdot K \cdot N)$ for rasterization, where $T$ is the number of tiles, $K$ is the average number of Gaussians per tile, and $N$ is the number of pixels per tile [S1, S4, S5, unofficial].
*   **Memory Complexity:** $O(P)$ for Gaussian buffers and $O(W \cdot H)$ for image buffers [S4, S5, unofficial].
*   **Performance:** Enables real-time rendering (100+ FPS) for scenes with millions of Gaussians on modern GPUs [S1, S6, unofficial].
*   **Strengths:** High performance, differentiable for training, efficient memory usage via tiling [S1, S3, S4, unofficial].
*   **Limitations:** Restricted to pinhole cameras, linear scaling with Gaussian count regardless of size, potential for aliasing if not properly filtered [S8].
*   **When to Use:** Real-time training and rendering of 3DGS scenes [S1, S5, unofficial].
*   **When Not to Use:** Scenes requiring complex light transport or non-pinhole cameras [S8].

## Background
Tiled 2D Gaussian Splatting Rasterization was introduced as part of the 3D Gaussian Splatting (3DGS) paper by Kerbl et al. in 2023 [S1, unofficial]. It was developed to address the computational bottleneck of rendering millions of Gaussian primitives in real-time, enabling both fast training and interactive novel view synthesis. The method has since become the standard rasterization pipeline for 3DGS and its variants [S1, S3, S4, S5, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
[S3] OpenSplat – 3D gaussian splatting in C++ with CPU and GPU support, https://github.com/pierotofy/OpenSplat, fetched 2026-10-26 (unofficial)
[S4] Lewis-Stuart-11/3DGS-to-PC, https://github.com/Lewis-Stuart-11/3DGS-to-PC, fetched 2026-10-27 (unofficial)
[S5] XiaoBin2001/Improved-GS, https://github.com/XiaoBin2001/Improved-GS, fetched 2026-10-27 (unofficial)
[S6] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S7] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
[S8] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
[S9] Explore: Gaussian splatting, OpenUSD, AI world generation, volumetric video — study, https://github.com/sceneview/sceneview/issues/4023, fetched 2026-10-27 (unofficial)
[S10] AdR-Gaussian: Accelerating Gaussian Splatting with Adaptive Radius, https://doi.org/10.1145/3680528.3687675, fetched 2026-10-26
[S11] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
[S12] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
