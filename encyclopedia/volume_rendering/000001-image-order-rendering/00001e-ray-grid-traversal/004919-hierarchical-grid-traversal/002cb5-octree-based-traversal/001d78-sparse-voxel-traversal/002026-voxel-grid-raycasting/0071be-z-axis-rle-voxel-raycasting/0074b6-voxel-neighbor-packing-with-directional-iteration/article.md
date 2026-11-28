# Voxel Neighbor Packing with Directional Iteration

Also called: Voxel Data Processing with Neighbor Packing and Directional Iteration, subcube processing, view/integrate/layer/z-shade rendering

## Overview
A high-performance voxel rendering method that accelerates ray traversal by packing four consecutive voxels into a single 32-bit integer to enable bulk zero-skipping, while maintaining strict back-to-front rendering order via a pre-computed directional flag. It differs from [Z-axis RLE Voxel Raycasting](22360-z-axis-rle-voxel-raycasting.md) by operating on fixed-width subcubes with byte-level packing rather than variable-length run-length encoding, optimizing for dense or semi-dense regions where RLE overhead might outweigh packing benefits.

## How it works
**Inputs:**
- `subcube.data`: A 3D array of `SHORT` voxels, accessed via slices `subcube.data[idx.z][idx.y][idx.x]`.
- `subcube.image`: A 2D output array indexed by fixed-point coordinates, type `BYTE` (or `INT` depending on mode).
- `threshold.value`: Minimum intensity for a voxel to be considered visible.
- `z.cut.i`: Maximum Z-depth for rendering; voxels with $Z > z.cut.i$ are clipped.
- `forward.x`: Boolean flag determining processing order along the X-axis (TRUE: low-to-high index, FALSE: high-to-low).
- `layer.value`: Constant value for LAYER mode.
- `unit.step.*`: Fixed-point increments for X, Y, Z coordinates.
- `idx.step.*`: Integer increments for voxel indices.

**Outputs:**
- `subcube.image`: The rendered 2D projection.

**Data Structures:**
- `dummy`: A 32-bit integer (`INT32`) holding four consecutive `SHORT` voxels.
- `dummy.byte`: A byte-array view of `dummy`, where `dummy.byte[0]` corresponds to the first voxel, `dummy.byte[1]` to the second, etc.
- Fixed-point accumulators: `xxi, xyi, xzi` (X-axis), `yxi, yyi, yzi` (Y-axis), `zxi, zyi, zzi` (Z-axis). These track the current position in fixed-point format (16.1 format implied by `INT xyi.16.1`).

**Algorithm Steps:**

1.  **Initialize Z-loop:**
    - Set `idx.z` to `idx.start.z`.
    - Set `zxi` to `start.xi`, `zyi` to `start.yi`, `zzi` to `start.zi`.
    - Loop `z.count` times (or until `idx.run.z` limit):
        - Check `z.cut.value`. If present, handle clipping logic.

2.  **Initialize Y-loop:**
    - Set `idx.y` to `idx.start.y`.
    - Set `yxi` to `zxi`, `yyi` to `zyi`, `yzi` to `zzi`.
    - Loop `y.count` times (or until `idx.run.y` limit):
        - Access slice `short_i = subcube.data.slice[idx.y]`.
        - Retype `short_i` as `[SCDX >> 2] INT short_i` to prepare for packing.

3.  **Initialize X-loop:**
    - Set `idx.x` to `idx.start.x`.
    - Set `xxi` to `yxi`, `xyi` to `yyi`, `xzi` to `yzi`.
    - Loop `x.count` times (or until `idx.run.x` limit):

4.  **Pack and Skip:**
    - Load four consecutive voxels into `dummy`:
      ```python
      dummy = INT32(short_i[idx.x], short_i[idx.x + 1], short_i[idx.x + 2], short_i[idx.x + 3])
      ```
      *(Note: The source implies a specific byte mapping where `dummy.byte[0]` is the first voxel. Endianness determines which physical byte corresponds to which voxel index. The source assumes `dummy.0` maps to `dummy.byte[0]`.)*
    - Check if `dummy == 0`.
    - **If all four are zero:**
        - Skip four voxels:
          ```python
          idx.x += idx.step.x * 4
          xxi += unit.step.4.xxi
          xyi += unit.step.4.xyi
          xzi += unit.step.4.xzi
          ```
        - Continue to next iteration of X-loop.
    - **If not all zero:**
        - Process each of the four voxels individually based on `forward.x`.

5.  **Process Voxels:**
    - **If `forward.x` is TRUE:**
        - Process `dummy.byte[0]` (Voxel 0).
        - Process `dummy.byte[1]` (Voxel 1).
        - Process `dummy.byte[2]` (Voxel 2).
        - Process `dummy.byte[3]` (Voxel 3).
        - After each voxel, increment accumulators:
          ```python
          xxi += unit.step.xxi
          xyi += unit.step.xyi
          xzi += unit.step.xzi
          ```
    - **If `forward.x` is FALSE:**
        - Process `dummy.byte[3]` (Voxel 3).
        - Process `dummy.byte[2]` (Voxel 2).
        - Process `dummy.byte[1]` (Voxel 1).
        - Process `dummy.byte[0]` (Voxel 0).
        - After each voxel, increment accumulators:
          ```python
          xxi += unit.step.xxi
          xyi += unit.step.xyi
          xzi += unit.step.xzi
          ```

6.  **Voxel Processing Logic (per voxel `v`):**
    - Let `idx_2d = [INT(xyi.16.1), INT(xxi.16.1)]`.
    - Let `z_dist = xzi`.
    - Condition: `IF (v > threshold.value) AND (z_dist < z.cut.i)`:
        - **VIEW Mode:**
          ```python
          subcube.image[idx_2d] = v
          ```
        - **INTEGRATE Mode:**
          ```python
          density = subcube.image[idx_2d]
          IF density < 255:
              density = min(255, density + 1)
          subcube.image[idx_2d] = density
          ```
        - **LAYER Mode:**
          ```python
          subcube.image[idx_2d] = layer.value
          ```
        - **Z SHADE Mode:**
          ```python
          subcube.image[idx_2d] = BYTE((INT(xzi.16.1)) >> 1)
          ```
    - **Else:**
      - Skip (do not update `subcube.image`).

7.  **Increment Y-loop:**
    - After X-loop completes:
      ```python
      idx.y += idx.step.y
      yxi += unit.step.yxi
      yyi += unit.step.yyi
      yzi += unit.step.yzi
      ```

8.  **Increment Z-loop:**
    - After Y-loop completes:
      ```python
      idx.z += idx.step.z
      zxi += unit.step.zxi
      zyi += unit.step.zyi
      zzi += unit.step.zzi
      ```

9.  **Merge:**
    - Send results to the merger module for combination with other Processing Elements (PEs).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_subcube(subcube_data, subcube_image, threshold_value, z_cut_i, forward_x, layer_value):
    # Initialize Z indices
    idx_z = idx_start_z
    zxi = start_xi
    zyi = start_yi
    zzi = start_zi
    
    for _ in range(z_count):
        # Initialize Y indices
        idx_y = idx_start_y
        yxi = zxi
        yyi = zyi
        yzi = zzi
        
        # Access current slice
        slice_data = subcube_data[idx_z]
        
        for _ in range(y_count):
            # Initialize X indices
            idx_x = idx_start_x
            xxi = yxi
            xyi = yyi
            xzi = yzi
            
            # Prepare packed data view
            # Assuming slice_data is an array of SHORTs
            # We process in chunks of 4
            num_voxels = x_count
            i = 0
            while i < num_voxels:
                # Pack 4 voxels into INT32
                # Note: Source implies specific byte mapping. 
                # dummy.byte[0] is the first voxel in the pack.
                if i + 3 < num_voxels:
                    v0 = slice_data[idx_x + 0]
                    v1 = slice_data[idx_x + 1]
                    v2 = slice_data[idx_x + 2]
                    v3 = slice_data[idx_x + 3]
                    
                    # Construct INT32. Endianness matters for byte mapping.
                    # Source: dummy.0 IS dummy.byte[0]
                    # We assume a layout where byte 0 is the first element.
                    dummy = (v0 << 24) | (v1 << 16) | (v2 << 8) | v3 
                    # Or depending on endianness, the shift order might vary.
                    # However, the source explicitly maps dummy.0 to byte[0].
                    # Let's use a byte array view for clarity.
                    dummy_bytes = [v0, v1, v2, v3] # Placeholder for packing logic
                    
                    # Check if all zero
                    if v0 == 0 and v1 == 0 and v2 == 0 and v3 == 0:
                        # Skip 4 voxels
                        idx_x += idx_step_x * 4
                        xxi += unit_step_4_xxi
                        xyi += unit_step_4_xyi
                        xzi += unit_step_4_xzi
                        i += 4
                        continue
                    else:
                        # Process individually
                        voxels = [v0, v1, v2, v3]
                        if forward_x:
                            indices = [0, 1, 2, 3]
                        else:
                            indices = [3, 2, 1, 0]
                            
                        for idx in indices:
                            v = voxels[idx]
                            process_voxel(v, xxi, xyi, xzi, subcube_image, threshold_value, z_cut_i, layer_value)
                            xxi += unit_step_xxi
                            xyi += unit_step_xyi
                            xzi += unit_step_xzi
                        i += 4
                else:
                    # Handle remaining voxels if count is not a multiple of 4
                    # Source doesn't explicitly detail non-multiple handling, 
                    # but implies loop runs for x_count.
                    # We assume x_count is a multiple of 4 or handle remainder.
                    # For safety, process remaining one by one.
                    while i < num_voxels:
                        v = slice_data[idx_x]
                        process_voxel(v, xxi, xyi, xzi, subcube_image, threshold_value, z_cut_i, layer_value)
                        xxi += unit_step_xxi
                        xyi += unit_step_xyi
                        xzi += unit_step_xzi
                        idx_x += idx_step_x
                        i += 1

            # Increment Y
            idx_y += idx_step_y
            yxi += unit_step_yxi
            yyi += unit_step_yyi
            yzi += unit_step_yzi
            
        # Increment Z
        idx_z += idx_step_z
        zxi += unit_step_zxi
        zyi += unit_step_zyi
        zzi += unit_step_zzi

def process_voxel(v, xxi, xyi, xzi, subcube_image, threshold_value, z_cut_i, layer_value):
    if v > threshold_value and xzi < z_cut_i:
        idx_2d = (int(xyi >> 1), int(xxi >> 1)) # Assuming 16.1 format means shift 1
        # Note: Source uses INT(xyi.16.1). In fixed point 16.1, the integer part is bits 16..1.
        # If xyi is stored as an integer representing Q16.1, then xyi >> 1 gives the integer part.
        
        # VIEW Mode
        # subcube_image[idx_2d] = v
        
        # INTEGRATE Mode
        density = subcube_image[idx_2d]
        if density < 255:
            density = min(255, density + 1)
        subcube_image[idx_2d] = density
        
        # LAYER Mode
        # subcube_image[idx_2d] = layer_value
        
        # Z SHADE Mode
        # subcube_image[idx_2d] = (int(xzi >> 1)) >> 1
        
        # The source lists these as separate modes. 
        # In a single implementation, one mode is active.
        # Here we show the logic for INTEGRATE as an example, 
        # but the method supports all.
        pass
```

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the subcube, but with a significant constant factor improvement due to skipping zero regions. The skip operation processes 4 voxels in the time of 1 check.
- **Memory Complexity:** $O(1)$ auxiliary space per subcube, plus the output image.
- **Performance:** Highly efficient for sparse volumes or volumes with large empty regions, as the `dummy == 0` check allows skipping 4 voxels at once. Performance degrades if most voxels are non-zero, as the overhead of unpacking and individual processing remains.
- **Strengths:** Simple implementation, low memory overhead, effective for skipping empty space.
- **Limitations:** Requires fixed-size subcubes (multiples of 4 voxels in X). Endianness-dependent byte mapping. No hierarchical acceleration beyond the subcube level.
- **When to use:** When rendering semi-dense volumes with significant empty space, and when hardware supports efficient 32-bit integer operations.
- **When not to use:** For very dense volumes where skipping is rare, or for volumes with irregular structures that don't align with the packing grid.

## Background
Originated in the context of voxel data processing on Transputer networks, as described in [S1, unofficial]. The method was designed to optimize rendering performance by reducing memory bandwidth and leveraging parallel processing elements (PEs). It is part of a pipeline that includes subcube processing and merging.

## Sources
[S1] DTIC ADA245410: Voxel Data Processing on a Transputer Network, https://archive.org/details/DTIC_ADA245410, fetched 2026-10-26 (unofficial)
