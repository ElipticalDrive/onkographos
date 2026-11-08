# Voxel Erosion / Fracture

Also called: GPU-accelerated Voxel Fracture, Iterative Voxel Erosion

## Overview
Voxel Erosion / Fracture is a GPU-accelerated iterative process that removes voxels from a solid voxelized model to simulate physical fracture or weathering effects [S1, unofficial]. It differs from standard mathematical morphology by introducing probabilistic removal based on noise buffers and configurable thresholds, allowing for irregular, natural-looking fragmentation rather than uniform shrinking [S1, unofficial]. It builds on [Mathematical Morphology on Voxelization](4702-mathematical-morphology-on-voxelization.md) by replacing deterministic logical operations with stochastic compute shader passes.

## How it works
**Inputs and Outputs**
- **Input:** A solid voxel grid (binary occupancy), convolution kernel size, number of iterations, erosion probability, erosion threshold, and fracture style (kernel shape).
- **Output:** A fractured voxel grid with removed voxels.

**Data Structures**
1.  **Grid:** 3D array of voxel states (occupied/empty).
2.  **Erosion Mask:** 1D float array of size $maskSize = convolutionSize^3$. Indexed linearly as $x \cdot convolutionSize^2 + y \cdot convolutionSize + z$. Values are $1.0$ if the kernel includes the voxel, $0.0$ otherwise.
3.  **Noise Buffer:** 1D float array of size $10^6$ containing uniform random floats in $[0.0, 1.0]$, generated on the CPU [S1, unofficial].
4.  **SSBOs:** Shader Storage Buffer Objects for the grid, mask, and noise, bound to the compute shader [S1, unofficial].

**Parameters**
-   $convolutionSize$: Integer size of the 3D erosion kernel. Must be odd. If even, it is auto-incremented to the next odd integer [S1, unofficial].
-   $numIterations$: Number of sequential erosion passes [S1, unofficial].
-   $erosionProbability$: Float threshold controlling the likelihood of voxel removal [S1, unofficial].
-   $erosionThreshold$: Float threshold used in the compute shader to decide voxel removal [S1, unofficial].
-   $fractureParams$: Enum specifying kernel shape:
    -   `SQUARE`: All voxels in the kernel are $1.0$.
    -   `CROSS`: Only the central row, column, and depth axis voxels are $1.0$.
    -   `ELLIPSE`: Voxels within distance $< convolutionCenter + \epsilon$ are $1.0$ [S1, unofficial].
-   $convolutionCenter = \lfloor convolutionSize / 2.0 \rfloor$ [S1, unofficial].

**Algorithm Steps**
1.  **Kernel Preparation:**
    -   If $convolutionSize$ is even, increment by 1.
    -   Calculate $maskSize = convolutionSize^3$.
    -   Calculate $convolutionCenter = \lfloor convolutionSize / 2.0 \rfloor$.
    -   Populate `erosionMask` based on $fractureParams$:
        -   For `SQUARE`: Set all entries to $1.0$.
        -   For `CROSS`: Set entries corresponding to the central axes to $1.0$.
        -   For `ELLIPSE`: Set entries where the Euclidean distance from the center is $< convolutionCenter + \epsilon$ to $1.0$.
    -   Normalize activation count: $activations = \sum(mask) / maskSize$ [S1, unofficial].

2.  **Noise Generation:**
    -   Generate $10^6$ uniform random floats in $[0.0, 1.0]$ on the CPU.
    -   Upload to GPU as `noiseSSBO` [S1, unofficial].

3.  **Iterative Erosion Loop:**
    For each iteration $i$ from $0$ to $numIterations - 1$:
    -   Call `detectBoundaries(1)` to identify surface voxels [S1, unofficial].
    -   Bind SSBOs: `_ssbo` (grid), `marching cubes grid SSBO`, `maskSSBO`, `noiseSSBO`.
    -   Set uniforms for `_erodeShader`:
        -   $numActivationsFloat$
        -   $gridDims$
        -   $maskSize$
        -   $maskSize2$
        -   $numCells$
        -   $noiseBufferSize$
        -   $erosionProbability$
        -   $erosionThreshold$
    -   Execute `_erodeShader` compute shader.
    -   Copy the updated grid via `_copyGridShader` [S1, unofficial].

4.  **Post-Processing:**
    -   Remove isolated regions (disconnected components) from the grid.
    -   Update grid state.
    -   Delete `maskSSBO` and `noiseSSBO` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_fracture(grid, convolution_size, num_iterations, erosion_probability, erosion_threshold, fracture_params):
    # 1. Kernel Preparation
    if convolution_size % 2 == 0:
        convolution_size += 1
    
    mask_size = convolution_size ** 3
    convolution_center = floor(convolution_size / 2.0)
    
    erosion_mask = [0.0] * mask_size
    
    if fracture_params == "SQUARE":
        for i in range(mask_size):
            erosion_mask[i] = 1.0
    elif fracture_params == "CROSS":
        # Set central row, column, depth to 1.0
        cx, cy, cz = convolution_center, convolution_center, convolution_center
        for x in range(convolution_size):
            erosion_mask[x * convolution_size**2 + cy * convolution_size + cz] = 1.0
            erosion_mask[cx * convolution_size**2 + y * convolution_size + cz] = 1.0 # Note: y loop needed in real impl
            erosion_mask[cx * convolution_size**2 + cy * convolution_size + z] = 1.0 # Note: z loop needed in real impl
        # Simplified logic for CROSS:
        for x in range(convolution_size):
            for y in range(convolution_size):
                for z in range(convolution_size):
                    if (x == cx) or (y == cy) or (z == cz):
                        erosion_mask[x * convolution_size**2 + y * convolution_size + z] = 1.0
    elif fracture_params == "ELLIPSE":
        for x in range(convolution_size):
            for y in range(convolution_size):
                for z in range(convolution_size):
                    dist = sqrt((x-cx)**2 + (y-cy)**2 + (z-cz)**2)
                    if dist < convolution_center + epsilon:
                        erosion_mask[x * convolution_size**2 + y * convolution_size + z] = 1.0
    
    activations = sum(erosion_mask) / mask_size
    
    # 2. Noise Generation
    noise_buffer = generate_random_floats(1000000, min_val=0.0, max_val=1.0)
    
    # 3. Iterative Erosion
    for i in range(num_iterations):
        # detectBoundaries(1) identifies surface voxels
        boundaries = detect_boundaries(grid)
        
        # Bind SSBOs and set uniforms
        bind_ssbo(grid_ssbo, grid)
        bind_ssbo(mask_ssbo, erosion_mask)
        bind_ssbo(noise_ssbo, noise_buffer)
        
        set_uniforms(erode_shader, {
            "numActivationsFloat": activations,
            "gridDims": grid.dimensions,
            "maskSize": mask_size,
            "maskSize2": mask_size * 2, # As per source
            "numCells": grid.num_cells,
            "noiseBufferSize": 1000000,
            "erosionProbability": erosion_probability,
            "erosionThreshold": erosion_threshold
        })
        
        execute_compute_shader(erode_shader)
        
        # Copy grid back
        grid = copy_grid(grid_ssbo)
        
    # 4. Post-Processing
    grid = remove_isolated_regions(grid)
    delete_ssbo(mask_ssbo)
    delete_ssbo(noise_ssbo)
    
    return grid
```

**Edge Cases and Pitfalls**
-   **Even Convolution Size:** The kernel centering requires an odd dimension. Even sizes must be incremented to the next odd number to maintain symmetry [S1, unofficial].
-   **Precision:** Floating point comparisons for distance in `ELLIPSE` mode should use a small epsilon to avoid boundary artifacts [S1, unofficial].
-   **Isolated Regions:** After erosion, disconnected fragments may remain. A post-processing step to remove isolated regions is required to clean up noise [S1, unofficial].

## Subtopics

### GPU-accelerated Voxel Fracture
The specific implementation using OpenGL compute shaders for parallel voxel processing. Choose when performance on large voxel grids is critical. [Link](https://github.com/AlfonsoLRz/VoxelFragmentML)

### Iterative Voxel Erosion
The sequential application of erosion passes. Choose when gradual weathering or controlled fragmentation is needed. [Link](https://github.com/AlfonsoLRz/VoxelFragmentML)

## Operating characteristics
-   **Time Complexity:** $O(numIterations \cdot N)$, where $N$ is the number of voxels in the grid. Each iteration involves a full grid pass and boundary detection.
-   **Memory Complexity:** $O(N + maskSize + 10^6)$ for the grid, mask, and noise buffer.
-   **Performance:** Measured on archaeological artefact datasets (Iberian vessels). Specific hardware metrics are not provided in the sources.
-   **Strengths:** GPU acceleration allows for fast processing of large voxel grids. Probabilistic removal creates natural-looking fractures.
-   **Limitations:** Requires GPU with OpenGL 4.6 support. Noise buffer size is fixed at $10^6$ floats, which may limit randomness for very large grids or many iterations.
-   **When to Use:** Generating training data for fragment assembly models, simulating weathering, or creating fractured assets for games/VR.
-   **When Not to Use:** When deterministic, uniform erosion is required (use standard mathematical morphology). When CPU-only implementation is necessary.

## Background
The method was developed for the *VoxelFragmentML* project, aimed at generating datasets for training fragment assembly models, specifically for archaeological artefacts like Iberian vessels [S1, unofficial]. It uses OpenGL compute shaders for GPU acceleration. Dependencies include Assimp, GLEW, OpenGL 4.6, GLFW3, GLM, and Fast-Quadric-Mesh-Simplification [S1, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
