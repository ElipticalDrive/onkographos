# Multi-view Mini-batch Rasterization

Also called: Multi-view mini-batch training for 3D Gaussian Splatting

## Overview
Multi-view mini-batch rasterization is a training strategy that renders multiple viewpoints simultaneously within a single mini-batch to reduce gradient variance and improve convergence speed [S1, unofficial]. It modifies the standard 3D Gaussian Splatting (3DGS) rasterizer pipeline by introducing view-specific frustum culling and a composite sorting ID, allowing the rasterizer to process multiple camera views in parallel while significantly reducing memory consumption compared to naive multi-view rendering [S1, unofficial]. Unlike single-view training, this method increases performance metrics (PSNR/SSIM) but introduces higher memory overhead and preprocessing time [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $P$ 3D Gaussians; a mini-batch of $B$ viewpoints (camera poses and intrinsics); tile resolution and screen dimensions.
*   **Outputs:** Rendered images for all $B$ viewpoints; gradients for Gaussian parameters (position, scale, rotation, opacity, color) propagated back from the loss function.

**Data Structures**
The core data structure is an array of projected Gaussian attributes. To support multi-view processing, the sorting identifier is modified from the standard 3DGS implementation:
*   **Standard 3DGS ID:** 64-bit integer composed of a 32-bit tile index and a 32-bit depth value [S1, unofficial].
*   **Multi-view ID:** 64-bit integer composed of:
    *   16-bit tile index
    *   16-bit viewpoint index (identifying which of the $B$ viewpoints the Gaussian belongs to)
    *   32-bit depth value [S1, unofficial]

**Numbered Steps**

1.  **Frustum Culling (Per Viewpoint):**
    For each of the $B$ viewpoints in the mini-batch, determine which of the $P$ total Gaussians are visible within the camera frustum. This step identifies the subset of Gaussians participating in rendering for each specific viewpoint [S1, unofficial].

2.  **Memory Allocation:**
    Allocate memory only for the Gaussians identified in Step 1. Let $N$ be the total number of Gaussian instances remaining after culling across all viewpoints. Typically, $N \ll B \times P$ because most Gaussians are not visible from every viewpoint [S1, unofficial].

3.  **Projection:**
    Compute the projected 2D Gaussian parameters (mean, covariance) for the participating Gaussians in screen space for their respective viewpoints [S1, unofficial]. Note that a single 3D Gaussian may be projected and appear multiple times in the array if it is visible in multiple viewpoints [S1, unofficial].

4.  **Duplication and ID Assignment:**
    Duplicate projected Gaussians for each tile they cover. Assign each duplicated Gaussian a unique 64-bit ID using the multi-view format (16-bit tile, 16-bit viewpoint, 32-bit depth). This ensures that projected Gaussians are separated by both tile and viewpoint during sorting [S1, unofficial].

5.  **Sorting:**
    Sort the array of projected Gaussians by their 64-bit ID. This groups Gaussians by tile and viewpoint, ensuring that rendering blocks for different viewpoints do not interfere with each other [S1, unofficial].

6.  **Rendering:**
    The rasterizer allocates multiple rendering blocks per tile—one block for each viewpoint. Each block renders its designated pixels using only the projected Gaussians corresponding to its assigned viewpoint index [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_view_rasterize(gaussians_3d, viewpoints, tile_size, screen_width, screen_height):
    """
    gaussians_3d: List of P 3D Gaussian objects (pos, cov, opacity, color)
    viewpoints: List of B camera objects (view_matrix, proj_matrix, intrinsics)
    tile_size: Integer, size of a tile in pixels (e.g., 16)
    screen_width, screen_height: Integer, dimensions of the image
    """
    all_projected_gaussians = []
    
    # Step 1 & 2: Frustum Culling and Memory Allocation
    # We collect only visible Gaussians for each viewpoint
    for viewpoint_idx, cam in enumerate(viewpoints):
        visible_gaussians = []
        for g in gaussians_3d:
            if is_in_frustum(g, cam):
                visible_gaussians.append(g)
        
        # Step 3: Projection
        for g in visible_gaussians:
            # Project 3D Gaussian to 2D screen space
            # Returns projected mean, 2D covariance, depth, and original 3D index
            proj_g = project_gaussian_2d(g, cam)
            
            # Determine which tiles this Gaussian covers
            tiles_covered = get_tiles_covered(proj_g, tile_size, screen_width, screen_height)
            
            # Step 4: Duplication and ID Assignment
            for tile_idx in tiles_covered:
                # Construct 64-bit ID: 16-bit tile | 16-bit viewpoint | 32-bit depth
                # Assuming depth is normalized or scaled to fit 32 bits
                id_tile = tile_idx & 0xFFFF
                id_view = viewpoint_idx & 0xFFFF
                id_depth = int(proj_g.depth * 1e6) & 0xFFFFFFFF # Placeholder for depth packing
                
                composite_id = (id_tile << 48) | (id_view << 32) | id_depth
                
                # Store projected Gaussian with its composite ID
                all_projected_gaussians.append({
                    'proj_g': proj_g,
                    'composite_id': composite_id,
                    'viewpoint_idx': viewpoint_idx,
                    'tile_idx': tile_idx
                })

    # Step 5: Sorting
    # Sort by composite ID. This implicitly sorts by tile, then viewpoint, then depth.
    all_projected_gaussians.sort(key=lambda x: x['composite_id'])

    # Step 6: Rendering
    # Initialize output images for all viewpoints
    images = [np.zeros((screen_height, screen_width, 3)) for _ in viewpoints]
    
    # Iterate through sorted Gaussians and accumulate into tiles
    # The rasterizer processes tiles. For each tile, it processes each viewpoint separately.
    # In a GPU implementation, this is handled by thread blocks.
    
    # Pseudocode for CPU-like accumulation for clarity
    for tile_x in range(0, screen_width, tile_size):
        for tile_y in range(0, screen_height, tile_size):
            tile_idx = (tile_y // tile_size) * (screen_width // tile_size) + (tile_x // tile_size)
            
            # Get all Gaussians for this tile across all viewpoints
            # Since sorted by ID, these are contiguous in the list
            tile_gaussians = [g for g in all_projected_gaussians if g['tile_idx'] == tile_idx]
            
            for viewpoint_idx in range(len(viewpoints)):
                # Filter for specific viewpoint
                view_gaussians = [g for g in tile_gaussians if g['viewpoint_idx'] == viewpoint_idx]
                
                # Sort by depth for alpha compositing
                view_gaussians.sort(key=lambda x: x['proj_g'].depth)
                
                # Render this tile for this viewpoint
                render_tile(images[viewpoint_idx], tile_x, tile_y, tile_size, view_gaussians)

    return images

def is_in_frustum(gaussian, camera):
    # [G] Placeholder for actual frustum culling logic
    # Check if the bounding sphere of the Gaussian intersects the camera frustum
    return True

def project_gaussian_2d(gaussian, camera):
    # [G] Placeholder for 3DGS projection math
    # Returns object with .mean, .cov, .depth
    return {'mean': [0,0], 'cov': [[1,0],[0,1]], 'depth': 1.0}

def get_tiles_covered(proj_g, tile_size, width, height):
    # [G] Placeholder for tile coverage calculation
    return [0]

def render_tile(image, tile_x, tile_y, tile_size, gaussians):
    # [G] Placeholder for alpha compositing
    pass
```

**Parameters**
*   **Pruning Threshold:** In standard 3DGS, the threshold for pruning Gaussians is multiplied by the number of images ($B$) in the mini-batch to account for changed learning patterns [S1, unofficial].
*   **Mean Noise Learning Rate:** In 3DGS-MCMC, this rate is divided by the number of images ($B$) when applying multi-view training [S1, unofficial].
*   **Number of Viewpoints ($B$):** The number of viewpoints rendered per mini-batch. Increasing $B$ improves performance but increases memory and time overhead [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Overfitting:** Multi-view mini-batch training can cause overfitting on specific scenes, notably the bicycle/stump scenes in the MipNeRF-360 dataset and scenes in the DeepBlending dataset [S1, unofficial].
*   **Preprocessing Overhead:** While rendering overhead is mitigated, the preprocessing steps (culling, sorting, ID assignment) remain. This prevents the method from achieving constant time complexity with respect to the number of images, unlike NeRF-based methods [S1, unofficial].
*   **Gradient Nullification:** If viewpoints are positioned symmetrically (e.g., opposite each other facing the origin), gradients from different viewpoints can cancel each other out for positional updates if not handled correctly [S1, unofficial].

## Subtopics

### Gaussian Splat Rasterization (Vertex Shader)
Uses vertex shaders for projection, differing in hardware acceleration approach. [Gaussian Splat Rasterization (Vertex Shader)](3253-gaussian-splat-rasterization-vertex-shader.md)

### Masked-Rasterization
Optimizes memory by masking out empty tiles, differing in how it handles sparsity. [Masked-Rasterization](4366-masked-rasterization.md)

### Differentiable Radiative Rasterization
Focuses on radiative transfer, differing in the physical model used. [Differentiable Radiative Rasterization](4526-differentiable-radiative-rasterization.md)

### 3D/2D Hybrid Gaussian Rasterization with Fallback Sigma
Combines 3D and 2D representations, differing in Gaussian representation. [3D/2D Hybrid Gaussian Rasterization with Fallback Sigma](4812-3d-2d-hybrid-gaussian-rasterization-with-fallback-sigma.md)

### Opacity Rendering with SH Colors
Handles opacity and spherical harmonics differently, differing in color representation. [Opacity Rendering with SH Colors](7480-opacity-rendering-with-sh-colors.md)

### 3D Gaussian Unscented Transform (3DGUT)
Uses unscented transform for uncertainty, differing in probabilistic modeling. [3D Gaussian Unscented Transform (3DGUT)](9154-3d-gaussian-unscented-transform-3dgut.md)

### RoofGS Quadratic Form Expansion and Coefficient Folding
Uses quadratic form expansion, differing in mathematical formulation. [RoofGS Quadratic Form Expansion and Coefficient Folding](10258-roofgs-quadratic-form-expansion-and-coefficient-folding.md)

### Transmittance of Gaussian Splatting
Focuses on transmittance calculation, differing in alpha compositing details. [Transmittance of Gaussian Splatting](10263-transmittance-of-gaussian-splatting.md)

### Rendering Pipeline and Configuration
General pipeline setup, differing in configuration scope. [Rendering Pipeline and Configuration](11406-rendering-pipeline-and-configuration.md)

### Differentiable Backpropagation Mechanisms
General backpropagation, differing in gradient flow specifics. [Differentiable Backpropagation Mechanisms](11407-differentiable-backpropagation-mechanisms.md)

## Operating characteristics
*   **Time Complexity:** Not constant with respect to the number of images due to preprocessing overhead (culling, sorting) [S1, unofficial].
*   **Memory Complexity:** Significantly reduced compared to naive multi-view rendering ($N \ll B \times P$), but still higher than single-view training [S1, unofficial].
*   **Performance:** On the MipNeRF-360 dataset, using 8 viewpoints in the mini-batch achieved a PSNR of 30.56, SSIM of 0.901, and LPIPS of 0.138 for 3DGS-MCMC + Ours, compared to 29.94 PSNR for the base 3DGS [S1, unofficial].
*   **Strengths:** Reduces gradient variance, improves convergence, and lowers memory usage compared to rendering all views independently.
*   **Limitations:** Preprocessing overhead remains; potential for overfitting on certain scenes; increased memory/time with more viewpoints [S1, unofficial].
*   **When to Use:** When training speed and convergence are prioritized, and hardware memory allows for multi-view processing.
*   **When Not to Use:** When constant time complexity with respect to image count is required, or when dealing with scenes prone to overfitting (e.g., bicycle/stump in MipNeRF-360) [S1, unofficial].

## Background
The method was introduced in "Efficient multi-view training for 3D Gaussian Splatting" [S1, unofficial]. It addresses the limitations of single-view training by leveraging multiple viewpoints per iteration to stabilize gradients. The approach builds on the 3DGS rasterizer [2813-3d-gaussian-splatting-differentiable-rasterizer.md] and modifies its preprocessing and sorting stages to handle multi-view data efficiently.

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
