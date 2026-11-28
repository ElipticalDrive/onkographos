# Brick Size and Octree Limit Tuning

Also called: Hardware-specific Bricking, Performance Optimization via Brick Size and Octree Limit Tuning, Sampling Direction Optimization

## Overview
Brick Size and Octree Limit Tuning optimizes GPU-based volume rendering by selecting the optimal dimensions for spatial bricks and the minimum depth of the hierarchical octree structure to balance rasterization load, triangle throughput, and texture memory constraints [S1, unofficial]. It distinguishes itself from sibling methods by treating brick size and octree limit as orthogonal variables that must be tuned specifically for the target hardware's texture sampling characteristics and memory hierarchy, rather than relying on a single fixed subdivision strategy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset, target graphics hardware specifications (texture memory size, sampling direction penalties), and rendering parameters (view direction, transfer function) [G].
*   **Output:** Optimized brick dimensions ($B_x, B_y, B_z$) and octree limit ($L$) [S1, unofficial].

**Data Structures:**
The method operates on the existing hierarchical grid structures defined in [Hierarchical and Block-Based Data Structures](4392-hierarchical-and-block-based-data-structures.md) [G]. The key tunable parameters are:
1.  **Brick Size:** The dimensions of the uniform 3D texture blocks. Sizes are typically powers of 2 to align with hardware texture requirements [S1, unofficial].
2.  **Octree Limit:** The smallest cube size allowed in the octree subdivision. Nodes smaller than this limit are not subdivided further, regardless of data sparsity [S1, unofficial].

**Steps for Tuning:**
1.  **Determine Hardware Constraints:** Identify the optimal texture size for the target GPU and any sampling direction penalties (e.g., linear vs. z-axis traversal) [S1, unofficial].
2.  **Fix One Variable:** Since brick size and octree limit are orthogonal, fix the octree limit to a candidate value (e.g., $8^3$ voxels) while varying the brick size [S1, unofficial].
3.  **Measure Performance:** Render the volume with different brick sizes and record frame rates [S1, unofficial]. If the viewing direction differs from the texture z-axis by more than $45^\circ$, z-direction traversal becomes inevitable; to mitigate hardware-specific z-direction performance penalties, alternate the orientation of the textures for each consecutive brick [S1, unofficial].
4.  **Fix Brick Size:** Set the brick size to the value yielding peak performance from Step 3 [S1, unofficial].
5.  **Vary Octree Limit:** Render the volume with different octree limits while keeping the brick size constant [S1, unofficial].
6.  **Select Optimal Octree Limit:** Identify the limit that balances rasterization load and triangle throughput [S1, unofficial]. Smaller limits cause high CPU overhead and excessive triangle counts, while larger limits cause rasterization overhead [S1, unofficial].

**Formulas:**
The volume rendering integral is evaluated in discrete steps for Early Ray Termination (ERT), which requires front-to-back traversal:
$$C_{i+1} = (1 - A_i) \cdot \alpha_i \cdot c_i + C_i$$
$$A_{i+1} = (1 - A_i) \cdot \alpha_i + A_i$$
Where $C$ is accumulated color, $A$ is accumulated opacity, $\alpha_i$ is sample opacity, and $c_i$ is sample color [S1, unofficial].

**Parameters:**
*   **Brick Size:** Depends on available texture memory, optimal texture size (power of 2), nature of the data set, overlap overhead, and hardware constraints [S1, unofficial].
*   **Octree Limit:** Balances rasterization load and triangle throughput [S1, unofficial].
*   **Sample Rate:** Typically 1.5 samples per voxel for performance measurements [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sampling Direction:** On hardware treating 3D textures as a stack of 2D slices (e.g., ATi FireGL X1), linear traversal (x or y) is much faster than z-direction traversal. Performance varies significantly based on viewing angle relative to texture axes [S1, unofficial].
*   **Optimal Values are Hardware-Specific:** The optimal brick size and octree limit differ for each graphics card and sampling direction. For example, larger bricks benefit from linear traversal in the xy-plane, while smaller bricks benefit from reduced cache trashing in other directions [S1, unofficial].
*   **Sparsity:** Early Ray Termination provides no performance gain for sparse data sets if the volume is already optimized for space-skipping [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time
import math

def tune_bricks_and_octree(volume, hardware_specs, view_direction):
    # Step 1: Determine hardware constraints
    optimal_texture_size = hardware_specs.get('optimal_texture_size')
    has_z_penalty = hardware_specs.get('has_z_penalty', False)
    texture_z_axis = hardware_specs.get('texture_z_axis', (0, 0, 1))

    # Step 2: Fix Octree Limit and vary Brick Size
    fixed_octree_limit = 8  # Example: 8^3 voxels
    best_brick_size = None
    max_fps = 0.0

    for brick_size in [4, 8, 16, 32, 64]:
        # If z-penalty exists and view direction is not aligned, consider orientation
        if has_z_penalty and not vectors_aligned(view_direction, texture_z_axis, threshold_degrees=45):
            # Alternate orientation for consecutive bricks to mitigate penalty
            pass

        # Measure performance for current brick size
        fps = measure_render_performance(volume, brick_size, fixed_octree_limit, hardware_specs)
        if fps > max_fps:
            max_fps = fps
            best_brick_size = brick_size

    # Step 3: Fix Brick Size and vary Octree Limit
    fixed_brick_size = best_brick_size
    best_octree_limit = None
    max_fps = 0.0

    for octree_limit in [4, 8, 16, 32, 64]:
        # Measure performance for current octree limit
        fps = measure_render_performance(volume, fixed_brick_size, octree_limit, hardware_specs)
        if fps > max_fps:
            max_fps = fps
            best_octree_limit = octree_limit

    return best_brick_size, best_octree_limit

def vectors_aligned(v1, v2, threshold_degrees):
    dot = v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
    mag1 = math.sqrt(v1[0]**2 + v1[1]**2 + v1[2]**2)
    mag2 = math.sqrt(v2[0]**2 + v2[1]**2 + v2[2]**2)
    if mag1 == 0 or mag2 == 0: return False
    angle = math.degrees(math.acos(dot / (mag1 * mag2)))
    return angle <= threshold_degrees

def measure_render_performance(volume, brick_size, octree_limit, hardware_specs):
    # Build octree with specified limit
    # Recursively subdivide volume until leaf cube size <= octree_limit
    # Create 3D textures for bricks
    # Partition volume into uniform 3D texture blocks of size brick_size
    # Apply orientation alternation if hardware has z-sampling penalty
    # Traverse and render
    start_time = time.perf_counter()
    for _ in range(100):
        # Traverse octree, sample textures, apply ERT formula
        pass
    end_time = time.perf_counter()
    return 100.0 / (end_time - start_time)
```

## Operating characteristics
*   **Time Complexity:** Octree build time is bounded by CPU performance. For a $512^3$ volume (256MB), building octrees took 6.5ms on a Xeon 1.7GHz and 3.5ms on a Xeon 3.0GHz [S1, unofficial].
*   **Memory Complexity:** Depends on brick size and available texture memory. Smaller bricks increase overlap overhead; larger bricks require more texture memory [S1, unofficial].
*   **Performance:** Significant acceleration is achieved over non-optimized GPU rendering. QuadroFX 3400: 73.5 fps (optimized) vs 9.6 fps (non-optimized), 7.66x acceleration [S1, unofficial]. ATi FireGL X1 (xy aligned): 83.3 fps (optimized) vs 0.23 fps (non-optimized), 362x acceleration [S1, unofficial]. ATi FireGL X1 (non xy aligned): 27.4 fps (optimized) vs 0.23 fps (non-optimized), 119x acceleration [S1, unofficial]. 3DLabs Wildcat 7110: 21.3 fps (optimized) vs 0.38 fps (non-optimized), 56.1x acceleration [S1, unofficial].
*   **Scaling:** Performance gains are hardware-dependent. Replacing a Xeon 3.0GHz with a 1.7GHz did not significantly affect rendering frame rates, as rendering is GPU-bound [S1, unofficial].
*   **Strengths:** High acceleration on sparse data; effective use of texture memory; mitigates hardware-specific bottlenecks [S1, unofficial].
*   **Limitations:** Requires hardware-specific tuning; complex to implement due to varying hardware behaviors; early ray termination is ineffective for sparse data if space-skipping is already optimal [S1, unofficial].

## Background
This method originates from research presented in [S1, unofficial]. It addresses the bottlenecks in the graphics pipeline for GPU-based volume rendering, specifically focusing on data locality, cache trashing, and hardware-specific texture sampling penalties. The approach combines bricking, octree acceleration, and early ray termination to lift these bottlenecks [S1, unofficial].

## Sources
[S1] Optimizing GPU Volume Rendering - Danny Ruijters, https://dannyruijters.nl/docs/OptimizingGPUVolumeRendering_final.pdf, fetched 2026-10-26 (unofficial)
