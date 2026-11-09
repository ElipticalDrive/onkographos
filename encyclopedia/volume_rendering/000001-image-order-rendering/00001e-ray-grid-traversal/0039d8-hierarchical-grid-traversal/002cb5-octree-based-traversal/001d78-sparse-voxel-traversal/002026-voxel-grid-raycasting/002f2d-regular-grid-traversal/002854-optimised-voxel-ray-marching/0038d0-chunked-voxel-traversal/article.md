# Chunked Voxel Traversal

Also called: Chunk-based Voxel Traversal, Fixed-step voxel ray marching

## Overview
Chunked Voxel Traversal is a hierarchical ray traversal method for sparse voxel grids that accelerates intersection testing by organizing voxels into a 3D grid of smaller 3D sub-grids called "chunks" [S1, unofficial]. It solves the problem of inefficiently stepping through empty space in large voxel worlds by allowing the ray to skip entire chunks that contain no geometry [S1, unofficial]. It sets itself apart from pure 3D grid traversal by reducing the number of intersection checks in empty regions, and from octree-based traversal by using fixed-size inner grids which simplifies implementation and memory layout [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, world-space voxel grid dimensions, chunk dimensions ($N_c \times N_c \times N_c$), and chunk occupancy data.
*   **Output:** The first non-empty voxel hit by the ray, or null if the ray exits the world without hitting a voxel.

**Data Structures:**
The structure is a two-level hierarchy:
1.  **Outer Grid:** A 3D array representing the world space, divided into chunks. Each entry stores a boolean or bitmask indicating if the chunk is empty.
2.  **Chunks:** Each chunk is a fixed-size 3D array of voxels.
    *   **Voxel Data:** Compressed to 8 bytes per voxel on the GPU to reduce transmission size [S1, unofficial]. Material properties are stored separately from voxel data [S1, unofficial].
    *   **Bit Packing:** Voxel attributes (e.g., visibility, normals) are packed into the 8 bytes. A critical implementation detail is that bitmasks must be applied **before** shifting data; applying them after results in zeroed-out data [S1, unofficial].

**Algorithm Steps:**
1.  **Outer Traversal:** Use the Digital Differential Analyzer (DDA) algorithm to traverse the outer 3D grid of chunks along the ray $\mathbf{d}$ [S1, unofficial].
2.  **Chunk Check:** For each chunk visited by the outer DDA:
    *   If the chunk is marked as empty, skip it and continue to the next chunk in the outer DDA loop [S1, unofficial].
    *   If the chunk is non-empty, proceed to Step 3.
3.  **Inner Traversal:** Use the DDA algorithm to traverse the identified non-empty chunk [S1, unofficial].
4.  **Voxel Hit Test:**
    *   If a voxel is hit within the chunk, return the voxel's information (position, normal, material) to the caller and terminate the traversal [S1, unofficial].
    *   If no voxel is hit in the chunk (e.g., the ray passes through empty space within the chunk), continue the outer DDA loop to check the next chunk [S1, unofficial].

**Transparency Handling:**
For transparent voxels, the traversal accumulates color information. A fraction of the transparent object's color is added to the color behind it. The traversal carries this transparent object information to the next voxel or chunk for final color determination [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunked_voxel_traversal(ray_origin, ray_direction, outer_grid, chunks, chunk_size):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    outer_grid: 3D array of booleans (chunk occupancy)
    chunks: 3D array of chunk objects
    chunk_size: int (e.g., 8)
    """
    # Initialize DDA for outer grid
    # Calculate step size and initial entry point for outer grid
    outer_step, outer_t_max, outer_current_chunk = init_dda(ray_origin, ray_direction, outer_grid.dimensions)
    
    while outer_t_max > 0:
        # Check if current chunk is empty
        if not outer_grid[outer_current_chunk]:
            # Skip empty chunk
            advance_dda(outer_step, outer_t_max, outer_current_chunk)
            continue
            
        # Chunk is non-empty, traverse inside it
        chunk = chunks[outer_current_chunk]
        
        # Initialize DDA for inner chunk
        # Adjust ray origin/direction relative to chunk local space if necessary
        inner_step, inner_t_max, inner_current_voxel = init_dda(ray_origin, ray_direction, chunk.dimensions)
        
        hit_voxel = None
        while inner_t_max > 0:
            # Check voxel occupancy
            if chunk.is_voxel_filled(inner_current_voxel):
                hit_voxel = chunk.get_voxel_data(inner_current_voxel)
                break
            advance_dda(inner_step, inner_t_max, inner_current_voxel)
            
        if hit_voxel is not None:
            return hit_voxel
            
        # No hit in this chunk, advance outer grid
        advance_dda(outer_step, outer_t_max, outer_current_chunk)
        
    return None
```

**Parameters:**
*   **Chunk Dimensions:** 8x8x8 voxels per chunk [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bitmask Order:** Applying bitmasks after shifting data results in zeroed-out data. Bitmasks must be applied before shifting [S1, unofficial].
*   **Padding:** When compressing voxel data to 8 bytes, padding variables may be required to maintain data alignment on the GPU [S1, unofficial].
*   **Empty Chunks:** The algorithm relies on accurate occupancy flags for chunks. Incorrect flags lead to skipped geometry or wasted traversal steps.

## Subtopics

### [Voxel Path Tracing with Chunked Storage](9384-voxel-path-tracing-with-chunked-storage.md)
A per-voxel lighting method that decouples illumination from the rendering pipeline, using 8x8x8 chunks to facilitate efficient skipping of empty space and targeted updates.

## Operating characteristics
*   **Time Complexity:** Depends on the sparsity of the voxel grid. In dense regions, it behaves like standard DDA. In sparse regions, it skips empty chunks, reducing the number of intersection tests.
*   **Memory Complexity:** Requires storage for the outer grid occupancy flags and the chunk data. Voxel data is compressed to 8 bytes per voxel [S1, unofficial].
*   **Strengths:** Simple implementation compared to octrees; efficient skipping of empty space; fixed-size chunks simplify memory management and bit-packing [S1, unofficial].
*   **Limitations:** Fixed chunk size may lead to fragmentation or inefficient storage if object sizes vary significantly; requires careful handling of data alignment and bit-packing on the GPU [S1, unofficial].
*   **When to Use:** Sparse voxel worlds with relatively uniform object sizes; when implementation simplicity is prioritized over optimal memory usage for highly irregular geometries.

## Background
The method was described in an unofficial blog post by a developer working on a voxel raytracer [S1, unofficial]. The author sought a middle ground between the simplicity of a 3D grid and the efficiency of octrees, leading to the "chunked" approach [S1, unofficial]. The implementation details, including the 8x8x8 chunk size and 8-byte voxel compression, were derived from practical experience in developing a real-time voxel rendering engine [S1, unofficial].

## Sources
[S1] Chunks and color | Voxel Raytracing, https://ii-day-ii.github.io/voxel_raytracer/2023/05/18/chunks-and-acceleration.html, fetched 2026-10-24 (unofficial)
