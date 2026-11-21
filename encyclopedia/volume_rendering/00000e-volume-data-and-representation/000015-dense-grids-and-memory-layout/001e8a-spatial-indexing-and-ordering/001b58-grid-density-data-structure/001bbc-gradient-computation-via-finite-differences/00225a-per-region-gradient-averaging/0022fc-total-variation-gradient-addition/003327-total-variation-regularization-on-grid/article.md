# Total Variation Regularization on Grid

Also called: DVGO Density Total Variation, TV Regularization

## Overview
Total Variation (TV) regularization on a grid adds a gradient term to the density or embedding grid to encourage piecewise constant solutions [S1, unofficial]. It computes the gradient of the total variation loss with respect to the grid embeddings to enforce smoothness [S2, unofficial]. The method operates directly on the accumulated gradients before the optimizer step, distinguishing it from loss-based regularization that modifies the scalar loss value [S2, unofficial].

## How it works
**Inputs:**
- `grid`: The dense grid tensor containing embeddings or density values.
- `grad`: The accumulated gradient buffer for the grid, same shape as `grid`.
- `inputs`: The 3D coordinates of the points being optimized, shape `[B, 3]`.
- `resolution`: The spatial resolution of the grid, scalar or tuple `(res_x, res_y, res_z)`.
- `world_size`: The physical dimensions of the world bounding box, used for coordinate scaling.
- `weight`: Scalar weight for the total variation loss [S2, unofficial].
- `align_corners`: Boolean flag indicating if grid corners align with coordinate boundaries.
- `hashmap_size`: Size of the hash map (for hash grids).
- `gridtype`: Type of grid (`'hash'` or `'tiled'`).
- `D`: Number of spatial dimensions (typically 3).
- `L`: Number of resolution levels (for multi-resolution grids).

**Outputs:**
- `grad`: Updated gradient buffer with TV regularization terms added.

**Data Structures:**
- `grid` and `grad`: Float tensors of shape `[num_embeddings, C]` where `C` is the channel dimension [S2, unofficial].
- `inputs`: Float tensor of shape `[B, D]`.
- `results` and `idelta`: Temporary per-point accumulation buffers of shape `[C]` [S2, unofficial].

**Parameters:**
- `weight`: Scalar multiplier for the TV loss [S2, unofficial].
- `N_THREAD`: 512 threads per GPU block [S2, unofficial].
- `epsilon`: $10^{-9}$ for numerical stability in the denominator [S2, unofficial].

**Steps:**
1. **Scale Weight:** Compute the effective weight by scaling the input `weight` by the maximum dimension of the world size divided by 128:
   $$ w_{scaled} = weight \times \frac{\max(world\_size)}{128} $$
   [S1, unofficial]
2. **Kernel Launch Configuration:** Set `N_THREAD = 512`. Calculate grid dimensions:
   $$ blocks\_x = \lceil B / N\_THREAD \rceil $$
   $$ blocks\_y = L $$
   [S2, unofficial]
3. **Coordinate Calculation:** For each point $i$ with coordinates $inputs[i]$:
   $$ pos[d] = inputs[i][d] \times scale + (align\_corners ? 0.0f : 0.5f) $$
   $$ pos\_grid[d] = \lfloor pos[d] \rfloor $$
   where $scale$ maps world coordinates to grid coordinates (typically $resolution / world\_size$) [G].
4. **Hash Index Calculation:** Compute the base grid index:
   $$ index = get\_grid\_index(gridtype, align\_corners, 0, hashmap\_size, resolution, pos\_grid) $$
   [S2, unofficial]
5. **Gradient Accumulation Loop:** Initialize $results[ch] = 0$ and $idelta[ch] = 0$ for each channel $ch$. For each dimension $d \in \{0, 1, 2\}$:
   - **Right Neighbor:** If $pos\_grid[d] < resolution[d]$, increment $pos\_grid[d]$, compute $index\_right$, and accumulate:
     $$ grad\_val = grid[index + ch] - grid[index\_right + ch] $$
     $$ results[ch] \leftarrow results[ch] + grad\_val $$
     $$ idelta[ch] \leftarrow idelta[ch] + grad\_val^2 $$
   - **Left Neighbor:** Decrement $pos\_grid[d]$ to restore the original position. If $pos\_grid[d] > 0$, compute $index\_left$, and accumulate:
     $$ grad\_val = grid[index + ch] - grid[index\_left + ch] $$
     $$ results[ch] \leftarrow results[ch] + grad\_val $$
     $$ idelta[ch] \leftarrow idelta[ch] + grad\_val^2 $$
   [S2, unofficial]
6. **Weight Calculation:** Normalize the scaled weight by the number of dimensions:
   $$ w = \frac{w_{scaled}}{2 \times D} $$
   [S2, unofficial]
7. **Atomic Write:** Update the gradient buffer for each channel:
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
    
    # Process each point (simulating kernel launch with B / N_THREAD blocks)
    for i in range(B):
        # Coordinate Calculation
        pos = [0.0] * D
        for d in range(D):
            scale = resolution[d] / max_world_dim
            pos[d] = inputs[i][d] * scale + (0.0 if align_corners else 0.5)
            
        pos_grid = [int(math.floor(p)) for p in pos]
        
        # Hash Index Calculation
        # Note: The sources do not specify the exact indexing implementation.
        # Standard hash/tiled indexing would be applied here.
        index = (pos_grid[0] * 1000003 + pos_grid[1] * 1000033 + pos_grid[2] * 1000193) % hashmap_size
        
        # Initialize accumulation buffers
        results = [0.0] * C
        idelta = [0.0] * C
        
        # Gradient Accumulation Loop
        for d in range(D):
            # Right Neighbor
            if pos_grid[d] < resolution[d]:
                pos_grid[d] += 1
                index_right = (pos_grid[0] * 1000003 + pos_grid[1] * 1000033 + pos_grid[2] * 1000193) % hashmap_size
                for ch in range(C):
                    grad_val = grid[index][ch] - grid[index_right][ch]
                    results[ch] += grad_val
                    idelta[ch] += grad_val * grad_val
                pos_grid[d] -= 1 # Restore original position
            
            # Left Neighbor
            if pos_grid[d] > 0:
                pos_grid[d] -= 1
                index_left = (pos_grid[0] * 1000003 + pos_grid[1] * 1000033 + pos_grid[2] * 1000193) % hashmap_size
                for ch in range(C):
                    grad_val = grid[index][ch] - grid[index_left][ch]
                    results[ch] += grad_val
                    idelta[ch] += grad_val * grad_val
                pos_grid[d] += 1 # Restore original position
                
        # Weight Calculation
        w = w_scaled / (2.0 * D)
        
        # Atomic Write (simulated with direct assignment for CPU pseudocode)
        for ch in range(C):
            norm_factor = 1.0 / math.sqrt(idelta[ch] + 1e-9)
            grad[index][ch] += w * results[ch] * norm_factor
            
    return grad
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The `if` checks for $pos\_grid[d] < resolution[d]$ and $pos\_grid[d] > 0$ prevent accessing out-of-bounds grid cells [S2, unofficial].
- **Numerical Stability:** The epsilon term ($10^{-9}$) prevents division by zero when $idelta[ch]$ is zero [S2, unofficial].
- **Precision:** Floating-point rounding in coordinate-to-grid mapping may cause adjacent points to hash to the same cell, which is handled by the accumulation logic [G].
- **Degenerate Input:** If $inputs$ fall outside the world bounds, $pos\_grid$ may exceed resolution limits; the implementation assumes inputs are clamped or validated upstream [G].

## Operating characteristics
- **Time Complexity:** $O(B \cdot C \cdot D)$ per kernel launch, as each point evaluates two neighbors across $D$ dimensions for $C$ channels [S2, unofficial].
- **Memory Complexity:** $O(C)$ temporary storage per point for `results` and `idelta` buffers [S2, unofficial].
- **Performance:** The kernel uses 512 threads per block, aligning with standard GPU warp sizes for coalesced memory access [S2, unofficial].
- **Strengths:** Enforces piecewise constant structures and smoothness directly on hash or tiled grid embeddings during optimization [S2, unofficial].
- **Limitations:** The fixed scaling factor ($world\_size / 128$) may require manual adjustment for scenes with extreme aspect ratios or scale variations [S1, S2, unofficial].
- **When to Use:** When regularizing density or embedding grids in neural rendering pipelines to reduce high-frequency noise [S1, S2, unofficial].
- **When Not to Use:** Not given in the sources.

## Background
The method originates from implementations in unofficial repositories for Unbounded NeRF and DNGaussian [S1, S2, unofficial]. It adapts classical total variation regularization, traditionally used in image processing, to the gradient domain of neural radiance field grid encoders [S2, unofficial]. The technique is applied directly to the gradient accumulation stage for hash and tiled grids, bypassing explicit loss recomputation [S2, unofficial].

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S2] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
