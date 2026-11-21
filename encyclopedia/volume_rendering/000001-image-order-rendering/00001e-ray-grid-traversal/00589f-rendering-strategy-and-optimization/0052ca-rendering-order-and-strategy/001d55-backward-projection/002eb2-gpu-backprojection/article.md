# GPU Backprojection

Also called: FDK Backprojection CUDA Kernel, Scale field rendering, Scale-Space Backprojection, Weighted blob blending

## Overview
GPU Backprojection is a rendering pass that visualizes detected scale-space blobs by mapping projection data back into a 3D reconstruction volume [S1, unofficial][S2][S3, unofficial]. It solves the problem of efficiently computing weighted contributions of scale-space features to image pixels by parallelizing the accumulation over the output space (voxels or pixels) on NVIDIA GPUs [S1, unofficial][S2][S3, unofficial]. It is distinguished from its siblings by its specific application to scale-space blob visualization [S3, unofficial] and its implementation as a CUDA kernel that decouples the expensive scale-space computation from the frequent backprojection step [S3, unofficial], unlike [Ray Segment Projection to Image](7631-ray-segment-projection-to-image.md) which projects geometric primitives, or [Cone Beam Forward Projection with Trilinear Interpolation](8247-cone-beam-forward-projection-with-trilinear-interpolation.md) which performs the forward mapping.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `projections`: A tensor of shape `[batch, views, rows, columns]` containing the projection data [S1, unofficial][S2][S3, unofficial].
    *   `matrices`: A contiguous float tensor containing the view transformation matrices [S1, unofficial].
    *   `volume_shape`: A tuple `(depth, height, width)` defining the 3D reconstruction volume dimensions [S1, S3, unofficial].
    *   `blob_data`: Information about detected scale-space blobs, including their radii and positions, used to generate slices [S2][S3, unofficial].
*   **Outputs:**
    *   `out`: A tensor of shape `[batch, 1, depth, height, width]` containing the backprojected volume [S1, S3, unofficial].

**Data Structures:**
*   **PanelStencil:** A structure used in the CUDA kernel to store the 4 pixels (indices) and their corresponding weights for a voxel's projection onto the 2D panel [S1, unofficial].
    *   `index[4]`: Unsigned integers representing the pixel indices in the flattened projection array.
    *   `weight[4]`: Floats representing the interpolated weights for each pixel.
*   **ViewColumn:** A structure storing linear coefficients for projecting a voxel's x, y, z coordinates to the 2D panel [S1, unofficial].
    *   `column, column_z`: Linear coefficients for the column coordinate.
    *   `row, row_z`: Linear coefficients for the row coordinate.
    *   `depth, depth_z`: Linear coefficients for the depth (distance along panel normal).

**Steps:**
1.  **Preprocessing:**
    *   Detach and convert `matrices` to a contiguous float tensor [S1, unofficial].
    *   Reshape `projections` to a flat tensor `flat` of shape `[batch, -1]` [S1, unofficial].
    *   Create the output tensor `out` with shape `[batch, 1, *volume_shape]` [S1, S3, unofficial].
    *   Load the kernel library containing the FDK backprojection kernels [S1, unofficial].

2.  **Batch Iteration:**
    *   Iterate over batches in groups of size `_GROUP` [S1, unofficial].
    *   For each group, select the appropriate kernel function (e.g., `fdk_backproject_c{group}`) [S1, unofficial].

3.  **Kernel Execution (Per Voxel):**
    *   Launch a CUDA kernel with grid `_volume_grid(volume_shape)` and block `_VOLUME_BLOCK` [S1, unofficial].
    *   Each thread processes a voxel at `(x, y, z)` in the volume [S1, unofficial].
    *   **View Projection:** Compute the `ViewColumn` for the voxel using the transformation matrix [S1, unofficial]:
        $$
        \begin{aligned}
        c &= M_{0,0}x + M_{0,1}y + M_{0,2}z + M_{0,3} \\
        r &= M_{1,0}x + M_{1,1}y + M_{1,2}z + M_{1,3} \\
        d &= M_{2,0}x + M_{2,1}y + M_{2,2}z + M_{2,3}
        \end{aligned}
        $$
        Where $M$ is the view matrix [S1, unofficial].
    *   **Panel Stencil:** Calculate the 2D panel coordinates and weights [S1, unofficial]:
        *   Compute depth $w = d_z z + d$. If $w \le 0$, the voxel is behind the panel; skip [S1, unofficial].
        *   Compute inverse $1/w$ [S1, unofficial].
        *   Compute column $u = (c_z z + c) / w$ and row $v = (r_z z + r) / w$ [S1, unofficial].
        *   Determine the bounding pixel indices $(c_0, r_0)$ and fractional parts $dc, dr$ [S1, unofficial].
        *   Compute weights using bilinear interpolation factors scaled by $1/w^2$ [S1, unofficial]:
            $$
            \begin{aligned}
            w_{00} &= \max(c_0, 0) \ge 0 ? (1-dc) : 0 \\
            w_{10} &= c_0+1 < \text{cols} ? dc : 0 \\
            w_{01} &= \max(r_0, 0) \ge 0 ? (1-dr) \cdot (1/w)^2 : 0 \\
            w_{11} &= r_0+1 < \text{rows} ? dr \cdot (1/w)^2 : 0
            \end{aligned}
            $$
            *Note: The source code applies the $1/w^2$ scaling to the row weights specifically in the provided snippet, but the formula section states the output is weighted by $(D/l)^2$ where $l$ is voxel depth [S1, S3, unofficial]. The stencil weights combine spatial interpolation and depth weighting [S1, unofficial].*
        *   Store the 4 pixel indices and weights in a `PanelStencil` [S1, unofficial].

4.  **Accumulation (Scale-Space Specific):**
    *   For each detected scale-space blob, generate a series of small slices with dimensions proportional to the blob's radius [S2][S3, unofficial].
    *   Compute the weighted contribution of the blob for each pixel covered by its slices [S2][S3, unofficial].
    *   Use alpha blending to accumulate the sum of weighted contributions from all blobs into the output volume [S1, S3, unofficial].

5.  **Normalization:**
    *   In a separate pass, normalize the accumulated result by dividing by the sum of weights, which is stored in a separate channel [S2][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_backprojection(projections, matrices, volume_shape, blob_data):
    """
    Performs GPU backprojection for scale-space blob visualization.
    
    Args:
        projections: Tensor of shape [batch, views, rows, columns]
        matrices: Contiguous float tensor of view matrices
        volume_shape: Tuple (depth, height, width)
        blob_data: List of blobs with radius and position
        
    Returns:
        out: Tensor of shape [batch, 1, depth, height, width]
    """
    batch_size = projections.shape[0]
    views, rows, columns = projections.shape[2:]
    depth, height, width = volume_shape
    
    # Preprocessing
    flat_projections = projections.reshape(batch_size, -1)
    matrices = matrices.detach().float().contiguous()
    out = torch.zeros((batch_size, 1, depth, height, width))
    
    # Load kernel library (pseudo-code for external CUDA kernel)
    library = load_kernel_library()
    
    # Iterate over batches in groups
    for start, count, group in group_batches(batch_size, GROUP_SIZE):
        kernel_name = f"fdk_backproject_c{group}"
        kernel = library[kernel_name]
        
        # Pack arguments for CUDA kernel
        args = pack_args(
            flat_projections, start, count, group,
            matrices, out[start],
            width, height, depth, views, rows, columns
        )
        
        # Execute kernel
        # Grid and block dimensions are derived from volume_shape
        grid = compute_volume_grid(volume_shape)
        block = VOLUME_BLOCK
        kernel(grid, block, args)
        
    # Normalization pass
    # Normalize by dividing by the sum of weights stored in a separate channel
    out = normalize_backprojection(out)
    
    return out

def compute_panel_stencil(view_matrix, x, y, z, rows, columns):
    """
    Computes the 4-pixel stencil and weights for a voxel.
    """
    # Extract matrix rows
    c = view_matrix[0]
    r = view_matrix[1]
    d = view_matrix[2]
    
    # Linear projection
    col = fma(c.x, x, fma(c.y, y, c.w))
    row = fma(r.x, x, fma(r.y, y, r.w))
    depth = fma(d.x, x, fma(d.y, y, d.w))
    
    # Depth coefficients
    col_z = c.z
    row_z = r.z
    depth_z = d.z
    
    # Project z
    w = fma(depth_z, z, depth)
    if w <= 0:
        return None # Behind panel
    
    inverse = 1.0 / w
    col_proj = fma(col_z, z, col) * inverse
    row_proj = fma(row_z, z, row) * inverse
    
    # Bounding box
    col_floor = floor(col_proj)
    row_floor = floor(row_proj)
    c0 = int(col_floor)
    r0 = int(row_floor)
    dc = col_proj - col_floor
    dr = row_proj - row_floor
    
    # Weights
    # Note: Source applies 1/w^2 to row weights in stencil calculation
    scale = inverse * inverse
    wc0 = (1.0 - dc) if c0 >= 0 else 0.0
    wc1 = dc if c0 + 1 < columns else 0.0
    wr0 = (1.0 - dr) * scale if r0 >= 0 else 0.0
    wr1 = dr * scale if r0 + 1 < rows else 0.0
    
    # Indices
    left = max(c0, 0)
    right = min(c0 + 1, columns - 1)
    top = max(r0, 0) * columns
    bottom = min(r0 + 1, rows - 1) * columns
    
    stencil = {
        'index': [
            top + left, top + right,
            bottom + left, bottom + right
        ],
        'weight': [
            wr0 * wc0, wr0 * wc1,
            wr1 * wc0, wr1 * wc1
        ]
    }
    return stencil
```

**Parameters:**
*   `GROUP_SIZE`: The number of batches processed together by a single kernel launch. Not given in the sources.
*   `TT_FDK_DEPTH`: Number of voxels along z processed per thread. Set to 4 in the source [S1, unofficial].
*   `VOLUME_BLOCK`: CUDA block dimension. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Behind Panel:** If the projected depth $w \le 0$, the voxel is behind the projection panel and should be skipped [S1, unofficial].
*   **Boundary Checks:** Pixel indices must be clamped to the valid range $[0, \text{rows}-1]$ and $[0, \text{columns}-1]$ to avoid out-of-bounds memory access [S1, unofficial].
*   **Precision:** The use of `float32` for addition order may cause slight differences between forward and backprojection due to non-associativity [S1, unofficial].
*   **Scale-Space Decoupling:** The scale-space computation is expensive and done once, while backprojection is faster and done frequently. Mixing these steps incorrectly can lead to performance degradation [S3, unofficial].

## Subtopics

### [CUDA Backprojection Implementation](9790-cuda-backprojection-implementation.md)
Implements the backprojection step of differentiable X-ray computed tomography on NVIDIA GPUs using CUDA. It is distinguished by its use of 3D thread blocks and 3D texture memory for input data access, and support for both Siddon and SF projector models. It builds on [GPU Backprojection](5843-gpu-backprojection.md).

## Operating characteristics
*   **Time Complexity:** The backprojection step is $O(N \cdot V)$, where $N$ is the number of views and $V$ is the number of voxels, but optimized by the stencil approach which limits writes to 4 pixels per voxel [S1, unofficial]. The scale-space computation is $O(S)$, where $S$ is the number of scale-space iterations, but is decoupled and done once [S3, unofficial].
*   **Memory Complexity:** Requires storage for the projection data, the volume, and temporary buffers for the stencil and weights. The source mentions using 768 MB of texture memory for a 256^3 volume [S3, unofficial].
*   **Measured Performance:** On an Intel Core 2 Duo 2.4 GHz with 2GB RAM and an nVidia GeForce 8800 GTX with 768 MB texture memory, the GPU implementation showed an improvement of about two orders of magnitude over the CPU version for data set sizes of 64^3, 128^3, and 256^3 [S3, unofficial].
*   **Strengths:** Interactive visualization is possible due to the speedup over CPU implementations. The decoupling of scale-space computation allows for frequent updates of smoothness and visibility parameters [S3, unofficial].
*   **Limitations:** Relies on a temporary volume to compute the scale field, which may limit real-time embedding in the volume rendering process as GPUs improve [S3, unofficial].
*   **When to Use:** When interactive visualization of scale-space blobs is required, and the scale-space computation can be decoupled from the backprojection [S3, unofficial].
*   **When Not to Use:** When the scale-space computation needs to be updated frequently with every parameter change, or when the hardware does not support the required CUDA features [S3, unofficial].

## Background
GPU Backprojection was introduced in the context of Size-based Transfer Functions for volume exploration [S2][S3, unofficial]. The method was developed to address the computational cost of visualizing scale-space blobs, which are detected maxima in a scale-space representation of the volume data [S2][S3, unofficial]. The GPU implementation was evaluated on an nVidia GeForce 8800 GTX, demonstrating the feasibility of interactive visualization for large volume datasets [S3, unofficial]. The approach decouples the expensive scale-space computation from the backprojection, allowing for efficient user interaction with smoothness and visibility parameters [S3, unofficial].

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S2] PDF Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
[S3] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
