# Terrain Chunk Update

Also called: Dynamic Terrain Update, Vertex Shift and Regeneration

## Overview
Updates a terrain grid when the camera moves by shifting existing vertex data and regenerating only the newly exposed rows/columns [S1, unofficial]. It differs from full regeneration by reusing valid vertex data, reducing computational cost for small camera displacements [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: `terrain_vertices` array (flat list of vertex positions), `num_chunks` (integer offset indicating how many grid steps the camera moved in x and z directions) [S1, unofficial].
- Output: Updated `terrain_vertices` array with shifted and regenerated data [S1, unofficial].

**Data structures:**
- `terrain_vertices`: A flat array of vertex positions. The layout is row-major, where each row has `TERRAIN_NUM_VERTICES_SIDE` vertices [S1, unofficial].
- `num_chunks`: A 2D integer vector `(num_chunks.x, num_chunks.z)` representing the camera's movement in chunk units [S1, unofficial].

**Steps:**
1. Calculate shift direction `sign` and offset `diff_shift` based on `num_chunks.x` and `num_chunks.z` [S1, unofficial].
2. Determine start/end indices for shifting columns (z-axis) and rows (x-axis) based on `sign` and magnitude [S1, unofficial].
3. Shift columns: loop `j` from `start.z` to `end.z`, copy `terrain_vertices[pos_current]` to `terrain_vertices[pos_current + TERRAIN_NUM_VERTICES_SIDE * num_chunks.z]` [S1, unofficial].
4. Shift rows: loop `j` from 0 to `TERRAIN_NUM_VERTICES_SIDE`, loop `i` from `start.x` to `end.x`, copy `terrain_vertices[pos_current]` to `terrain_vertices[pos_current + num_chunks.x]` [S1, unofficial].
5. Regenerate vertices on the newly exposed z-edges using `fill_terrain_vertices` [S1, unofficial].
6. Regenerate vertices on the newly exposed x-edges using `fill_terrain_vertices` [S1, unofficial].
7. Update normals for the regenerated regions by calling `fill_terrain_normals` on the new z and x strips [S1, unofficial].

**Formulas:**
- `sign.x = glm_sign(num_chunks.x)`, `sign.z = glm_sign(num_chunks.z)` [S1, unofficial].
- `diff_shift.x = TERRAIN_NUM_VERTICES_SIDE - abs(num_chunks.x)`, `diff_shift.z = TERRAIN_NUM_VERTICES_SIDE - abs(num_chunks.z)` [S1, unofficial].
- `start.x = (num_chunks.x >= 0) ? (diff_shift.x - 1) : abs(num_chunks.x)` [S1, unofficial].
- `end.x = start.x - (diff_shift.x * sign.x)` [S1, unofficial].
- `start.z = (num_chunks.z >= 0) ? (diff_shift.z - 1) : abs(num_chunks.z)` [S1, unofficial].
- `end.z = start.z - (diff_shift.z * sign.z)` [S1, unofficial].

**Parameters:**
- `num_chunks`: Integer offset indicating how many grid steps the camera moved in x and z directions [S1, unofficial].
- `TERRAIN_NUM_VERTICES_SIDE`: Number of vertices along one side of the terrain grid [S1, unofficial].

**Edge cases and pitfalls:**
- When `num_chunks` is zero, no shift occurs [S1, unofficial].
- When `num_chunks` is large, the shift may exceed the grid boundaries, requiring careful handling of indices [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def terrain_chunk_update(terrain_vertices, num_chunks, TERRAIN_NUM_VERTICES_SIDE):
    # Calculate shift direction and offset
    sign_x = 1 if num_chunks.x > 0 else (-1 if num_chunks.x < 0 else 0)
    sign_z = 1 if num_chunks.z > 0 else (-1 if num_chunks.z < 0 else 0)
    
    diff_shift_x = TERRAIN_NUM_VERTICES_SIDE - abs(num_chunks.x)
    diff_shift_z = TERRAIN_NUM_VERTICES_SIDE - abs(num_chunks.z)
    
    # Determine start/end indices for shifting columns (z-axis) and rows (x-axis)
    start_x = (diff_shift_x - 1) if num_chunks.x >= 0 else abs(num_chunks.x)
    end_x = start_x - (diff_shift_x * sign_x)
    
    start_z = (diff_shift_z - 1) if num_chunks.z >= 0 else abs(num_chunks.z)
    end_z = start_z - (diff_shift_z * sign_z)
    
    # Shift columns
    for j in range(start_z, end_z, -sign_z):
        for i in range(TERRAIN_NUM_VERTICES_SIDE):
            pos_current = (j * TERRAIN_NUM_VERTICES_SIDE) + i
            pos_new = pos_current + (TERRAIN_NUM_VERTICES_SIDE * num_chunks.z)
            terrain_vertices[pos_new] = terrain_vertices[pos_current]
            
    # Shift rows
    for j in range(TERRAIN_NUM_VERTICES_SIDE):
        for i in range(start_x, end_x, -sign_x):
            pos_current = (j * TERRAIN_NUM_VERTICES_SIDE) + i
            pos_new = pos_current + num_chunks.x
            terrain_vertices[pos_new] = terrain_vertices[pos_current]
            
    # Regenerate vertices on the newly exposed z-edges
    fill_terrain_vertices(terrain_vertices, start_z, end_z, TERRAIN_NUM_VERTICES_SIDE)
    
    # Regenerate vertices on the newly exposed x-edges
    fill_terrain_vertices(terrain_vertices, start_x, end_x, TERRAIN_NUM_VERTICES_SIDE)
    
    # Update normals for the regenerated regions
    fill_terrain_normals(terrain_vertices, start_z, end_z, TERRAIN_NUM_VERTICES_SIDE)
    fill_terrain_normals(terrain_vertices, start_x, end_x, TERRAIN_NUM_VERTICES_SIDE)
```

## Subtopics

## Operating characteristics
- **Time complexity:** O(N) where N is the number of vertices in the newly exposed rows/columns [S1, unofficial].
- **Memory complexity:** O(1) additional memory, as the update is done in-place [S1, unofficial].
- **Measured performance:** Not given in the sources [S1, unofficial].
- **Scaling:** Scales linearly with the number of vertices in the newly exposed rows/columns [S1, unofficial].
- **Strengths:** Efficient for small camera displacements, reduces computational cost by reusing valid vertex data [S1, unofficial].
- **Limitations:** Not efficient for large camera displacements, as the shift may exceed grid boundaries [S1, unofficial].
- **When to use:** When the camera moves small distances and the terrain grid is large [S1, unofficial].
- **When not to use:** When the camera moves large distances or the terrain grid is small [S1, unofficial].

## Background
Origin: Not given in the sources [S1, unofficial].
History and development: Not given in the sources [S1, unofficial].
Where it is used: Not given in the sources [S1, unofficial].

## Sources
[S1] vittoboa/procedural-terrain-generation, https://github.com/vittoboa/procedural-terrain-generation, fetched 2026-10-27 (unofficial)
