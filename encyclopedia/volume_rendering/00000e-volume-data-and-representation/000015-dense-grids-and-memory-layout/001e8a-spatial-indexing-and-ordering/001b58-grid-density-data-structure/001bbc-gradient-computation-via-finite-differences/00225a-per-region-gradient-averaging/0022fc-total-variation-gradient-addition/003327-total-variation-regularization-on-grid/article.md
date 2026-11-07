# Total Variation Regularization on Grid

Also called: DVGO Density Total Variation, TV Regularization

## Overview
Total Variation (TV) regularization on a grid adds a gradient term to the density (or embedding) grid to encourage piecewise constant solutions, effectively reducing noise and enforcing smoothness in the learned volume representation [S1, S2, unofficial]. Unlike standard L2 regularization which penalizes large values, TV regularization penalizes the magnitude of gradients, preserving sharp boundaries while smoothing flat regions [S2, unofficial]. It is applied after gradient computation and per-region averaging, serving as the final step in the optimization update for hash-based or tiled grid encoders [S1, S2, unofficial].

## How it works
**Inputs:**
- `grid`: The dense grid tensor (hash grid or tiled grid) containing embeddings or density values. Shape: `[num_embeddings, C]` where `C` is the channel dimension (e.g., 3 for RGB density, or higher for multi-resolution features).
- `grad`: The accumulated gradient buffer for the grid, same shape as `grid`.
- `inputs`: The 3D coordinates of the points being optimized, shape `[B, 3]`.
- `resolution`: The spatial resolution of the grid, scalar or tuple `(res_x, res_y, res_z)`.
- `world_size`: The physical dimensions of the world bounding box, used for scaling.
- `weight`: Scalar weight for the TV loss.
- `align_corners`: Boolean flag indicating if grid corners align with coordinate boundaries.
- `hashmap_size`: Size of the hash map (for hash grids).
- `gridtype`: Type of grid ('hash' or 'tiled').
- `D`: Number of dimensions (typically 3).

**Outputs:**
- `grad`: Updated gradient buffer with TV regularization terms added.

**Data Structures:**
- `grid`: Float tensor.
- `grad`: Float tensor.
- `inputs`: Float tensor.

**Parameters:**
- `weight`: Typical values are small (e.g., $10^{-4}$ to $10^{-2}$), scaled by resolution. [S2, unofficial]
- `N_THREAD`: 512. [S2, unofficial]
- `epsilon`: $10^{-9}$ for numerical stability. [S2, unofficial]

**Steps:**

1.  **Scale Weight:** Calculate the effective weight by scaling the input `weight` by the maximum dimension of the world size divided by 128.
    $$ w_{scaled} = weight \times \frac{\max(world\_size)}{128} $$
    [S1, S2, unofficial]

2.  **Kernel Launch Configuration:**
    -   Set `N_THREAD = 512`.
    -   Calculate blocks:
        $$ blocks\_x = \lceil B / N\_THREAD \rceil $$
        $$ blocks\_y = L $$
        where $B$ is the batch size (number of points) and $L$ is the number of levels (for multi-resolution grids) or 1 for single resolution. [S2, unofficial]

3.  **Coordinate Calculation (Per Point):**
    For each point $i$ with input coordinates $inputs[i]$:
    -   Calculate continuous position:
        $$ pos[d] = inputs[i][d] \times scale + (align\_corners ? 0.0f : 0.5f) $$
        where $scale$ is derived from the world size and resolution. [S2, unofficial]
    -   Calculate integer grid position:
        $$ pos\_grid[d] = \lfloor pos[d] \rfloor $$
    [S2, unofficial]

4.  **Hash Index Calculation:**
    -   Calculate the base hash index for the current grid cell using the grid's indexing function:
        $$ index = get\_grid\_index(gridtype, align\_corners, 0, hashmap\_size, resolution, pos\_grid) $$
    [S2, unofficial]

5.  **Gradient Accumulation Loop:**
    Initialize `results` (sum of differences) and `idelta` (sum of squared differences) for each channel $ch$ to 0. [S2, unofficial]
    For each dimension $d \in \{0, 1, 2\}$:
    -   **Right Neighbor:**
        -   If $pos\_grid[d] < resolution[d]$:
            -   Increment $pos\_grid[d]$ by 1.
            -   Calculate $index\_right$ using the updated $pos\_grid$.
            -   For each channel $ch$:
                $$ grad\_val = grid[index + ch] - grid[index\_right + ch] $$
                $$ results[ch] += grad\_val $$
                $$ idelta[ch] += grad\_val^2 $$
    -   **Left Neighbor:**
        -   Decrement $pos\_grid[d]$ by 1 (restoring original position).
        -   If $pos\_grid[d] > 0$:
            -   Calculate $index\_left$ using the updated $pos\_grid$.
            -   For each channel $ch$:
                $$ grad\_val = grid[index + ch] - grid[index\_left + ch] $$
                $$ results[ch] += grad\_val $$
                $$ idelta[ch] += grad\_val^2 $$
    [S2, unofficial]

6.  **Weight Calculation:**
    $$ w = \frac{w_{scaled}}{2 \times D} $$
    [S2, unofficial]

7.  **Atomic Write:**
    For each channel $ch$:
    $$ grad[index + ch] \leftarrow grad[index + ch] + w \cdot results[ch] \cdot \frac{1}{\sqrt{idelta[ch] + 10^{-9}}} $$
    [S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def total_variation_regularization_on_grid(
    grid, 
    grad, 
    inputs, 
    resolution, 
    world_size, 
    weight, 
    align_corners, 
    hashmap_size, 
    gridtype, 
    D=3
):
    """
    Adds total variation regularization gradient to the grid.
    
    Args:
        grid: Tensor [num_embeddings, C]
        grad: Tensor [num_embeddings, C] (accumulated gradients)
        inputs: Tensor [B, 3] (point coordinates)
        resolution: int or tuple (grid resolution)
        world_size: float or tuple (world dimensions)
        weight: float (TV weight)
        align_corners: bool
        hashmap_size: int (for hash grids)
        gridtype: str ('hash' or 'tiled')
        D: int (dimensions, default 3)
        
    Returns:
        grad: Updated gradient tensor
    """
    # Scale weight by max world dimension / 128
    max_world_dim = max(world_size) if isinstance(world_size, (list, tuple)) else world_size
    w_scaled = weight * (max_world_dim / 128.0)
    
    B = inputs.shape[0]
    C = grid.shape[1]
    N_THREAD = 512
    
    # Calculate blocks
    blocks_x = math.ceil(B / N_THREAD)
    blocks_y = 1 # Assuming single resolution level for simplicity; L for multi-level
    
    # Process each point
    for i in range(B):
        # Coordinate Calculation
        pos = [0.0, 0.0, 0.0]
        for d in range(D):
            scale = resolution[d] / max_world_dim # Approximate scaling
            pos[d] = inputs[i][d] * scale + (0.0 if align_corners else 0.5)
            
        pos_grid = [int(math.floor(p)) for p in pos]
        
        # Hash Index Calculation
        # Note: get_grid_index is a helper that maps 3D coords to hash index
        index = get_grid_index(gridtype, align_corners, 0, hashmap_size, resolution, pos_grid)
        
        # Initialize accumulation buffers
        results = [0.0] * C
        idelta = [0.0] * C
        
        # Gradient Accumulation Loop
        for d in range(D):
            # Right Neighbor
            if pos_grid[d] < resolution[d]:
                pos_grid[d] += 1
                index_right = get_grid_index(gridtype, align_corners, 0, hashmap_size, resolution, pos_grid)
                for ch in range(C):
                    grad_val = grid[index][ch] - grid[index_right][ch]
                    results[ch] += grad_val
                    idelta[ch] += grad_val * grad_val
                pos_grid[d] -= 1 # Restore
            
            # Left Neighbor
            if pos_grid[d] > 0:
                pos_grid[d] -= 1
                index_left = get_grid_index(gridtype, align_corners, 0, hashmap_size, resolution, pos_grid)
                for ch in range(C):
                    grad_val = grid[index][ch] - grid[index_left][ch]
                    results[ch] += grad_val
                    idelta[ch] += grad_val * grad_val
                pos_grid[d] += 1 # Restore
                
        # Weight Calculation
        w = w_scaled / (2.0 * D)
        
        # Atomic Write (simulated with direct assignment for CPU pseudocode)
        for ch in range(C):
            norm_factor = 1.0 / math.sqrt(idelta[ch] + 1e-9)
            grad[index][ch] += w * results[ch] * norm_factor
            
    return grad

def get_grid_index(gridtype, align_corners, level, hashmap_size, resolution, pos_grid):
    """
    Placeholder for grid-specific indexing logic.
    For hash grids, this involves hashing the 3D coordinates.
    For tiled grids, this involves linear indexing.
    """
    if gridtype == 'hash':
        # Simplified hash function for pseudocode
        # In reality, this uses a specific hash function and collision resolution
        x, y, z = pos_grid
        hash_val = (x * 73856093) ^ (y * 19349663) ^ (z * 83492791)
        return hash_val % hashmap_size
    else:
        # Tiled grid linear index
        x, y, z = pos_grid
        res_x, res_y, res_z = resolution
        return (z * res_y + y) * res_x + x
```

**Edge Cases and Pitfalls:**
-   **Boundary Conditions:** If $pos\_grid[d]$ is at the boundary (0 or $resolution[d]$), the corresponding neighbor gradient is skipped. This prevents out-of-bounds access. [S2, unofficial]
-   **Numerical Stability:** The epsilon ($10^{-9}$) in the denominator prevents division by zero when $idelta[ch]$ is zero (i.e., when the point is identical to its neighbors). [S2, unofficial]
-   **Precision:** Floating-point precision issues may arise in hash grid indexing if the hash function is not robust. [S2, unofficial]
-   **Degenerate Input:** If $inputs$ are outside the world bounds, $pos\_grid$ may be out of bounds. The implementation assumes inputs are clamped or within bounds. [G]

## Operating characteristics
-   **Time Complexity:** $O(B \cdot C \cdot D)$, where $B$ is the number of points, $C$ is the channel dimension, and $D$ is the number of spatial dimensions. Each point requires checking 2 neighbors per dimension. [S2, unofficial]
-   **Memory Complexity:** $O(C)$ per point for temporary accumulation buffers (`results` and `idelta`). No additional global memory is allocated. [S2, unofficial]
-   **Performance:** The kernel launch uses 512 threads per block, which is a common warp size multiple for efficient GPU execution. [S2, unofficial]
-   **Strengths:** Effectively reduces noise and enforces piecewise constant structures, which is beneficial for density fields in NeRFs. [S1, S2, unofficial]
-   **Limitations:** Can oversmooth fine details if the weight is too high. The scaling by world size/128 may require tuning for different scene scales. [S1, S2, unofficial]
-   **When to Use:** When the density field exhibits noisy artifacts or when piecewise constant assumptions are valid. [S1, S2, unofficial]
-   **When Not to Use:** When fine-grained details are critical and the TV penalty might blur them. [G]

## Background
Total Variation regularization is a classic technique in image processing and computer vision for denoising and edge preservation [S1, S2, unofficial]. In the context of neural radiance fields (NeRFs), it has been adapted to regularize the density or embedding grids to improve training stability and visual quality. The specific implementation described here is found in unofficial repositories for Unbounded NeRF and DNGaussian, indicating its adoption in recent variants of NeRF for unbounded scenes and dynamic Gaussian splatting. [S1, S2, unofficial]

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S2] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
