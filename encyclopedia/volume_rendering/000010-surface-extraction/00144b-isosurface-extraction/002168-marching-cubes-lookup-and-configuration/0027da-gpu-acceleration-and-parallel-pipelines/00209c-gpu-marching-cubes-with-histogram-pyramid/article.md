# GPU Marching Cubes with Histogram Pyramid

Also called: GPU Marching Cubes with HistoPyramid, GPU Marching Cubes with HistoPyramid-style Compaction, GPU-MC, Histogram Pyramid Marching Cubes, OpenCL Marching Cubes

## Overview
GPU Marching Cubes with Histogram Pyramid is a parallel isosurface extraction method that uses a Histogram Pyramid (HP) to count triangles per voxel and compactly store vertex data in a Vertex Buffer Object (VBO) without atomic operations or sorting [S1, S2, unofficial]. It differs from sibling methods like [GPU Marching Cubes (Compute Shader)](4687-gpu-marching-cubes-compute-shader.md) by replacing atomic counters with a hierarchical reduction structure, and from [Capacity-Safe Marching Cubes Pipeline](4596-capacity-safe-marching-cubes-pipeline.md) by using a top-down traversal of the HP to write geometry directly, avoiding a separate prefix-sum allocation pass [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Scalar field volume: 3D grid of size $N_x \times N_y \times N_z$ (padded to power-of-two $SIZE \times SIZE \times SIZE$). Data type: 8-bit unsigned integer (`CL_UNSIGNED_INT8` or `char`) [S1, S2, unofficial].
*   Isosurface threshold: `isolevel` (default 51 for 8-bit data, normalized to $[0,1]$ by dividing by 255.0 for display) [S1, unofficial].
*   Lookup tables: `numVertsTable` (uint[256]) and `triTable` (uint[256*16]) from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) [S1, S2, unofficial].

**Outputs:**
*   Vertex Buffer Object (VBO): Stores triangle vertices. Each triangle has 3 vertices. Each vertex has 3 floats for position and 3 floats for normal (24 bytes total). Total size: $totalSum \times 18 \times 4$ bytes, where $totalSum$ is the total triangle count [S1, unofficial].

### Data Structures
1.  **Histogram Pyramid (HP):** A hierarchical structure storing partial sums of triangle counts.
    *   **3D Texture Path (Preferred):**
        *   Level 0: `CL_RGBA`, `CL_UNSIGNED_INT8`, size $SIZE/2 \times SIZE/2 \times SIZE/2$ [S1, unofficial].
        *   Level 1: `CL_R`, `CL_UNSIGNED_INT8`, size $SIZE/4 \times SIZE/4 \times SIZE/4$ [S1, unofficial].
        *   Levels 2–4: `CL_R`, `CL_UNSIGNED_INT16`, sizes halving each level ($SIZE/8$ to $SIZE/32$) [S1, unofficial].
        *   Level 5+: `CL_R`, `CL_UNSIGNED_INT32`, sizes halving each level until 1 (or forced to 2 if 1) [S1, unofficial].
    *   **Buffer Path (Fallback for NVIDIA):**
        *   Level 0: `char`, size $SIZE^3/8$ [S1, unofficial].
        *   Level 1: `char`, size $SIZE^3/64$ [S1, unofficial].
        *   Level 2: `short`, size $SIZE^3/512$ [S1, unofficial].
        *   Level 3: `short`, size $SIZE^3/4096$ [S1, unofficial].
        *   Level 4: `short`, size $SIZE^3/32768$ [S1, unofficial].
        *   Level 5+: `int`, size halving by 8 each level [S1, unofficial].
    *   **Top Level:** Contains 8 integers (sum[0] to sum[7]) representing the total triangle count [S1, unofficial].
2.  **Cube Indexes:** A buffer/image of size $SIZE^3$ storing the 8-bit cube index for each voxel. In buffer mode, this is a `char` buffer copied to a `CL_R, CL_UNSIGNED_INT8` image for traversal [S1, unofficial].
3.  **Intermediate Arrays (Buffer Path Only):**
    *   `voxelVerts`: uint array, size $numVoxels$ [S2, unofficial].
    *   `voxelOccupied`: uint array, size $numVoxels$ [S2, unofficial].
    *   `voxelOccupiedScan`: uint array, size $numVoxels$ (exclusive scan result) [S2, unofficial].
    *   `compactedVoxelArray`: uint array, size $numVoxels$ [S2, unofficial].

### Steps
1.  **Input Preparation:**
    *   Read raw volume data.
    *   If step sizes ($stepSizeX, stepSizeY, stepSizeZ$) > 1, subsample the volume [S1, unofficial].
    *   Calculate $largestSize = \max(N_x, N_y, N_z)$.
    *   Find smallest power of two $SIZE$ such that $2^i \ge largestSize$ [S1, unofficial].
    *   Allocate new volume of size $SIZE^3$ initialized to 0.
    *   Copy original data to new volume at indices $x + y \cdot SIZE + z \cdot SIZE^2$ for $0 \le x < N_x, 0 \le y < N_y, 0 \le z < N_z$ [S1, unofficial].

2.  **Voxel Classification:**
    *   Launch kernel `classifyCubesKernel` with work group $NDRange(SIZE, SIZE, SIZE)$ [S1, unofficial].
    *   For each voxel at grid position $(i, j, k)$:
        *   Compute field values at 8 corners: $field_0$ to $field_7$. If using 3D textures, use `computeValue4` to get $float4$ containing $(dx, dy, dz, value)$ [S2, unofficial].
        *   Calculate cube index $C$:
            $$C = \sum_{k=0}^{7} \mathbb{I}(field_k < isolevel) \cdot 2^k$$
            where $\mathbb{I}$ is the indicator function (1 if true, 0 if false) [S2, unofficial].
        *   Look up $numVerts = numVertsTable[C]$ [S1, S2, unofficial].
        *   Store $C$ in `cubeIndexesBuffer` (or `cubeIndexesImage`) [S1, S2, unofficial].
        *   (Buffer Path Only) Store $numVerts$ in `voxelVerts[i]`. If $numVerts > 0$, set `voxelOccupied[i] = 1`, else 0 [S2, unofficial].

3.  **Histogram Pyramid Construction:**
    *   **Base Level:**
        *   Launch kernel to compute base of HP.
        *   If 3D textures: `constructHPLevelKernel` reads `images[0]`, writes to `images[1]`. Work group $NDRange(SIZE/2, SIZE/2, SIZE/2)$ [S1, unofficial].
        *   If buffers: `constructHPLevelCharCharKernel` reads `buffers[0]`, writes to `buffers[1]`. Work group $NDRange(SIZE/2, SIZE/2, SIZE/2)$ [S1, unofficial].
    *   **Intermediate Levels:**
        *   Iterate from level $i=1$ to $\log_2(SIZE)-2$.
        *   If 3D textures: `constructHPLevelKernel` reads `images[i]`, writes to `images[i+1]`. Work group size halves each iteration [S1, unofficial].
        *   If buffers: Use type-specific kernels for transitions:
            *   Level 1→2: `constructHPLevelCharShortKernel` (char→short) [S1, unofficial].
            *   Level 2→3: `constructHPLevelShortShortKernel` (short→short) [S1, unofficial].
            *   Level 3→4: `constructHPLevelShortShortKernel` (short→short) [S1, unofficial].
            *   Level 4→5: `constructHPLevelShortIntKernel` (short→int) [S1, unofficial].
            *   Level 5+: `constructHPLevelKernel` (int→int) [S1, unofficial].
        *   Work group size is $previous/2$ where $previous$ starts at $SIZE/2$ and halves each iteration [S1, unofficial].

4.  **Triangle Count Retrieval:**
    *   Read the top of the HP (8 integers) from the last image/buffer [S1, unofficial].
    *   Sum these 8 integers to get $totalSum$.
    *   If $totalSum == 0$, no triangles were extracted; return early [S1, S2, unofficial].

5.  **VBO Allocation:**
    *   Create a new VBO with size $totalSum \times 18 \times 4$ bytes [S1, unofficial].

6.  **Voxel Compaction (Buffer Path Only):**
    *   Perform exclusive scan on `voxelOccupied` using `thrust::exclusive_scan` to produce `voxelOccupiedScan` [S2, unofficial].
    *   Launch `compactVoxels` kernel. For each thread $i$, if `voxelOccupied[i]` is true, write original index $i$ to `compactedVoxelArray[voxelOccupiedScan[i]]` [S2, unofficial].

7.  **Triangle Compaction (HP Traversal):**
    *   Launch `traverseHPKernel` to traverse the HP from top to bottom and fill the VBO [S1, S2, unofficial].
    *   Arguments: `rawData` (or skipped in texture mode), `cubeIndexesImage`, HP levels, VBO buffer, `isolevel`, $totalSum$ [S1, unofficial].
    *   Work group size: Next multiple of 64 greater than or equal to $totalSum$: $global\_work\_size = totalSum + 64 - (totalSum - 64 \cdot (totalSum / 64))$ [S1, unofficial].
    *   Local size: 64 [S1, unofficial].
    *   (Buffer Path Only) For each active voxel index $i$ in `compactedVoxelArray`:
        *   Compute corner positions $v_0$ to $v_7$ [S2, unofficial].
        *   Compute field values and gradients using `computeValue4` to get $field_0$ to $field_7$ ($float4$) [S2, unofficial].
        *   Calculate $C$ as in Step 2 [S2, unofficial].
        *   Compute 12 edge intersection points $vertlist[0]$ to $vertlist[11]$ using linear interpolation:
            $$t = \frac{isolevel - f_0.w}{f_1.w - f_0.w}, \quad p = p_0 + t \cdot (p_1 - p_0)$$
            where $f_0.w$ and $f_1.w$ are the scalar values at the two endpoints of the edge, and $p_0, p_1$ are their positions [S2, unofficial].
        *   Determine $numVerts$ from `numVertsTable` [S2, unofficial].
        *   For each triangle vertex $j$ in $0 \dots numVerts-1$:
            *   Get edge index $edge = triTable[C \cdot 16 + j]$ [S2, unofficial].
            *   Write $vertlist[edge]$ to `result[numVertsScanned[compactedVoxelArray[i]] + j]` [S2, unofficial].
    *   (3D Texture Path Only) The traversal kernel directly writes to the VBO using the HP structure to determine offsets, avoiding explicit compaction arrays [S1, unofficial].

8.  **Rendering:**
    *   Set up OpenGL matrices (projection, modelview) with translation, rotation, and scaling [S1, unofficial].
    *   Scaling factor: $scalingFactor = spacing \cdot 1.5 / SIZE$ [S1, unofficial].
    *   Bind VBO.
    *   Set vertex pointer (3 floats, stride 24, offset 0) and normal pointer (3 floats, stride 24, offset 12) [S1, unofficial].
    *   Draw triangles using `glDrawArrays(GL_TRIANGLES, 0, totalSum * 3)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_marching_cubes_histogram_pyramid(volume, size_x, size_y, size_z, isolevel=51):
    # Step 1: Input Preparation
    largest_size = max(size_x, size_y, size_z)
    size = 1
    while pow(2, size) < largest_size:
        size *= 2
    
    padded_volume = zeros((size, size, size), dtype=uint8)
    padded_volume[:size_x, :size_y, :size_z] = volume
    
    # Step 2: Voxel Classification
    cube_indexes = zeros((size, size, size), dtype=uint8)
    for k in range(size):
        for j in range(size):
            for i in range(size):
                # Get 8 corner values
                field_values = get_field_values(padded_volume, i, j, k, size)
                # Calculate cube index
                cube_index = 0
                for n in range(8):
                    if field_values[n] < isolevel:
                        cube_index += (1 << n)
                cube_indexes[i, j, k] = cube_index
    
    # Step 3: Histogram Pyramid Construction
    # Base level
    hp_levels = []
    current_level_data = zeros((size//2, size//2, size//2), dtype=uint8)
    # ... construct base level from cube_indexes ...
    hp_levels.append(current_level_data)
    
    # Intermediate levels
    current_size = size // 2
    for level in range(1, int(log2(size))):
        prev_data = hp_levels[-1]
        new_size = current_size // 2
        if level == 1:
            new_dtype = uint8
        elif level <= 4:
            new_dtype = uint16
        else:
            new_dtype = uint32
        
        new_level_data = zeros((new_size, new_size, new_size), dtype=new_dtype)
        # ... construct intermediate level by summing from prev_data ...
        hp_levels.append(new_level_data)
        current_size = new_size
    
    # Step 4: Triangle Count Retrieval
    top_level = hp_levels[-1]
    total_sum = sum(top_level)
    
    if total_sum == 0:
        return empty_vbo
    
    # Step 5: VBO Allocation
    vbo_size = total_sum * 18 * 4
    vbo = allocate_vbo(vbo_size)
    
    # Step 6 & 7: HP Traversal and VBO Writing
    # Launch traverseHPKernel with arguments:
    # rawData=padded_volume, cubeIndexes=cube_indexes, hp_levels=hp_levels, vbo=vbo, isolevel=isolevel, sum=total_sum
    # Work group size: next multiple of 64 >= total_sum
    # Local size: 64
    # Kernel logic:
    #   For each thread, traverse HP to find offset in VBO for this triangle
    #   Compute edge intersections and write to VBO
    
    # Step 8: Rendering
    # Setup OpenGL matrices, bind VBO, draw triangles
    return vbo
```

### Parameters
*   `isolevel`: Scalar threshold for surface extraction. Default 51 for 8-bit data [S1, unofficial].
*   `stepSizeX, stepSizeY, stepSizeZ`: Subsampling factors. Default 1 [S1, unofficial].
*   `SIZE`: Dimension of the cubic voxel grid, always a power of two. Passed to OpenCL kernel via `-DSIZE=<value>` [S1, unofficial].
*   `scalingFactor`: 3D vector for rendering scale. Calculated as $spacing \cdot 1.5 / SIZE$ [S1, unofficial].
*   `voxelSize`: Physical size of each voxel $(dx, dy, dz)$ [S2, unofficial].
*   `basePoint`: 3D origin of the voxel grid [S2, unofficial].

### Edge Cases and Pitfalls
*   **Power of Two Requirement:** Volume dimensions must be powers of two. If not, pad with zeros to the next power of two of the largest dimension [S1, S2, unofficial].
*   **NVIDIA GPU Limitation:** Lack of 3D texture write support on NVIDIA GPUs requires a fallback to regular buffers and morton codes, which is slower [S1, unofficial].
*   **Grid Dimension Limit:** CUDA grid dimensions are limited to 65535 in each dimension. If $numVoxels$ requires a larger grid, adjust launch configuration: $grid.x = 32768$, $grid.y = grid.x / 32768$ (capping x at 32768 and distributing remainder to y) [S2, unofficial].
*   **Empty Volume:** If no voxels are active ($totalSum == 0$), return early with empty VBO [S1, S2, unofficial].
*   **Image Size Constraint:** OpenGL/OpenCL images cannot be $1 \times 1 \times 1$. If buffer size becomes 1 during HP construction, force it to 2 [S1, unofficial].
*   **Boundary Checks:** Threads processing voxels at the grid boundary must verify that all 8 corner vertices exist within the volume bounds to avoid out-of-bounds memory access when sampling the scalar field [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3)$ for classification and HP construction, where $N = SIZE$. Traversal is $O(T)$ where $T$ is the number of triangles [S1, S2, unofficial].
*   **Memory Complexity:** $O(N^3)$ for the volume and cube indexes, plus $O(N^3)$ for the HP (with decreasing sizes per level) and $O(T)$ for the VBO [S1, S2, unofficial].
*   **Measured Performance:** Performance depends on GPU hardware and volume resolution. The 3D texture path is faster than the buffer path on supported GPUs [S1, unofficial].
*   **Scaling:** Scales with volume size and triangle count. The HP approach avoids atomic contention, making it efficient for high-density surfaces [S1, S2, unofficial].
*   **Strengths:** No atomic operations or sorting required; efficient memory usage via HP; supports dynamic scenes [S1, S2, unofficial].
*   **Limitations:** Requires power-of-two dimensions; fallback on NVIDIA GPUs is slower; complex implementation due to HP traversal [S1, S2, unofficial].
*   **When to Use:** When high-performance surface extraction is needed on GPUs with 3D texture write support, and atomic operations are a bottleneck [S1, S2, unofficial].
*   **When Not to Use:** On NVIDIA GPUs where the fallback is too slow, or when volume dimensions are not easily padded to powers of two [S1, S2, unofficial].

## Background
*   **Origin:** The method is based on "High-speed marching cubes using histopyramids" by Dyken, Ziegler, Theobalt, and Seidel (2008) [S2, unofficial].
*   **History:** Implemented in OpenCL and OpenGL by Erik Smistad, later moved to the FAST framework [S1, unofficial].
*   **Usage:** Used in real-time volume visualization and scientific computing applications requiring fast isosurface extraction [S1, S2, unofficial].

## Sources
[S1] smistad/GPU-Marching-Cubes, https://github.com/smistad/GPU-Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] AlbertLiDesign/ALG_MarchingCubes_GPU, https://github.com/AlbertLiDesign/ALG_MarchingCubes_GPU, fetched 2026-10-27 (unofficial)
