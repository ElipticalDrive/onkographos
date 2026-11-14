# GPU Backprojection

Also called: FDK Backprojection CUDA Kernel, Scale field rendering, Scale-Space Backprojection, Weighted blob blending

## Overview
GPU Backprojection is a rendering pass that visualizes detected scale-space blobs by generating small slices for each blob, computing weighted contributions, and blending them to form the final image, followed by normalization [S2][S3, unofficial]. It is distinct from standard ray-casting as it operates on a sparse abstraction of the volume (scale maxima) rather than the full voxel grid, decoupling the expensive scale-space computation from the frequent backprojection step [S3, unofficial]. It builds on [Backward Projection](1491-backward-projection.md) [G].

## How it works
**Inputs:**
*   `projections`: A tensor of shape `[batch, views, rows, columns]` containing the projection data or scale-space blob parameters [S1, unofficial].
*   `matrices`: A tensor of shape `[batch, views, 3, 4]` containing the projection matrices for each view [S1, unofficial].
*   `volume_shape`: A tuple `(depth, height, width)` defining the output volume dimensions [S1, unofficial].
*   Data is stored as contiguous 32-bit floating point arrays [S4, unofficial].

**Outputs:**
*   `out`: A tensor of shape `[batch, 1, depth, height, width]` containing the backprojected volume [S1, unofficial].

**Data Structures:**
*   `PanelStencil`: A structure used in the CUDA kernel to store the four corners of the projected pixel footprint and their corresponding weights [S1, unofficial].
    *   `index[4]`: Unsigned integers representing the linear indices of the four pixels in the projection plane [S1, unofficial].
    *   `weight[4]`: Floats representing the interpolated weights for each pixel [S1, unofficial].
*   `ViewColumn`: A structure storing precomputed linear coefficients for a voxel's position relative to a view matrix [S1, unofficial].
    *   `column`, `column_z`: Linear coefficients for the column coordinate [S1, unofficial].
    *   `row`, `row_z`: Linear coefficients for the row coordinate [S1, unofficial].
    *   `depth`, `depth_z`: Linear coefficients for the depth (panel distance) coordinate [S1, unofficial].

**Steps:**
1.  **Preprocessing:** Detach and convert input matrices to contiguous float tensors: `matrices = matrices.detach().float().contiguous()` [S1, unofficial]. Reshape projections to a flat tensor: `flat = projections.reshape(batch, -1)` [S1, unofficial]. Extract dimensions: `depth, height, width` from `volume_shape`; `views, rows, columns` from `projections.shape[2:]` [S1, unofficial]. Create output tensor `out` of shape `[batch, 1, *volume_shape]` [S1, unofficial].
2.  **Kernel Execution (Per Batch Group):** Iterate over batches in groups defined by `_GROUP` [S1, unofficial]. For each group, load the specific kernel function `f"fdk_backproject_c{group}"` [S1, unofficial]. Execute the kernel with grid `_volume_grid(volume_shape)` and block `_VOLUME_BLOCK` [S1, unofficial]. Arguments passed to the kernel include: `flat`, `start`, `count`, `group`, `matrices`, `out[start]`, `width`, `height`, `depth`, `views`, `rows`, `columns`, `count`, and `depth * height * width` [S1, unofficial].
3.  **Kernel Logic (Per Voxel):** Each thread processes a subset of voxels (e.g., `TT_FDK_DEPTH` voxels along z) [S1, unofficial]. For each voxel $(x, y, z)$ and each view: compute `ViewColumn` using the projection matrix and voxel $(x, y)$ [S1, unofficial]. Compute the panel distance $w = \text{fmaf}(v.\text{depth\_z}, z, v.\text{depth})$ [S1, unofficial]. If $w \le 0$, skip the view (voxel is behind the panel) [S1, unofficial]. Compute inverse depth $inverse = 1/w$ [S1, unofficial]. Compute normalized coordinates: $column = \text{fmaf}(v.\text{column\_z}, z, v.\text{column}) \times inverse$ and $row = \text{fmaf}(v.\text{row\_z}, z, v.\text{row}) \times inverse$ [S1, unofficial]. Check if $(column, row)$ is within the projection bounds $column > -1$, $column < \text{columns}$, $row > -1$, $row < \text{rows}$ [S1, unofficial]. If within bounds, compute the `PanelStencil`: determine floor coordinates $c0 = \lfloor column \rfloor$, $r0 = \lfloor row \rfloor$ [S1, unofficial]. Compute deltas $dc = column - c0$, $dr = row - r0$ [S1, unofficial]. Compute weights using bilinear interpolation factors scaled by $inverse^2$: $wc0 = (c0 \ge 0) ? (1 - dc) : 0$, $wc1 = (c0 + 1 < \text{columns}) ? dc : 0$, $wr0 = (r0 \ge 0) ? (1 - dr) \times inverse^2 : 0$, $wr1 = (r0 + 1 < \text{rows}) ? dr \times inverse^2 : 0$ [S1, unofficial]. Store indices and weights in `PanelStencil` [S1, unofficial]. Scatter the weighted contribution to the output volume. The weight applied is proportional to $(D/l)^2$, where $l$ is the voxel depth along the panel normal and $D$ is the panel depth, implemented as $inverse^2$ [S1, unofficial].
4.  **Normalization:** In a separate pass, normalize the accumulated result by dividing by the sum of weights, which is stored in a separate channel [S2][S3, unofficial].

**Formulas:**
*   $w = \text{fmaf}(v.\text{depth\_z}, z, v.\text{depth})$ [S1, unofficial]
*   $inverse = 1/w$ [S1, unofficial]
*   $column = \text{fmaf}(v.\text{column\_z}, z, v.\text{column}) \times inverse$ [S1, unofficial]
*   $row = \text{fmaf}(v.\text{row\_z}, z, v.\text{row}) \times inverse$ [S1, unofficial]
*   $wc0 = (c0 \ge 0) ? (1 - dc) : 0$ [S1, unofficial]
*   $wc1 = (c0 + 1 < \text{columns}) ? dc : 0$ [S1, unofficial]
*   $wr0 = (r0 \ge 0) ? (1 - dr) \times inverse^2 : 0$ [S1, unofficial]
*   $wr1 = (r0 + 1 < \text{rows}) ? dr \times inverse^2 : 0$ [S1, unofficial]
*   Weight factor: $(D/l)^2$ [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def gpu_backprojection(projections: torch.Tensor, matrices: torch.Tensor, volume_shape: tuple) -> torch.Tensor:
    """
    Performs GPU backprojection of scale-space blobs.
    
    Args:
        projections: Tensor of shape [batch, views, rows, columns]
        matrices: Tensor of shape [batch, views, 3, 4]
        volume_shape: Tuple (depth, height, width)
        
    Returns:
        Tensor of shape [batch, 1, depth, height, width]
    """
    batch, views, rows, columns = projections.shape
    depth, height, width = volume_shape
    
    # Preprocessing
    matrices = matrices.detach().float().contiguous()
    flat = projections.reshape(batch, -1)
    out = torch.zeros((batch, 1, depth, height, width), dtype=torch.float32)
    
    GROUP = 32  # Example group size
    for start in range(0, batch, GROUP):
        count = min(GROUP, batch - start)
        group_idx = start // GROUP
        
        # CUDA kernel launch:
        # grid = _volume_grid(volume_shape)
        # block = _VOLUME_BLOCK
        # args = [flat, start, count, group_idx, matrices, out[start],
        #         width, height, depth, views, rows, columns, count, depth*height*width]
        # kernel = _kernels().get(f"fdk_backproject_c{group_idx}")
        # kernel(grid, block, args)
        
        # Normalization pass (separate kernel or operation)
        # out[start] = out[start] / weight_sum_channel[start]

    return out
```

**Parameters:**
*   `GROUP`: The batch group size for kernel execution. Not given in the sources.
*   `TT_FDK_DEPTH`: The number of voxels processed along the z-axis per thread. Set to 4 [S1, unofficial].
*   `_VOLUME_BLOCK`: The CUDA block size. Not given in the sources.
*   `_volume_grid`: The CUDA grid configuration. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Zero/Behind Panel:** If $w \le 0$, the voxel is behind the projection panel and should be skipped to avoid division by zero or invalid weights [S1, unofficial].
*   **Boundaries:** The stencil code checks for bounds $column > -1$, $column < \text{columns}$, $row > -1$, $row < \text{rows}$, which implies a coordinate system where the projection plane extends slightly beyond the integer pixel grid [S1, unofficial].
*   **Normalization Channel:** The sum of weights must be accumulated in a separate output channel during the backprojection pass to enable the subsequent normalization division [S2][S3, unofficial].

## Subtopics

### CUDA Backprojection Implementation
Parallelization is performed over the reconstruction voxels in the output space using 3D threads [S4, unofficial]. The implementation utilizes 3D texture memory for the input data (CT projections for backprojection) [S4, unofficial]. CT projection data and reconstruction volumes are stored as contiguous 32-bit floating point arrays, with users providing pointers to these arrays [S4, unofficial]. If input data is not on the GPU, the software copies data to the GPU, performs the calculation, and copies the result back to the CPU [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The backprojection step is significantly faster than the scale-space computation [S3, unofficial]. The scale-space computation is done once, while backprojection can be done frequently [S3, unofficial].
*   **Memory Complexity:** Requires temporary volume storage for the scale field [S3, unofficial]. The output volume size is $O(N^3)$ where $N$ is the volume dimension [S3, unofficial].
*   **Performance:** On an Intel Core 2 Duo 2.4 GHz with 2GB RAM and an nVidia GeForce 8800 GTX with 768 MB texture memory, the GPU implementation showed an improvement of about two orders of magnitude over the CPU version for data set sizes of $64^3$, $128^3$, and $256^3$ [S2][S3, unofficial].
*   **Strengths:** Decouples the expensive scale-space computation from the backprojection, allowing for interactive visualization where the user can change smoothness and visibility parameters frequently [S3, unofficial].
*   **Limitations:** Relies on a temporary volume to compute the scale field. Real-time implementation may be limited by GPU memory as the volume size increases [S3, unofficial].

## Background
GPU Backprojection is part of the "Size-based Transfer Functions" technique for volume exploration [S2][S3, unofficial]. It was developed to address the computational cost of interactive volume visualization by leveraging the GPU for the backprojection step [S2][S3, unofficial]. The method was evaluated on an nVidia GeForce 8800 GTX [S2][S3, unofficial].

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S2] PDF Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
[S3] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
[S4] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
