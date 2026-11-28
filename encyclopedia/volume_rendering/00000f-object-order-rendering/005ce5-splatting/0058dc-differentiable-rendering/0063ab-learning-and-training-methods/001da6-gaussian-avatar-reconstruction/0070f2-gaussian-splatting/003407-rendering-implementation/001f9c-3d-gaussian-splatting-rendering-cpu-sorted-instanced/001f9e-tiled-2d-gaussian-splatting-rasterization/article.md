# Tiled 2D Gaussian Splatting Rasterization

Also called: 3D Gaussian Splatting Rasterization, Tile-based Gaussian Rasterization, Tiled Rendering of 3DGS, 3DGS Rasterization, Gaussian Rasterization (GPU)

## Overview
Tiled 2D Gaussian Splatting Rasterization is a GPU-optimized, differentiable rendering method for 3D Gaussian Splatting (3DGS) that partitions the image plane into macro-tiles to enable parallel, memory-efficient alpha blending [S1, S2, S8, unofficial]. It solves the performance bottleneck of sorting millions of splats globally by assigning each tile to a CUDA thread block, which cooperatively fetches depth-sorted Gaussians and composites them per-pixel [S1, S8, unofficial]. It sets itself apart from its parent [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md) by moving the sorting and blending entirely to the GPU, enabling real-time frame rates for training and inference [S10, unofficial][S11].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian parameters (means $\mathbf{x}$, opacities $\alpha$, scales $\mathbf{s}$, rotations $\mathbf{r}$), spherical harmonics coefficients (or precomputed colors), camera matrices (view $V$, projection $P$), image dimensions $(W, H)$, and field-of-view tangents ($\tan \phi_x, \tan \phi_y$).
*   **Outputs:** Rendered RGB image, auxiliary maps (depth, normals, opacity), and gradients for training.

**Data Structures:**
The method relies on three primary state buffers allocated on the GPU [S7, S8, unofficial]:
1.  **GeometryState:** Per-Gaussian buffers.
    *   `depths`: float, size $P$ (number of Gaussians).
    *   `means2D`: float2, size $P$ (screen-space centers).
    *   `cov3D`: float, size $P \times 6$ (symmetric 3D covariance).
    *   `conic_opacity`: float4, size $P$ (inverse 2D covariance + opacity).
    *   `rgb`: float, size $P \times 3$ (projected colors).
    *   `tiles_touched`: uint32_t, size $P$ (count of tiles intersected).
    *   `point_offsets`: uint32_t, size $P$ (start index in flattened tile list).
2.  **BinningState:** Sorting buffers.
    *   `point_list_keys`: uint64_t, size $P$ (encoded tile ID + depth).
    *   `point_list`: uint32_t, size $P$ (Gaussian indices).
3.  **ImageState:** Per-pixel/tile buffers.
    *   `accum_alpha`: float, size $W \times H$.
    *   `ranges`: uint2, size $N_{tiles}$ (start/end indices for each tile in the sorted list).

**Steps:**

1.  **Preprocessing & Projection:**
    For each Gaussian $i$:
    *   Transform mean $\mathbf{x}_i$ to camera space: $\mathbf{x}'_i = V \mathbf{x}_i$.
    *   Compute 3D covariance $\Sigma_i$ from scales and rotations.
    *   Project to 2D screen space using the Jacobian of the projection matrix $J$ and camera intrinsics $K$:
        $$ \Sigma'_i = J \begin{bmatrix} V & \mathbf{x}'_i \\ 0 & 1 \end{bmatrix} \Sigma_i \begin{bmatrix} V & \mathbf{x}'_i \\ 0 & 1 \end{bmatrix}^T J^T $$
        (Simplified: $\Sigma'_{2D} = J \Sigma_{3D} J^T$ restricted to the top-left $2 \times 2$ block after perspective division) [S1, S9, unofficial].
    *   Compute the 2D conic matrix (inverse covariance) $\mathbf{C}_i = (\Sigma'_{2D})^{-1}$ [S9, unofficial].
    *   Calculate the bounding radius $r_i = \lceil 3 \sqrt{\lambda_{\max}} \rceil$ where $\lambda_{\max}$ is the largest eigenvalue of $\Sigma'_{2D}$ [S1, unofficial].
    *   Compute screen-space color $\mathbf{c}_i$ by evaluating Spherical Harmonics at the view direction [S9, unofficial].

2.  **Tile Binning:**
    *   Determine the tile grid: $N_{tiles} = \lceil W / T_x \rceil \times \lceil H / T_y \rceil$, where $T_x, T_y$ are tile dimensions (typically 16x16 or 32x32) [S1, S9, unofficial].
    *   For each Gaussian, compute the axis-aligned bounding box (AABB) in tile coordinates.
    *   Assign the Gaussian to all overlapping tiles. Generate a 64-bit key for each Gaussian-tile pair: `key = (tile_id << 32) | depth_id`, where `depth_id` is the view-space depth $z_i$ [S8, unofficial][S11].
    *   Store pairs in `point_list_keys` and `point_list` [S8, unofficial].

3.  **Sorting:**
    *   Perform a device-side radix sort on `point_list_keys` to group Gaussians by tile and sort them front-to-back (increasing depth) [S1, S8, unofficial].
    *   Compute inclusive prefix sums on `tiles_touched` to determine `point_offsets`, which mark the start of each tile's Gaussian list in the sorted array [S7, unofficial].
    *   Scan the sorted keys to populate `ranges` in `ImageState`, storing the start and end indices for each tile [S8, unofficial].

4.  **Rasterization (Alpha Blending):**
    *   Launch a CUDA kernel where each thread block $(bx, by)$ is assigned to one tile [S1, unofficial].
    *   Each thread $i$ in the block corresponds to a pixel $(u, v)$ within the tile [S2, unofficial].
    *   Threads cooperatively iterate through the sorted Gaussian list for their tile (using `ranges` to bound the loop) [S1, unofficial].
    *   For each Gaussian $j$ overlapping pixel $(u, v)$, compute the squared Mahalanobis distance:
        $$ dist\_sq = \mathbf{d}^T \mathbf{C}_j \mathbf{d} = a \cdot du^2 + 2b \cdot du \cdot dv + c \cdot dv^2 $$
        where $\mathbf{d} = [u - u_j, v - v_j]^T$ and $\mathbf{C}_j = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$ [S9, unofficial].
    *   Compute opacity contribution:
        $$ \alpha_j = \sigma(o_j) \exp(-\frac{1}{2} dist\_sq) $$
        where $\sigma$ is the sigmoid function and $o_j$ is the log-opacity [S10, unofficial].
    *   Accumulate color using alpha blending:
        $$ C(u, v) = \sum_{j=1}^{N} \mathbf{c}_j \alpha_j \prod_{k=1}^{j-1} (1 - \alpha_k) $$
        [S1, S10, unofficial].
    *   Early termination: If the accumulated transmittance $\prod (1-\alpha_k)$ drops below a threshold (e.g., $1/255$), stop processing further Gaussians for that pixel [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def tiled_gaussian_rasterization(
    means3D: torch.Tensor,      # [P, 3]
    opacities: torch.Tensor,    # [P]
    scales: torch.Tensor,       # [P, 3]
    rotations: torch.Tensor,    # [P, 4] (quaternion)
    shs: torch.Tensor,          # [P, K, 3] or None
    viewmatrix: torch.Tensor,   # [4, 4]
    projmatrix: torch.Tensor,   # [4, 4]
    focal_x: float,
    focal_y: float,
    tan_fovx: float,
    tan_fovy: float,
    image_height: int,
    image_width: int,
    tile_size: int = 16,
    bg_color: torch.Tensor = torch.tensor([0.0, 0.0, 0.0])
):
    P = means3D.shape[0]
    device = means3D.device
    
    # 1. Precompute 2D Covariance and Conics
    # Transform means to camera space
    means_cam = (viewmatrix[:3, :3] @ means3D.T + viewmatrix[:3, 3:4]).T
    
    # Compute 3D Covariance from scales and rotations
    # (Simplified: assume cov3D_precomp is available or computed via rotation matrix R)
    # R = quaternion_to_rotation_matrix(rotations)
    # S = diag(exp(scales))
    # Sigma_3D = R @ S @ S.T @ R.T
    
    # Project to 2D
    # J = d(proj) / d(cam_pos)
    # Sigma_2D = J @ Sigma_3D @ J.T
    # Conic C = inv(Sigma_2D)
    
    # 2. Binning
    # Compute bounding boxes in tile coordinates
    # Assign Gaussians to tiles
    # Generate keys: (tile_id << 32) | depth
    
    # 3. Sorting
    # Sort keys and values (Gaussian indices)
    # Compute ranges for each tile
    
    # 4. Rasterization Kernel Launch
    # Block: (tile_size, tile_size)
    # Grid: (ceil(W/tile_size), ceil(H/tile_size))
    
    # Inside Kernel:
    # tile_id = blockIdx.x + blockIdx.y * gridDim.x
    # pixel_u = threadIdx.x + blockIdx.x * tile_size
    # pixel_v = threadIdx.y + blockIdx.y * tile_size
    
    # if pixel_u >= W or pixel_v >= H: return
    
    # start_idx = ranges[tile_id].x
    # end_idx = ranges[tile_id].y
    
    # acc_alpha = 1.0
    # acc_color = bg_color[:, pixel_v, pixel_u]
    
    # for k in range(start_idx, end_idx):
    #     g_idx = sorted_gaussian_indices[k]
    #     if g_idx is invalid: break
    #     
    #     # Check if pixel is inside Gaussian bounding box (optional optimization)
    #     # Compute dist_sq
    #     alpha = sigmoid(opacities[g_idx]) * exp(-0.5 * dist_sq)
    #     
    #     if alpha < 1e-4: continue
    #     
    #     color = shs[g_idx] # or precomputed
    #     acc_color += color * alpha * acc_alpha
    #     acc_alpha *= (1.0 - alpha)
    #     
    #     if acc_alpha < 1e-4: break
    #     
    # out_image[:, pixel_v, pixel_u] = acc_color
    # out_alpha[:, pixel_v, pixel_u] = 1.0 - acc_alpha
    
    return out_image, out_alpha
```

## Subtopics

### [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md)
A rasterization method that iterates over Gaussians rather than tiles, using axis-aligned bounding boxes to limit pixel iteration and transmittance-based early ray termination for performance optimization. It produces visually identical results to tiled rasterization in most cases but is significantly faster due to early stopping and better culling.

### [Gaussian Splat Projection and Sorting](3369-gaussian-splat-projection-and-sorting.md)
A GPU compute shader pass that projects 3D Gaussian splats into 2D screen space, culls invisible splats, computes view-dependent lighting, and prepares data for tile-based sorting. It differs from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by handling the geometric projection, visibility culling, and sorting key generation, whereas the parent handles the actual pixel-level rasterization within tiles [G]. It differs from [Gaussian Splat View Data Calculation](10400-gaussian-splat-view-data-calculation.md) by focusing on the projection and sorting preparation rather than the evaluation of appearance (SH) and 4D state resolution, although both may coexist in different pipeline implementations [G].

### [Spirula Renderer](6909-spirula-renderer.md)
The Spirula Renderer is a GPU-accelerated Gaussian Splatting renderer implemented as a native Vulkan bridge (`librosplat_spirula.so`) called via Python's `ctypes`. It handles tile sorting, Gaussian rasterization, and rendering to Vulkan textures. It explicitly excludes CUDA, PyTorch, `gsplat`, or OpenGL renderer paths. It builds on [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by performing the tile sorting and rasterization internally during the render call.

### [Depth Rendering with Constant Opacity](7462-depth-rendering-with-constant-opacity.md)
Depth Rendering with Constant Opacity is a specialized rendering pass for 3D Gaussian Splatting that produces clean depth maps by forcing all Gaussians to have uniform opacity and color. Unlike standard alpha-blending which weights depth by Gaussian opacity and transmittance, this method ensures that depth accumulation is driven purely by geometric distance and rasterization order, eliminating shading artifacts and opacity-induced depth bias. It is distinct from [Depth Rendering with Variable Opacity](12622-depth-rendering-with-variable-opacity.md), which respects the learned alpha values of the Gaussians.

### [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md)
Tile-based Rasterization with Radix Sort is a GPU-parallel rasterization method for 3D Gaussian Splatting that divides the image into tiles to manage memory and processing load, assigning Gaussians to tiles based on their projected 2D splats [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md). It solves the performance bottleneck of per-pixel sorting by using a single global radix sort to achieve per-tile depth ordering, enabling real-time performance for millions of splats [Tile-based Rasteriser](2846-tile-based-rasteriser.md). Unlike sibling methods that may perform redundant depth sorting or evaluate splats against irrelevant pixels, this method ensures that each tile is processed by one GPU thread block walking a single sorted list [Tile-based Rasteriser](2846-tile-based-rasteriser.md).

### [Gaussian Splatting Rendering with Camera Pose Override](7971-gaussian-splatting-rendering-with-camera-pose-override.md)
This method modifies the standard 3D Gaussian Splatting rasterization pipeline to allow the camera pose to be overridden by an external tensor during rendering, rather than using the stored viewpoint parameters. It solves the problem of rendering a pre-trained Gaussian model from arbitrary or dynamically computed camera positions (e.g., from a SLAM system or a differentiable reconstruction loop) without retraining the model. It sets apart from standard rendering by injecting a custom world-to-camera transformation (`rel_w2c`) and quaternion rotation into the projection and covariance computation steps, enabling differentiable rendering with respect to the camera pose.

### [Load Balancing for Pixel-parallel Splatting](8373-load-balancing-for-pixel-parallel-splatting.md)
A strategy to address uneven computational load across pixel threads in Gaussian rasterization by reducing the number of Gaussians rendered for heavy-load pixels and increasing it for light-load pixels. It sets itself apart from sibling methods like [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by explicitly optimizing the *count* of splats per pixel to balance GPU thread utilization, rather than just spatially partitioning the image.

### [Half-cosine Squared Splatting](9915-half-cosine-squared-splatting.md)
Half-cosine Squared Splatting is a radial basis function technique for 3D Gaussian Splatting that replaces the standard Gaussian decay with a half-cosine squared function. It solves the problem of slow convergence and high memory usage in standard splatting by providing higher opacity values per primitive, allowing fewer primitives to achieve the same accumulated opacity. It differs from standard Gaussian Splatting by using a bounded decay profile that accelerates training and reduces memory footprint.

### [HSPT Cut and Gaussian Loading Optimization](10300-hspt-cut-and-gaussian-loading-optimization.md)
This method optimizes the HSPT cut and Gaussian loading by storing SPT properties in a continuous GPU buffer and transferring Gaussians via a single copy operation, reorganizing them on the GPU into a Structure of Arrays (SoA) layout. It reduces PCIe bus traffic and kernel launch overhead inherent in hierarchical 3D Gaussian Splatting rendering.

### [Tight Quad Rasterization with Eigenvalue Decomposition](10699-tight-quad-rasterization-with-eigenvalue-decomposition.md)
This method renders 3D Gaussians as tightly fitted, non-axis-aligned quads to minimize overdraw and improve rendering performance, particularly on mobile hardware. It differs from standard axis-aligned bounding box approaches by computing a rotated rectangle that incorporates the splat's alpha value, ensuring only relevant pixels are processed.

### [DP-GES Rendering](12576-dp-ges-rendering.md)
DP-GES is a sort-free rendering method that combines 2D surfels with semi-transparent boundaries and 3D Gaussians to achieve high-fidelity novel view synthesis, eliminating aliasing and popping artifacts while enabling correct transmittance modulation. It sets itself apart from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) and sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) by introducing a two-stage pipeline: Stage 1 uses z-buffer-based depth peeling to extract the nearest three layers of depth, color, and opacity from surfels, and Stage 2 splats 3D Gaussians without sorting, modulating their weights by the transmittance from the peeled surfel layers. Unlike Gaussian-Enhanced Surfels (GES), which uses fully opaque surfels, DP-GES uses semi-transparent boundaries to mitigate aliasing from depth testing and allow differentiable joint optimization.

### [Per-fragment Voxel-Cell Mask Injection](12613-per-fragment-voxel-cell-mask-injection.md)
Per-fragment Voxel-Cell Mask Injection is a technique for carving cube-shaped holes in 3D Gaussian Splatting scenes by injecting a Signed Distance Field (SDF) evaluation into the fragment shader at runtime. It solves the visual artifacts caused by deleting whole splats (the "H2" baseline) by operating at the fragment level, producing clean, sharp cube edges without forking the underlying renderer. Unlike its sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) which manages opacity globally, this method uses a 3D texture lookup to discard specific fragments, leaving neighboring voxels untouched.

### [Thread-efficient Partial Rendering](13363-thread-efficient-partial-rendering.md)
Thread-efficient partial rendering is a GPU rasterization optimization for 3D Gaussian Splatting (3DGS) designed to reduce the computational overhead of multi-view training. It solves the problem of low GPU utilization caused by naive binary masking in tiled rendering, where idle threads wait for active threads to finish. Unlike naive multi-view training which renders all pixels for all views, or naive partial rendering with masks which leaves most threads in a block idle, this method maintains the same number of training pixels as single-view training while maximizing parallelization by assigning multiple GPU blocks to a single tile.

### [3D Gaussian Splatting Rasterization (GPU)](13469-3d-gaussian-splatting-rasterization-gpu.md)
A differentiable rasterization method for 3D Gaussians that sorts Gaussians by tile and depth to perform alpha blending in parallel per tile. It solves the problem of efficiently rendering millions of 3D Gaussian splats to a 2D screen while maintaining exact gradients for all Gaussian parameters (means, opacities, scales, rotations, spherical harmonics) via a CUDA backend. It sets itself apart from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by implementing the specific sorting, binning, and tile-based parallel rendering kernels required for the standard 3DGS forward and backward passes, often exposed as a `torch.autograd.Function`.

### [vtkPointGaussianMapper](14148-vtkpointgaussianmapper.md)
`vtkPointGaussianMapper` renders point clouds as simple points or anisotropic 3D Gaussian splats without requiring vertex cells, supporting per-point scale, rotation, opacity, and emissive properties. It differs from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by operating in object-order on the CPU-side data structure and projecting Gaussians via a vertex shader rather than GPU-parallel tile-based rasterization, and from [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) by lacking transmittance culling and relying on standard OpenGL blending.

### [Feature 3DGS Rendering](15900-feature-3dgs-rendering.md)
Feature 3DGS is a method for rendering high-dimensional semantic feature maps alongside RGB images using 3D Gaussians, enabling the distillation of knowledge from 2D foundation models (such as SAM or LSeg) into an explicit 3D scene representation. It sets itself apart from NeRF-based feature field methods by using an explicit representation that avoids interference between the radiance and feature fields, allowing for equal-weighted joint optimization without the sensitivity to hyperparameters required by implicit neural networks.

### [3D Gaussian Splatting (glTF KHR_gaussian_splatting)](16403-3d-gaussian-splatting-gltf-khr-gaussian-splatting.md)
This method defines the rendering pipeline for 3D Gaussian splats stored in glTF assets via the `KHR_gaussian_splatting` extension, treating splats as point primitives with specific geometric and color attributes. It differs from general rasterization implementations by enforcing a strict data layout and transformation constraints defined by the glTF standard, ensuring interoperability and fallback to sparse point cloud rendering if the extension is unsupported.

### [Softmax-GS Rendering Algorithm](17684-softmax-gs-rendering-algorithm.md)
Softmax-GS is a rendering method for 3D Gaussian Splatting that employs a softmax-based competition mechanism to handle overlapping Gaussians, aiming to provide order-invariance and smooth transitions at Gaussian boundaries. Unlike standard 3D Gaussian Splatting, which suffers from popping artifacts due to sorting dependencies, or StopThePop, which exhibits abrupt color changes, Softmax-GS produces flicker-free, smooth transitions by treating overlapping splats as a competitive blending process rather than a strict depth-sorted accumulation. It builds on [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by replacing the standard alpha blending step with this differentiable, order-invariant softmax competition.

### [XPR-3DGS](17685-xpr-3dgs.md)
XPR-3DGS is a point-based differentiable rendering method that implements 3D Gaussian Splatting within the XPR framework. It projects 3D Gaussians to 2D, culls tiles based on conic bounds, and evaluates alpha blending per pixel. It sets itself apart from its siblings by providing a high-level, Python-based implementation that requires significantly fewer lines of code than traditional C++/CUDA implementations, while maintaining comparable visual fidelity.

### [Stochastic Solid Volume Rendering](17691-stochastic-solid-volume-rendering.md)
Stochastic Solid Volume Rendering (SSVR) is a volume rendering technique used as an alternative to vanilla 3D Gaussian Splatting (3DGS) rasterization to improve geometric fidelity, particularly for depth and surface reconstruction. It produces RGB images nearly identical to standard 3DGS rasterization under suitable conditions but offers geometric advantages by reducing holes and blurred boundaries in depth maps. It is equivalent to 3DGS rasterization when an appropriate vacancy function is applied.

### [Mip-Splatting Rasterization Pipeline](17970-mip-splatting-rasterization-pipeline.md)
The Mip-Splatting Rasterization Pipeline is a differentiable GPU-based rendering method for 3D Gaussian Splatting that integrates view-dependent scaling (mip-splatting) to handle aliasing and resolution changes. It extends the tiled rasterization approach by using a `kernel_size` parameter to adjust the Gaussian projection scale, ensuring consistent visual quality across different camera distances and resolutions. It builds on [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by adding specific preprocessing for mip-levels and a robust binning/sorting mechanism for tile-based rendering.

### [SimpleRenderer (CPU Gaussian Splatting)](19078-simplerenderer-cpu-gaussian-splatting.md)
SimpleRenderer is a CPU-based rendering method for 3D Gaussian point clouds that approximates Gaussian Splatting using a painter's algorithm. It projects 3D Gaussians to 2D screen ellipses and composites them to generate an RGB image, suitable for visualization and small-scale point clouds where GPU acceleration is unavailable. Unlike standard 3DD Gaussian Splatting which relies on CUDA-based tile-based rasterization for performance and differentiability, SimpleRenderer uses a simplified 2D ellipse approximation and non-differentiable alpha blending.

### [Custom Shadow Mapping for Satellite Imagery](19452-custom-shadow-mapping-for-satellite-imagery.md)
This method casts geometrically consistent shadows in 3D Gaussian Splatting (3DGS) by comparing elevation renders from a satellite camera and a "sun camera," avoiding ray marching which is incompatible with the splatting primitive. It differs from NeRF-based shadowing (e.g., EO-NeRF) by relying solely on rasterization and elevation comparison rather than volumetric ray marching.

### [Bounded Execution of Rasterization](20070-bounded-execution-of-rasterization.md)
Bounded Execution of Rasterization (BoundedGS) is a memory-constrained rasterization strategy for 3D Gaussian Splatting that partitions the image into non-overlapping leaves to bound peak memory usage. It solves the problem of high memory peaks during training by processing leaves sequentially, releasing local state after each leaf, and accumulating gradients into a shared buffer. It sets itself apart from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by explicitly separating the rasterization and projection backward passes, ensuring that rasterization caches do not coexist with projection gradients, thereby reducing the mean working set significantly.

### [CoherentRaster Rasterization Pipeline](23406-coherentraster-rasterization-pipeline.md)
CoherentRaster is a GPU-parallel rasterization pipeline designed for efficient 3D Gaussian Splatting on light field displays (LFDs), synthesizing interlaced light field images by separating view-dependent attributes from cluster-shared ones. It distinguishes itself from standard 3D Gaussian Splatting [13469-3d-gaussian-splatting-gpu.md] and tiled variants [2061-tiled-2d-gaussian-splatting-rasterization.md] by employing Cross-view Coherent Attribute Reuse to eliminate redundant projection computations and View-coherent Remapping to enable coalesced memory accesses during alpha blending.

### [Rasterizing Depth for Gaussian Splats](23407-rasterizing-depth-for-gaussian-splats.md)
A method to compute a spatially varying depth within the projected 2D Gaussian, rather than using a constant depth per Gaussian splat, to capture shape details. Standard Gaussian Splatting evaluates the depth of each 2D Gaussian by its center to sort them for alpha blending; this constant depth cannot capture shape details. This method derives a rasterized formula for efficient evaluation of spatially varying depths within a projected Gaussian.

## Operating characteristics
*   **Time Complexity:** $O(P \log P)$ for sorting, where $P$ is the number of Gaussians. Rasterization is $O(T \cdot N_{avg})$, where $T$ is the number of tiles and $N_{avg}$ is the average number of Gaussians per tile.
*   **Memory Complexity:** $O(P + W \cdot H)$ for storing geometry, binning, and image buffers [S7, S8, unofficial].
*   **Performance:** Enables real-time rendering (60+ FPS) for scenes with millions of Gaussians on modern GPUs [S1, unofficial].
*   **Strengths:** High parallelism, differentiable, efficient memory usage via tiling.
*   **Limitations:** Restricted to pinhole cameras; sorting overhead can be significant for very sparse scenes; memory usage scales with image resolution [S12].

## Background
Originated with the 3D Gaussian Splatting paper by Kerbl et al. (2023) [S1, unofficial]. It was developed to enable real-time differentiable rendering of 3D Gaussian primitives, overcoming the slowness of volume rendering and the lack of differentiability in standard rasterization. It is widely used in 3D reconstruction, novel view synthesis, and dynamic scene modeling.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
[S3] OpenSplat – 3D gaussian splatting in C++ with CPU and GPU support, https://github.com/pierotofy/OpenSplat, fetched 2026-10-26 (unofficial)
[S4] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
[S5] GAP-LAB-CUHK-SZ/gaustudio, https://github.com/GAP-LAB-CUHK-SZ/gaustudio, fetched 2026-10-26 (unofficial)
[S6] yulunwu0108/Perturbed-Gaussian-Ensemble, https://github.com/yulunwu0108/Perturbed-Gaussian-Ensemble, fetched 2026-10-26 (unofficial)
[S7] Lewis-Stuart-11/3DGS-to-PC, https://github.com/Lewis-Stuart-11/3DGS-to-PC, fetched 2026-10-27 (unofficial)
[S8] XiaoBin2001/Improved-GS, https://github.com/XiaoBin2001/Improved-GS, fetched 2026-10-27 (unofficial)
[S9] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S10] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
[S11] CoherentRaster: Efficient 3D Gaussian Splatting for Light Field Displays, https://doi.org/10.1145/3799902.3811217, fetched 2026-11-08
[S12] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
[S13] Explore: Gaussian splatting, OpenUSD, AI world generation, volumetric video — study, https://github.com/sceneview/sceneview/issues/4023, fetched 2026-10-27 (unofficial)
[S14] AdR-Gaussian: Accelerating Gaussian Splatting with Adaptive Radius, https://doi.org/10.1145/3680528.3687675, fetched 2026-10-26
[S15] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
[S16] GGRt: Towards Pose-Free Generalizable 3D Gaussian Splatting in Real-Time, https://doi.org/10.1007/978-3-031-73209-6_19, fetched 2026-10-29
[S17] From Splats to Silicon: Rethinking Computational Efficiency of 3DGS, https://arxiv.org/abs/2609.06157, fetched 2026-10-29 (unofficial)
[S18] FlashGS: Efficient 3D Gaussian Splatting for Large-scale and High-resolution Rendering, https://doi.org/10.1109/cvpr52734.2025.02482, fetched 2026-10-26
