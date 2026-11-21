# Shader Occupancy Optimization for DDA

Also called: DDA Shader Register Optimization, Optimizing 2-Level DDA Shader, Shader Occupancy Optimization

## Overview
This method increases GPU shader occupancy for Digital Differential Analyzer (DDA) ray-grid traversal by reducing register pressure and minimizing state carried across nested loops [S1, unofficial]. It addresses the performance degradation caused by low occupancy (e.g., 35.43%) and high register allocation (e.g., 92 registers) in unoptimized 2-level DDA shaders, which leads to memory stalls despite high ALU usage [S1, unofficial]. Unlike general occupancy tuning, it specifically targets the variable bloat inherent in nested loop structures for voxel traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An unoptimized 2-level DDA shader implementation containing separate `min_cell` and `max_cell` variables and redundant `tdelta` calculations.
*   **Output:** An optimized shader with reduced register allocation and higher occupancy.

**Data Structures:**
*   `local_cell`: A single `int3` variable representing the current voxel coordinates.
*   `VX_BRICK_EXT`: A constant defining the brick dimensions (e.g., 8 or 16).
*   `voxel_mask`: An array of `uint` values used for sparse voxel access.

**Steps:**
1.  **Variable Consolidation:** Replace the pair of variables `min_cell` and `max_cell` with a single local variable `local_cell` [S1, unofficial].
2.  **Bounds Checking:** Implement bounds checking using a unified comparison against the brick extent [S1, unofficial]:
    ```cpp
    if (any((uint3)local_cell >= (uint)VX_BRICK_EXT)) {
        // Handle out-of-bounds or termination
    }
    ```
3.  **Deduplication:** Remove redundant `tdelta` calculations and unnecessary work within the nested loops [S1, unofficial].
4.  **Loop State Reduction:** Ensure that fewer variables are carried over the loops into registers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Optimized 2-level DDA loop structure
def optimized_dda_loop(ray_origin, ray_direction, brick_min, brick_max, voxel_mask, voxels):
    # Initialize local_cell based on ray entry
    local_cell = compute_initial_cell(ray_origin, ray_direction, brick_min)
    
    # Pre-calculate tdelta once if possible, or deduplicate if computed per axis
    # Note: Exact tdelta deduplication logic depends on specific DDA variant,
    # but the key is avoiding recomputation or storing redundant deltas in registers.
    
    while True:
        # Bounds check using single variable and brick extent
        if any((uint3)local_cell >= (uint)VX_BRICK_EXT):
            break
            
        # Compute voxel index
        voxel_idx = local_cell.x + local_cell.y * VX_BRICK_EXT + local_cell.z * VX_BRICK_EXT * VX_BRICK_EXT
        
        # Sparse voxel access
        word = voxel_mask[voxel_idx >> 5u]
        bit = 1u << (voxel_idx & 31u)
        
        if (word & bit) != 0u:
            # Load voxel data
            return voxels.Load(int4(brick_min + local_cell, 0)).r
        
        # Advance ray
        local_cell = advance_dda_step(local_cell, ray_direction)
        
        # Check for brick exit or other termination conditions
        if is_brick_exit(local_cell, brick_max):
            break
```

**Parameters:**
*   `VX_BRICK_EXT`: Brick dimension. Typical values are 8 or 16. Not given in the sources.
*   Target Register Allocation: < 72 registers (achieved 72 vs 92) [S1, unofficial].
*   Target Occupancy: > 40% (achieved 42.37% vs 35.43%) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Low Occupancy:** Unoptimized shaders may have occupancy as low as 35.43% with 92 registers, causing memory stalls [S1, unofficial].
*   **Register Pressure:** Carrying excessive state across nested loops increases register allocation, reducing the number of concurrent warps [S1, unofficial].

## Subtopics

### [CUDA Execution Configuration Optimization](15104-cuda-execution-configuration-optimization.md)
Focuses on thread block/grid dimensions and shared memory to maximize occupancy, accounting for branch divergence in ray-grid traversal.

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Reduced register usage (from 92 to 72 registers) [S1, unofficial].
*   **Time:** Performance improvements vary by scene complexity.

**Measured Performance:**
*   **monu2-1 (128³):** FPS 143.20 -> 144.00 (+0.56%), Walltime 6.58 ms -> 5.37 ms (-18.32%) [S1, unofficial].
*   **sponza-256-1:** FPS 59.78 -> 72.83 (+21.83%), Walltime 28.39 ms -> 22.51 ms (-20.71%) [S1, unofficial].
*   **sponza-256-2:** FPS 109.53 -> 143.94 (+31.42%), Walltime 13.40 ms -> 7.32 ms (-45.33%) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant walltime reduction in complex scenes (up to 45.33% faster) [S1, unofficial].
*   **Limitations:** Modest FPS gains in simpler scenes (0.56%) [S1, unofficial].

**When to Use:**
*   Use when DDA shader occupancy is low (<40%) and register allocation is high (>80 registers) [S1, unofficial].

## Background
Originated from development logs of a voxel raytracer, specifically addressing the performance drop in 2-level DDA shaders compared to naive DDA [S1, unofficial]. The unoptimized 2-level DDA shader had an occupancy of 35.43% and 92 registers, compared to 71.39% and 42 registers for the naive DDA [S1, unofficial].

## Sources
[S1] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
