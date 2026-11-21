# Hash Map Based Marching Cubes

Also called: Hash map based approach

## Overview
Hash Map Based Marching Cubes is a surface extraction method that employs surface-following traversal to extract isosurfaces, utilizing hash maps to store corner vertices and a 3D lookup cache for particle identification [S1]. It replaces static grid storage with dynamic hash maps to handle sparse or unbounded datasets efficiently [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset (e.g., particle positions or scalar field values) and an isovalue threshold [S1].
*   **Output:** A mesh (vertices and indices) representing the extracted isosurface [S1].

**Data Structures**
1.  **Corner Cache:** A hash map mapping 3D grid coordinates $(x, y, z)$ to vertex data (position, scalar value) [S1].
2.  **Particle Lookup Cache:** A 3D hash map used to identify which particles fall within a specific voxel or cube during the marching process [S1].

**Algorithm Steps**
1.  **Initialization:** Initialize the corner cache and particle lookup cache as empty hash maps [S1].
2.  **Surface Following:** Traverse cubes intersecting the isosurface without scanning the entire volume [S1].
3.  **Cube Processing:** For each active cube:
    a.  **Particle Lookup:** Query the 3D lookup cache to find particles within the cube's bounds [S1].
    b.  **Scalar Evaluation:** Compute the scalar field value at the cube's corners based on the interpolated particle data [G].
    c.  **Corner Storage:** Store or retrieve corner vertices in the corner cache hash map using their grid coordinates as keys [S1].
    d.  **Marching Cubes Lookup:** Determine the cube case (one of the 256 configurations) based on corner signs [G].
    e.  **Triangle Generation:** Generate triangles for the intersected edges of the cube [G].
4.  **Cache Management:** Build and destroy the particle cache on a per-block basis to remove unneeded particles [S1].

**Formulas**
*   **Lookup Volume Efficiency:** The method avoids searching a cubic volume $(3R_c)^3$ and culling by distance, which returns significantly more particles than a spherical volume $(4/3)\pi R_c^3$ [S1].
*   **Isosurface Condition:** The surface is defined where the scalar field $f(x,y,z) = 0$ [S2, unofficial].

**Parameters**
*   **Lookup Grid Resolution:** The resolution of the 3D lookup cache [S1].

**Edge Cases and Pitfalls**
*   **Cost Dominance:** If the lookup grid resolution is increased to reduce false positives, the insertion and lookup costs grow rapidly and dominate overall extraction time, particularly because every cube access requires a hash map lookup [S1].
*   **Memory Management:** Memory usage remains comparable to previous approaches because the particle cache is built and destroyed on a per-block basis [S1].

## Subtopics

### Spatially Hashed Marching Cubes
This variant uses spatially hashed 3D chunks of volumetric data to handle large distances without exhausting memory, similar to techniques used in games like Minecraft [S2, unofficial]. It avoids octrees for implicit surface representation because octrees are inefficient for the iteration-dominated marching cubes algorithm, recommending fixed 3D arrays or chunks instead [S2, unofficial]. Dual Marching Cubes can be applied here, scanning from an origin point to locate volumetric edges and place edging/skin [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by hash map lookups for corner storage and particle identification [S1]. Performance degrades if the lookup grid resolution is set too high, as lookup costs dominate [S1].
*   **Memory Complexity:** Memory usage is kept low by building and destroying the particle cache on a per-block basis [S1].
*   **Strengths:** Efficient for unbounded or sparse rendering; avoids the iteration overhead of dense arrays and the query latency of octrees [S1][S2, unofficial].
*   **Limitations:** Hash map overhead increases with high-resolution lookups; iteration speed is prioritized over memory savings in dense scenes [S2, unofficial].
*   **When to Use:** Sparse particle simulations or unbounded volumes requiring memory efficiency and selective processing [S1].
*   **When Not to Use:** Scenes requiring maximum iteration speed where memory constraints are absent, as fixed 3D arrays outperform hash maps for dense iteration [S2, unofficial].

## Background
The method is based on research into real-time particle isosurface extraction, adapting surface-following algorithms to use hash maps for dynamic particle data handling [S1]. It contrasts with traditional grid-based and octree-based methods, which suffer from iteration or query inefficiencies respectively [S2, unofficial].

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
[S2] Implicit Functions and extracting an isosurface, https://gamedev.stackexchange.com/questions/85873/implicit-functions-and-extracting-an-isosurface, fetched 2026-10-26 (unofficial)
