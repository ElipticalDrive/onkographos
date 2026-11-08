# Manifold Dual Marching Cubes

Also called: Dual Marching Cubes (DMC), DualMC problematic configurations, DualMCComputeFramework, Manifold DualMC table generation, Primal Contouring of Dual Grids

## Overview
Manifold Dual Marching Cubes (MDMC) is a surface extraction algorithm that generates a topologically correct, manifold mesh from a scalar field by operating on a dual grid [S2][S5, unofficial]. Unlike standard Dual Marching Cubes, which may produce non-manifold artifacts in ambiguous configurations (specifically cases C16 and C19), MDMC resolves these ambiguities by inspecting neighboring cells and inverting the cell configuration when necessary to ensure consistent connectivity [S5, unofficial]. It builds on [Dual Mesh Building](1423-dual-mesh-building.md) by ensuring the dual vertices are placed such that the resulting quad strips form a valid 2-manifold without self-intersections or holes [S5, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field (volume data) stored in a grid, an isosurface threshold value ($iso$), and a lookup table for ambiguous configurations.
- **Output:** An indexed triangle mesh (vertices and normals).

**Data Structures:**
1.  **Volume Data:** Stored in a packed array `VolumeData.Data` of `uint32_t`. Each `uint32_t` holds two 16-bit unsigned short values. The size is $64^3 = 262,144$ elements, supporting 6-bit Morton encoded indices [S5, unofficial].
2.  **Dual Points List (`DualPointsList`):** A compact array of 512 `uint32_t` values (1024 `uint16_t` shorts). It maps each of the 256 cube configurations to 4 dual point codes (2 per face), packed as 2 shorts per `uint32_t` [S5, unofficial].
3.  **Problematic Configs (`ProblematicConfigs`):** A compact array of 64 `uint32_t` values (256 `uint8_t` bytes). Each byte stores the ambiguous face direction (0-5) or 255 if non-problematic. Four bytes are packed into one `uint32_t` [S5, unofficial].
4.  **Dual Point Map (`DualPointMap`):** A hash map (array) of size $64^3$ (space for every possible Morton ID) storing computed dual point positions. IDs are masked to 18 bits, using linear probing for collision resolution [S5, unofficial].
5.  **Triangle Output:** An array of `Triangle` structs, each containing three `glm::vec3` vertices and one `glm::vec3` normal [S5, unofficial].

**Corner Bitmasks:**
The 8 corners of a cube are indexed 0-7. The configuration value is the sum of active corner bits:
- $C_0=1, C_1=2, C_2=4, C_3=8, C_4=16, C_5=32, C_6=64, C_7=128$ [S3, unofficial].

**Edge Codes:**
12 edges are defined as powers of two:
- $EDGE_0=1, EDGE_1=2, \dots, EDGE_{11}=2048$ [S3, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    -   Pack `DualPointsList` and `ProblematicConfigs` from precomputed tables [S5, unofficial].
    -   Initialize `DualPointMap` with empty markers (e.g., `~0ull`) [S5, unofficial].
    -   Initialize `TriangleCounter` to 0 [S5, unofficial].

2.  **Main Loop:**
    -   Iterate over voxel coordinates $x, y, z$ from 0 to 35 (for a $36 \times 36 \times 36$ grid) [S5, unofficial].
    -   Early-out if $x \ge 34$, $y \ge 34$, or $z \ge 34$ (boundary checks for neighbor access) [S5, unofficial].

3.  **Cell Code Calculation:**
    -   For the current cell position, sample the 8 corner vertices using Morton encoding [S5, unofficial].
    -   Vertex order: $(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)$ [S5, unofficial].
    -   Bit $i$ is set if $Value_i \ge iso$. This forms the 8-bit `cellCode` [S5, unofficial].

4.  **Ambiguity Resolution (Manifold Check):**
    -   Retrieve the direction byte from `ProblematicConfigs` using `cellCode` [S5, unofficial].
    -   If `direction != 255`, the configuration is ambiguous (C16 or C19) [S5, unofficial].
    -   Decode direction: `component = direction >> 1` (0=X, 1=Y, 2=Z), `delta = (direction & 1) ? 1 : -1` [S5, unofficial].
    -   Calculate neighbor coordinates: `neighbourCoords = cellPos + (delta * axis_vector)` [S5, unofficial].
    -   Check if neighbor is within valid bounds ($0 \le coord < REDUCED - 1$) [S5, unofficial].
    -   If within bounds, get `neighbourCellCode`. If `neighbourCellCode` is also problematic (`direction != 255`), invert the current `cellCode`: `cellCode ^= 0xFF` [S5, unofficial].

5.  **Dual Point Code Lookup:**
    -   For each intersected edge, determine the dual point code by iterating $i$ from 0 to 3 in `DualPointsList` for the resolved `cellCode` [S5, unofficial].
    -   If `(entry & edgeCode) != 0`, return `entry` as the dual point ID [S5, unofficial].

6.  **Edge Processing and Triangle Generation:**
    -   Process X, Y, and Z edges separately. For each edge, check if it intersects the isosurface (one endpoint $< iso$, other $\ge iso$) [S5, unofficial].
    -   **X-Edge (between $xyz$ and $xyz+(1,0,0)$):**
        -   Requires $z>0$ and $y>0$ [S5, unofficial].
        -   Vertices: $vert_0$ at $xyz$ (EDGE0), $vert_1$ at $xyzZ_1$ (EDGE2), $vert_2$ at $xyzY_1Z_1$ (EDGE6), $vert_3$ at $xyzY_1$ (EDGE4) [S5, unofficial].
        -   If entering ($val_{xyz} < iso \le val_{xyz+(1,0,0)}$): Output triangles $(vert_0, vert_1, vert_2)$ and $(vert_2, vert_3, vert_0)$ [S5, unofficial].
        -   If exiting: Output triangles $(vert_2, vert_1, vert_0)$ and $(vert_0, vert_3, vert_2)$ [S5, unofficial].
    -   **Y-Edge (between $xyz$ and $xyz+(0,1,0)$):**
        -   Requires $z>0$ and $x>0$ [S5, unofficial].
        -   Vertices: $vert_0$ at $xyz$ (EDGE8), $vert_1$ at $xyzZ_1$ (EDGE11), $vert_2$ at $xyzX_1Z_1$ (EDGE10), $vert_3$ at $xyzX_1$ (EDGE9) [S5, unofficial].
        -   If exiting: Output $(vert_0, vert_1, vert_2)$ and $(vert_2, vert_3, vert_0)$ [S5, unofficial].
        -   If entering: Output $(vert_2, vert_1, vert_0)$ and $(vert_0, vert_3, vert_2)$ [S5, unofficial].
    -   **Z-Edge (between $xyz$ and $xyz+(0,0,1)$):**
        -   Requires $x>0$ and $y>0$ [S5, unofficial].
        -   Vertices: $vert_0$ at $xyz$ (EDGE3), $vert_1$ at $xyzX_1$ (EDGE1), $vert_2$ at $xyzX_1Y_1$ (EDGE5), $vert_3$ at $xyzY_1$ (EDGE7) [S5, unofficial].
        -   If exiting: Output $(vert_0, vert_1, vert_2)$ and $(vert_2, vert_3, vert_0)$ [S5, unofficial].
        -   If entering: Output $(vert_2, vert_1, vert_0)$ and $(vert_0, vert_3, vert_2)$ [S5, unofficial].
    -   Reserve triangle indices using atomic addition [S5, unofficial].
    -   Calculate face normals using the cross product of edge vectors [S5, unofficial].

7.  **Dual Point Computation (if not cached):**
    -   If the dual point ID is not found in `DualPointMap`, compute it [S5, unofficial].
    -   Initialize $p=0$, $points=0$ [S5, unofficial].
    -   For each of the 12 edges, if intersected, add the linearly interpolated intersection coordinate to $p$ and increment $points$ [S5, unofficial].
    -   Interpolation for EDGE0 (x-axis): $x = (iso - val_0) / (val_1 - val_0)$ [S5, unofficial].
    -   Average $p$ by $points$ and offset by voxel coordinates [S5, unofficial].
    -   Store in `DualPointMap` using linear probing [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def manifold_dual_marching_cubes(volume_data, iso, problematic_configs, dual_points_list):
    triangle_output = []
    dual_point_map = {} # Hash map for caching dual points
    triangle_counter = 0
    
    # Constants
    REDUCED = 34
    VOLUME_SIZE = 64
    
    def morton3d(v):
        v = v & 63
        v = (v | (v << 8)) & 0x300F
        v = (v | (v << 4)) & 0x30C3
        v = (v | (v << 2)) & 0x9249
        return (v.x) | ((v.y) << 1) | ((v.z) << 2)

    def get_data(morton_code):
        data_idx = morton_code // 2
        offset = 16 * (morton_code % 2)
        return (volume_data[data_idx] >> offset) & 0xFFFF

    def get_problematic_config(byte_index):
        uint_index = byte_index // 4
        bit_offset = (byte_index % 4) * 8
        return (problematic_configs[uint_index] >> bit_offset) & 0xFF

    def get_dual_point_entry(cell_code, index):
        idx = (cell_code * 2) + (index // 2)
        packed = dual_points_list[idx]
        shift = (index % 2) * 16
        return (packed >> shift) & 0xFFFF

    def calculate_dual_point(cell_pos, edge_code):
        morton = morton3d(cell_pos)
        if morton in dual_point_map:
            return dual_point_map[morton]
        
        p = [0.0, 0.0, 0.0]
        points = 0
        
        # Simplified: In practice, iterate 12 edges, check intersection, interpolate
        # This is a placeholder for the specific interpolation logic per edge
        # See source [S5, unofficial] for full 12-edge interpolation logic
        # For brevity, assuming a generic interpolation function exists
        # p = interpolate_edges(cell_pos, edge_code, iso)
        
        if points > 0:
            p = [x / points for x in p]
            p = [p[i] + cell_pos[i] for i in range(3)]
        
        dual_point_map[morton] = p
        return p

    def calculate_normal(v0, v1, v2):
        p01 = [v1[i] - v0[i] for i in range(3)]
        p02 = [v2[i] - v0[i] for i in range(3)]
        # Cross product
        cx = p01[1]*p02[2] - p01[2]*p02[1]
        cy = p01[2]*p02[0] - p01[0]*p02[2]
        cz = p01[0]*p02[1] - p01[1]*p02[0]
        # Normalize
        length = (cx**2 + cy**2 + cz**2)**0.5
        if length == 0: return [0,0,0]
        return [-cx/length, -cy/length, -cz/length]

    for z in range(36):
        for y in range(36):
            for x in range(36):
                if x >= REDUCED or y >= REDUCED or z >= REDUCED:
                    continue
                
                xyz = (x, y, z)
                
                # Get cell code
                cell_code = 0
                for i, offset in enumerate([(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)]):
                    pos = (x+offset[0], y+offset[1], z+offset[2])
                    val = get_data(morton3d(pos))
                    if val >= iso:
                        cell_code |= (1 << i)
                
                # Ambiguity resolution
                direction = get_problematic_config(cell_code)
                if direction != 255:
                    component = direction >> 1
                    delta = 1 if (direction & 1) else -1
                    neighbour_coords = list(xyz)
                    if component == 0: neighbour_coords[0] += delta
                    elif component == 1: neighbour_coords[1] += delta
                    else: neighbour_coords[2] += delta
                    
                    if 0 <= neighbour_coords[0] < REDUCED-1 and \
                       0 <= neighbour_coords[1] < REDUCED-1 and \
                       0 <= neighbour_coords[2] < REDUCED-1:
                        n_cell_code = 0
                        for i, offset in enumerate([(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)]):
                            pos = (neighbour_coords[0]+offset[0], neighbour_coords[1]+offset[1], neighbour_coords[2]+offset[2])
                            val = get_data(morton3d(pos))
                            if val >= iso:
                                n_cell_code |= (1 << i)
                        if get_problematic_config(n_cell_code) != 255:
                            cell_code ^= 0xFF
                
                # Process X-Edge
                if z > 0 and y > 0:
                    val_curr = get_data(morton3d(xyz))
                    val_next = get_data(morton3d((x+1, y, z)))
                    entering = val_curr < iso and val_next >= iso
                    exiting = val_curr >= iso and val_next < iso
                    
                    if entering or exiting:
                        v0 = calculate_dual_point(xyz, 1) # EDGE0
                        v1 = calculate_dual_point((x, y, z-1), 4) # EDGE2
                        v2 = calculate_dual_point((x, y-1, z-1), 64) # EDGE6
                        v3 = calculate_dual_point((x, y-1, z), 16) # EDGE4
                        
                        idx = triangle_counter
                        triangle_counter += 2
                        
                        if entering:
                            n1 = calculate_normal(v0, v1, v2)
                            n2 = calculate_normal(v2, v3, v0)
                            triangle_output.append((v0, v1, v2, n1))
                            triangle_output.append((v2, v3, v0, n2))
                        else:
                            n1 = calculate_normal(v2, v1, v0)
                            n2 = calculate_normal(v0, v3, v2)
                            triangle_output.append((v2, v1, v0, n1))
                            triangle_output.append((v0, v3, v2, n2))

                # Process Y-Edge
                if z > 0 and x > 0:
                    val_curr = get_data(morton3d(xyz))
                    val_next = get_data(morton3d((x, y+1, z)))
                    entering = val_curr < iso and val_next >= iso
                    exiting = val_curr >= iso and val_next < iso
                    
                    if entering or exiting:
                        v0 = calculate_dual_point(xyz, 256) # EDGE8
                        v1 = calculate_dual_point((x, y, z-1), 2048) # EDGE11
                        v2 = calculate_dual_point((x-1, y, z-1), 1024) # EDGE10
                        v3 = calculate_dual_point((x-1, y, z), 512) # EDGE9
                        
                        idx = triangle_counter
                        triangle_counter += 2
                        
                        if exiting:
                            n1 = calculate_normal(v0, v1, v2)
                            n2 = calculate_normal(v2, v3, v0)
                            triangle_output.append((v0, v1, v2, n1))
                            triangle_output.append((v2, v3, v0, n2))
                        else:
                            n1 = calculate_normal(v2, v1, v0)
                            n2 = calculate_normal(v0, v3, v2)
                            triangle_output.append((v2, v1, v0, n1))
                            triangle_output.append((v0, v3, v2, n2))

                # Process Z-Edge
                if x > 0 and y > 0:
                    val_curr = get_data(morton3d(xyz))
                    val_next = get_data(morton3d((x, y, z+1)))
                    entering = val_curr < iso and val_next >= iso
                    exiting = val_curr >= iso and val_next < iso
                    
                    if entering or exiting:
                        v0 = calculate_dual_point(xyz, 8) # EDGE3
                        v1 = calculate_dual_point((x-1, y, z), 2) # EDGE1
                        v2 = calculate_dual_point((x-1, y-1, z), 32) # EDGE5
                        v3 = calculate_dual_point((x, y-1, z), 128) # EDGE7
                        
                        idx = triangle_counter
                        triangle_counter += 2
                        
                        if exiting:
                            n1 = calculate_normal(v0, v1, v2)
                            n2 = calculate_normal(v2, v3, v0)
                            triangle_output.append((v0, v1, v2, n1))
                            triangle_output.append((v2, v3, v0, n2))
                        else:
                            n1 = calculate_normal(v2, v1, v0)
                            n2 = calculate_normal(v0, v3, v2)
                            triangle_output.append((v2, v1, v0, n1))
                            triangle_output.append((v0, v3, v2, n2))

    return triangle_output
```

## Subtopics

### [Dual Marching Cubes with Adaptive Octree and QEF Feature Isolation](2291-dual-marching-cubes-with-adaptive-octree-and-qef-feature-iso.md)
This method extracts an isosurface from a scalar field by recursively subdividing space into an adaptive octree based on a Quadratic Error Function (QEF) threshold, then applying a dual marching cubes algorithm on the leaf nodes. It differs from standard Marching Cubes by replacing the fixed grid with an adaptive octree and using QEF-based feature isolation to determine subdivision, extracting the surface from the dual cells formed by these leaves.

### [Dual Isosurfacing with Analytical Trilinear Intersection](3798-dual-isosurfacing-with-analytical-trilinear-intersection.md)
Dual Isosurfacing with Analytical Trilinear Intersection extracts the topology of an analytical isosurface defined by a trilinear function within a voxel cell by determining the optimal set of diagonals to intersect with rays, ensuring one point per disjoint surface component. Unlike standard Marching Cubes which uses linear interpolation and fixed lookup tables, this method uses the trilinear function exhaustively to determine the topology of the analytical isosurface, allowing for more accurate diagonal selection. It builds on [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md) by resolving ambiguous configurations through analytical ray intersection rather than precomputed lookup tables.

### [Curvature/Normal Similarity Loss](4354-curvature-normal-similarity-loss.md)
A differentiable loss function that enforces normal vector alignment between triangles in adjacent grid cells, weighted by the joint probability of their topological configurations. It addresses the discontinuity and noise in extracted surfaces by ensuring that neighboring triangles share consistent orientations, which is critical for smooth surface reconstruction in differentiable isosurfacing pipelines. It sets itself apart from sibling methods by explicitly modeling the probabilistic nature of topology selection (via softmax over 256 cases) rather than relying on deterministic hard assignments or purely geometric penalties.

### [Geometric Self-Intersection Elimination](4757-geometric-self-intersection-elimination.md)
Geometric Self-Intersection Elimination is a post-processing geometric treatment applied during the connection of dual vertices in adaptive octree constructions to ensure the resulting isosurface is free of polygon self-intersections. It addresses the topological defects inherent in naive dual triangulations of adaptive grids by replacing trivial triangulations with triangle fans or geometrically validated subsets when ambiguous face configurations are detected. Unlike ambiguity treatments for regular (non-adaptive) constructions, which primarily ensure manifoldness without increasing vertex/face counts, this method explicitly increases mesh complexity to guarantee intersection-free geometry.

### [Parallel Processing and Pipeline Optimization](6411-parallel-processing-and-pipeline-optimization.md)
This family defines a two-pass parallel algorithm for extracting manifold quadrilateral meshes from segmented voxel data or scalar fields on GPUs. It solves the problem of unconnected primitives and missing adjacency data inherent in standard parallel extraction by combining static and dynamic lookup tables with parallel reduction algorithms. Unlike standard parallel Marching Cubes which produces redundant and unconnected fragments, this method ensures manifold connectivity through a multi-staged pipeline that switches from block-based to cell-based processing. It builds on [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md) but extends it for interactive performance on parallel hardware.

### [Dual Nodes Generation with Connectivity Inversion](7685-dual-nodes-generation-with-connectivity-inversion.md)
This method resolves non-manifold edge configurations in Manifold Dual Marching Cubes (MDMC) by generating two dual nodes instead of one for cells sharing ambiguous faces, thereby recovering topological tunnels. It differs from standard DMC by analyzing the connectivity of both interior (blue) and exterior (white) vertex components to determine node placement, whereas standard DMC relies solely on interior components.

### Dual Marching Cubes Problematic Config Packing
Initializes the `ProblematicConfigs` array (size 64 uints, 256 bytes) to zero. Iterates `pack` from 0 to 63. Calculates `config` as `pack * 4`. Packs four consecutive values from the `problematicConfigs` array into the current `ProblematicConfigs[pack]` entry using bitwise OR and shifts: bit 0-7 from `config`, bit 8-15 from `config+1`, bit 16-23 from `config+2`, bit 24-31 from `config+3` [S5, unofficial].

### Ambiguity Treatment for Regular Constructions
A method to ensure the resulting surface is manifold by identifying and resolving ambiguous faces in adjacent cells during the construction phase, specifically for regular octree constructions [S6, unofficial]. This treatment ensures a manifold surface without increasing the number of vertices and faces, unlike the self-intersection elimination method in adaptive constructions [S6, unofficial]. Processing time impact (Level 6) increases processing time by only 3 seconds [S6, unofficial].

### Schaefer et al. Hexahedral Grid Contouring
A method that extracts contours by performing Marching Cubes on a hexahedral grid dual to the octree grid. The hexahedral grid is generated by extending DC to volumetric functions [S7, unofficial]. The resulting surface is guaranteed to be a 2-manifold, but the method is very expensive to compute [S7, unofficial].

### Surface Diffusion Flow and Tangent Movement Smoothing
A smoothing technique applied to the extracted mesh to improve visual fidelity, utilizing the 1-ring vertex neighborhood gathered during extraction [S8, unofficial]. It gathers the 1-ring vertex neighborhood for each vertex directly during the surface extraction process, applies surface diffusion flow to the mesh vertices, and applies tangent movement to the mesh vertices [S8, unofficial]. Smoothing requires manifold surfaces, connectivity information, and 1-ring neighborhoods, which are typically missing in standard parallel extractions. This method integrates neighbor gathering into extraction to solve this [S8, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels processed, $O(N)$, where $N$ is the number of cells in the grid. The ambiguity resolution step adds a constant-time neighbor lookup [S5, unofficial].
- **Memory Complexity:** Dominated by the volume data storage and the dual point cache. For a $64^3$ grid, the volume data is 1MB (packed), and the dual point map is 512KB (pointers/IDs) [S5, unofficial].
- **Measured Performance:** On a $36 \times 36 \times 36$ grid, the theoretical worst-case triangle buffer size is $34^3 \times 18 = 707,472$ triangles [S5, unofficial].
- **Strengths:** Guarantees manifold connectivity for ambiguous cases (C16/C19) by checking neighbors [S5, unofficial]. Preserves sharp features better than primal methods in some contexts [S2].
- **Limitations:** In flat regions where the adaptive octree is much coarser than the model thickness, the dual partition may fail to generate samples interior to the surface, resulting in topological holes [S1, unofficial]. Requires neighbor lookups which can complicate parallel implementations [S5, unofficial].
- **When to use:** When a manifold mesh is required from ambiguous configurations and the grid is relatively regular or the ambiguity resolution overhead is acceptable.
- **When not to use:** For highly adaptive octrees with extreme coarseness in flat regions, or when strict performance constraints prevent neighbor lookups [S1, unofficial].

## Background
Manifold Dual Marching Cubes was introduced to address the topological inconsistencies of standard Dual Marching Cubes in ambiguous configurations. The concept of a dual grid for preserving sharp features using MC was presented in earlier publications [S2]. The specific problematic configurations C16 and C19 were identified in the Nielsen paper [S3, unofficial]. The algorithm relies on precomputed lookup tables for topology determination [S4, unofficial]. It is used in various GPU-based implementations for efficient contouring of scalar fields [S5, unofficial].

## Sources
[S1] Unconstrained isosurface extraction on arbitrary octrees, https://doi.org/10.5555/1281991.1282009, fetched 2026-10-27 (unofficial)
[S2] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S3] dominikwodniok/dualmc, https://github.com/dominikwodniok/dualmc, fetched 2026-10-26 (unofficial)
[S4] aleokdev/dmc, https://github.com/aleokdev/dmc, fetched 2026-10-27 (unofficial)
[S5] Markyparky56/dualmc-compute, https://github.com/Markyparky56/dualmc-compute, fetched 2026-10-27 (unofficial)
[S6] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
[S7] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
[S8] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
