# Chunked Voxel Traversal

Also called: Chunk-based Voxel Traversal, Fixed-step voxel ray marching

## Overview
Chunked Voxel Traversal is a hierarchical ray traversal method for sparse voxel grids that accelerates intersection testing by organizing voxels into a 3D grid of smaller 3D sub-grids called "chunks" [S1, unofficial]. It solves the problem of inefficiently stepping through empty space in large voxel worlds by allowing the ray to skip entire chunks that contain no geometry [S1, unofficial]. It sets itself apart from pure 3D grid traversal by reducing the number of intersection checks in empty regions, and from octree-based traversal by using fixed-size inner grids which simplifies implementation and memory layout [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, outer grid occupancy data, and chunk data [S1, unofficial].
*   **Output:** The first non-empty voxel hit by the ray, or null if the ray exits the world without hitting a voxel [S1, unofficial].

**Data Structures:**
The structure is a two-level hierarchy:
1.  **Outer Grid:** A 3D array representing the world space, divided into chunks. Each entry stores a boolean or bitmask indicating if the chunk is empty [S1, unofficial].
2.  **Chunks:** Each chunk is a fixed-size 3D array of voxels [S1, unofficial].
    *   **Voxel Data:** Compressed to 8 bytes per voxel on the GPU to reduce transmission size; material properties are sent separately from voxel data [S1, unofficial].
    *   **Bit Packing:** Voxel attributes are packed into the 8 bytes. A critical implementation detail is that bitmasks must be applied **before** shifting data; applying them after results in zeroed-out data [S1, unofficial].

**Algorithm Steps:**
1.  **Outer Traversal:** Use the Digital Differential Analyzer (DDA) algorithm to traverse the outer 3D grid of chunks along the ray direction $\mathbf{d}$ [S1, unofficial].
2.  **Chunk Check:** For each chunk visited by the outer DDA, check its occupancy flag. If the chunk is marked as empty, skip it and continue to the next chunk in the outer DDA loop [S1, unofficial].
3.  **Inner Traversal:** If the chunk is non-empty, use the DDA algorithm to traverse the identified chunk [S1, unofficial].
4.  **Voxel Hit Test:** If a voxel is hit within the chunk, return that information to the outer DDA loop and terminate traversal [S1, unofficial]. If no voxel is hit in the chunk, continue iterating the outer DDA loop to check the next chunk [S1, unofficial].

**Transparency Handling:**
For transparent voxels, a fraction of the transparent object's color is added to the color behind it [S1, unofficial]. The traversal carries this transparent object information to the next voxel or chunk for final color determination [S1, unofficial].

**DDA Step Formula:**
The traversal advances along the axis with the smallest entry distance: $t_{next} = t_{current} + \Delta t$, where $\Delta t_i = 1/|d_i|$ for axis $i$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunked_voxel_traversal(ray_origin, ray_direction, outer_grid, chunks, chunk_size):
    # Initialize outer DDA
    step = [1 if ray_direction[i] > 0 else -1 for i in range(3)]
    t_max = [float('inf')] * 3
    t_delta = [abs(1.0 / ray_direction[i]) if ray_direction[i] != 0 else float('inf') for i in range(3)]
    current_chunk = [int(ray_origin[i] // chunk_size) for i in range(3)]
    for i in range(3):
        if ray_direction[i] != 0:
            t_max[i] = ((current_chunk[i] + (1 if step[i] > 0 else 0)) * chunk_size - ray_origin[i]) / ray_direction[i]
    
    while True:
        # Check if current chunk is empty
        if not outer_grid[current_chunk[0]][current_chunk[1]][current_chunk[2]]:
            # Advance outer DDA
            t_min = min(t_max)
            if t_min == float('inf'): break
            for i in range(3):
                if t_max[i] == t_min:
                    current_chunk[i] += step[i]
                    t_max[i] += t_delta[i]
            continue
        
        # Chunk is non-empty, traverse inside it
        chunk = chunks[current_chunk[0]][current_chunk[1]][current_chunk[2]]
        local_origin = [ray_origin[i] - current_chunk[i] * chunk_size for i in range(3)]
        
        # Inner DDA initialization
        inner_step = [1 if ray_direction[i] > 0 else -1 for i in range(3)]
        inner_t_max = [float('inf')] * 3
        inner_t_delta = [abs(1.0 / ray_direction[i]) if ray_direction[i] != 0 else float('inf') for i in range(3)]
        inner_voxel = [int(local_origin[i]) for i in range(3)]
        for i in range(3):
            if ray_direction[i] != 0:
                inner_t_max[i] = (inner_voxel[i] + (1 if inner_step[i] > 0 else 0) - local_origin[i]) / ray_direction[i]
        
        hit_voxel = None
        while True:
            if chunk.is_voxel_filled(inner_voxel[0], inner_voxel[1], inner_voxel[2]):
                hit_voxel = chunk.get_voxel_data(inner_voxel[0], inner_voxel[1], inner_voxel[2])
                break
            t_min = min(inner_t_max)
            if t_min == float('inf'): break
            for i in range(3):
                if inner_t_max[i] == t_min:
                    inner_voxel[i] += inner_step[i]
                    inner_t_max[i] += inner_t_delta[i]
        
        if hit_voxel is not None: return hit_voxel
        
        # No hit in this chunk, advance outer grid
        t_min = min(t_max)
        if t_min == float('inf'): break
        for i in range(3):
            if t_max[i] == t_min:
                current_chunk[i] += step[i]
                t_max[i] += t_delta[i]
    return None
```

**Parameters:**
*   **Chunk Dimensions:** 8x8x8 voxels per chunk [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bitmask Order:** Applying bitmasks after shifting data results in zeroed-out data. Bitmasks must be applied before shifting [S1, unofficial].
*   **Padding:** Compressing voxel data to 8 bytes may require padding variables to maintain data alignment on the GPU [S1, unofficial].
*   **Empty Chunks:** The algorithm relies on accurate occupancy flags for chunks; incorrect flags lead to skipped geometry or wasted traversal steps [S1, unofficial].

## Subtopics

### [Voxel Path Tracing with Chunked Storage](9384-voxel-path-tracing-with-chunked-storage.md)
A per-voxel lighting method that decouples illumination calculation from the rendering pipeline to enable dynamic streaming and instant map edits. Unlike standard ray-traced voxel rendering which computes lighting per-pixel per-frame, this method accumulates diffuse lighting over multiple frames while calculating direct and specular components via fixed uniform sampling, allowing the scene to remain interactive during edits. It builds on [Chunked Voxel Traversal](8311-chunked-voxel-traversal.md) by organizing voxels into 8x8x8 chunks to facilitate efficient skipping of empty space and targeted updates.

## Operating characteristics
*   **Time Complexity:** Depends on the sparsity of the voxel grid; skips empty chunks to reduce intersection tests, behaving like standard DDA in dense regions [S1, unofficial].
*   **Memory Complexity:** Requires storage for outer grid occupancy flags and chunk data, with voxels compressed to 8 bytes per voxel [S1, unofficial].
*   **Strengths:** Efficient skipping of empty space; fixed-size chunks simplify memory layout [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.

## Background
The method was described in an unofficial blog post by a developer working on a voxel raytracer [S1, unofficial]. The author sought a middle ground between the simplicity of a 3D grid and the efficiency of octrees, leading to the "chunked" approach [S1, unofficial]. The implementation details, including the 8x8x8 chunk size and 8-byte voxel compression, were derived from practical experience in developing a real-time voxel rendering engine [S1, unofficial].

## Sources
[S1] Chunks and color | Voxel Raytracing, https://ii-day-ii.github.io/voxel_raytracer/2023/05/18/chunks-and-acceleration.html, fetched 2026-10-24 (unofficial)
