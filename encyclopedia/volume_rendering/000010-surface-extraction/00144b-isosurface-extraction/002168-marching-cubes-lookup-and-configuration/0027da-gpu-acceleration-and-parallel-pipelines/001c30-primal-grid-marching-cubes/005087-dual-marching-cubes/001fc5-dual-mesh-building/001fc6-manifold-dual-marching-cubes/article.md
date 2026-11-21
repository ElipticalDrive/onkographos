# Manifold Dual Marching Cubes

Also called: Dual Marching Cubes (DMC), DualMC problematic configurations, DualMCComputeFramework, Manifold DualMC table generation, Primal Contouring of Dual Grids

## Overview
Manifold Dual Marching Cubes (MDMC) extracts a manifold isosurface from a scalar field by placing vertices at dual points (face centers or cell centers) rather than primal grid vertices, resolving the topological ambiguities (C16/C19 cases) inherent in standard Dual Marching Cubes [S3, S5, unofficial]. Unlike standard Dual Marching Cubes, which may produce non-manifold edges or self-intersections in ambiguous configurations, MDMC ensures a watertight, manifold mesh by inverting the cell configuration when shared neighbors are also ambiguous, or by using precomputed lookup tables that encode manifold-preserving connections [S5, unofficial]. It builds on [Dual Mesh Building](1423-dual-mesh-building.md) by providing the specific logic to determine which dual vertices to connect and how to resolve topological conflicts [S5, unofficial].

## How it works
**Inputs:** A scalar volume data array (e.g., distance transform or density field), an isosurface threshold value ($iso$), and grid dimensions [S5, unofficial].
**Outputs:** An indexed triangle mesh (vertices, normals, and connectivity) [S5, unofficial].

**Data Structures:**
1.  **Volume Data:** Packed into a 1D array. For a $64^3$ grid, indices are 6-bit Morton codes ($0 \dots 262143$) [S5, unofficial]. Data is stored as two 16-bit shorts per 32-bit integer to save memory [S5, unofficial].
    *   Layout: `VolumeData.Data[dataIdx]` where `dataIdx = mortonCode // 2` [S5, unofficial].
    *   Retrieval: `value = (VolumeData.Data[dataIdx] >> (16 * (mortonCode % 2))) & 0xFFFF` [S5, unofficial].
2.  **Dual Points List (`DualPointsList`):** A compact lookup table mapping each of the 256 cube configurations to dual point indices [S5, unofficial].
    *   Layout: Array of 512 32-bit integers [S5, unofficial]. Each `uint` stores two 16-bit dual point codes [S5, unofficial].
    *   Packing: Each cube case (0-255) maps to 2 faces, each face has 2 vertices (4 shorts total per case), packed into 2 `uint`s per case [S5, unofficial].
3.  **Problematic Configs Table (`ProblematicConfigs`):** A lookup table for ambiguous cases (C16/C19) [S5, unofficial].
    *   Layout: Array of 64 32-bit integers (256 bytes) [S5, unofficial].
    *   Packing: 4 bytes per entry. Entry `i` stores directions for configs `4*i` to `4*i+3` [S5, unofficial].
    *   Value: Stores a direction code (0-5 for axes, 255 for non-problematic) [S3, S5, unofficial].
4.  **Dual Point Map (`DualPointMap`):** A hash map (array) storing computed dual vertex positions [S5, unofficial].
    *   Layout: Array of size $64^3$ (space for every possible Morton ID), initialized to a sentinel value [S5, unofficial].
    *   Collision Resolution: Linear probing [S5, unofficial]. IDs are masked to 18 bits (`id &= 0x3ffff`) [S5, unofficial].
5.  **Triangle Output:** Array of structs containing `Vert0`, `Vert1`, `Vert2` (positions) and `Normal` [S5, unofficial].
    *   Max Size: $34^3 \times 18 = 707,472$ triangles for a $36^3$ active grid [S5, unofficial].

**Corner Bitmasks:**
Corners are indexed 0-7. Bitmasks: $C_0=1, C_1=2, C_2=4, C_3=8, C_4=16, C_5=32, C_6=64, C_7=128$ [S3, unofficial].
Configuration value is the sum of active corner bits [S3, unofficial].

**Edge Codes:**
12 edges defined as powers of two: $EDGE_0=1, EDGE_1=2, \dots, EDGE_{11}=2048$ [S3, unofficial].

**Algorithm Steps:**

1.  **Precomputation (Table Generation):**
    *   Generate `DualPointsList` using standard Dual Marching Cubes logic, but ensuring manifold connections for ambiguous cases [S5, unofficial].
    *   Generate `ProblematicConfigs`:
        *   Identify representative ambiguous configurations: C16 (corners 0,1,2,6,7 active) and C19 (corners 0,1,2,4,6,7 active) [S3, unofficial].
        *   Assert initial ambiguous face is on the positive X direction: `(config & (C1|C3|C5|C7))` must equal `(C1|C7)` or `(C3|C5)` [S3, unofficial].
        *   Rotate configurations around X, Y, and Z axes to cover all orientations [S3, unofficial].
        *   Rotation functions shift corner bits:
            *   X-axis: `((config & (C0|C1)) << 2) | ((config & (C2|C3)) << 4) | ((config & (C4|C5)) >> 4) | ((config & (C6|C7)) >> 2)` [S3, unofficial].
            *   Y-axis: `((config & (C0|C2)) << 4) | ((config & (C1|C3)) >> 1) | ((config & (C4|C6)) << 1) | ((config & (C5|C7)) >> 4)` [S3, unofficial].
            *   Z-axis: `((config & (C0|C4)) << 1) | ((config & (C1|C5)) << 2) | ((config & (C2|C6)) >> 2) | ((config & (C3|C7)) >> 1)` [S3, unofficial].
        *   Store the axis direction (0-5) for each rotated configuration in `ProblematicConfigs` [S3, unofficial]. Non-problematic configs store 255 [S3, unofficial].

2.  **Main Loop (Per Voxel):**
    *   Iterate over voxel coordinates $(x, y, z)$ from $0$ to $35$ (for a $36^3$ grid) [S5, unofficial].
    *   **Boundary Check:** If $x \ge REDUCED\_X$ (34), $y \ge REDUCED\_Y$ (34), or $z \ge REDUCED\_Z$ (34), skip [S5, unofficial].
    *   **Cell Code Calculation:**
        *   Sample 8 vertices using Morton encoding [S5, unofficial].
        *   Vertex order: $(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)$ [S5, unofficial].
        *   Bit $i$ is set if $value \ge iso$ [S5, unofficial].
    *   **Ambiguity Resolution:**
        *   Get direction from `ProblematicConfigs[cellCode]` [S5, unofficial].
        *   If direction $\neq 255$:
            *   Decode direction: `component = direction >> 1` (0=X, 1=Y, 2=Z), `delta = (direction & 1) ? 1 : -1` [S5, unofficial].
            *   Calculate neighbor coordinates: `neighbourCoords = cellPos + (delta * axis_vector)` [S5, unofficial].
            *   Check if neighbor is within valid bounds ($0 \le coord < REDUCED - 1$) [S5, unofficial].
            *   If valid, get `neighbourCellCode`.
            *   If `ProblematicConfigs[neighbourCellCode] != 255` (neighbor is also ambiguous), invert the current cell code: `cellCode ^= 0xFF` [S5, unofficial].
    *   **Dual Point Lookup:**
        *   For each intersected edge, look up the dual point index in `DualPointsList` [S5, unofficial].
        *   Invert `cellCode` (`cellCode ^= 0xFF`) before lookup to handle ambiguous cases correctly [S5, unofficial].
        *   Iterate $i$ from 0 to 3. Get entry from `DualPointsList`. If `(entry & edgeCode) != 0`, return entry [S5, unofficial].
    *   **Dual Point Computation (if not cached):**
        *   Compute position as the mean of interpolated intersection points on the intersected edges of the face [S5, unofficial].
        *   Interpolation for an edge between $V_a$ and $V_b$: $P = V_a + \frac{iso - V_a}{V_b - V_a}(V_b - V_a)$ [S5, unofficial].
        *   Store in `DualPointMap` using linear probing [S5, unofficial].
    *   **Triangle Generation:**
        *   For each intersected edge (X, Y, Z), generate a quad of 2 triangles [S5, unofficial].
        *   **X-Edge (between $xyz$ and $xyz+(1,0,0)$):**
            *   Check intersection: `(dataCellPos < iso && dataCellPosX1 >= iso)` (entering) or `(dataCellPos >= iso && dataCellPosX1 < iso)` (exiting) [S5, unofficial].
            *   Vertices: $vert_0$ at $xyz$ (EDGE0), $vert_1$ at $xyzZ_1$ (EDGE2), $vert_2$ at $xyzY_1Z_1$ (EDGE6), $vert_3$ at $xyzY_1$ (EDGE4) [S5, unofficial].
            *   Entering: Triangles $(vert_0, vert_1, vert_2)$ and $(vert_2, vert_3, vert_0)$ [S5, unofficial].
            *   Exiting: Triangles $(vert_2, vert_1, vert_0)$ and $(vert_0, vert_3, vert_2)$ [S5, unofficial].
        *   **Y-Edge (between $xyz$ and $xyz+(0,1,0)$):**
            *   Vertices: $vert_0$ at $xyz$ (EDGE8), $vert_1$ at $xyzZ_1$ (EDGE11), $vert_2$ at $xyzX_1Z_1$ (EDGE10), $vert_3$ at $xyzX_1$ (EDGE9) [S5, unofficial].
            *   Exiting: Triangles $(vert_0, vert_1, vert_2)$ and $(vert_2, vert_3, vert_0)$ [S5, unofficial].
            *   Entering: Triangles $(vert_2, vert_1, vert_0)$ and $(vert_0, vert_3, vert_2)$ [S5, unofficial].
        *   **Z-Edge (between $xyz$ and $xyz+(0,0,1)$):**
            *   Vertices: $vert_0$ at $xyz$ (EDGE3), $vert_1$ at $xyzX_1$ (EDGE1), $vert_2$ at $xyzX_1Y_1$ (EDGE5), $vert_3$ at $xyzY_1$ (EDGE7) [S5, unofficial].
            *   Exiting: Triangles $(vert_0, vert_1, vert_2)$ and $(vert_2, vert_3, vert_0)$ [S5, unofficial].
            *   Entering: Triangles $(vert_2, vert_1, vert_0)$ and $(vert_0, vert_3, vert_2)$ [S5, unofficial].
        *   Calculate normals using `calculateFaceNormal(vert0, vert1, vert2)` which returns $-\text{normalize}((vert_1-vert_0) \times (vert_2-vert_0))$ [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def execute(volume_data, grid_size, iso):
    REDUCED_X = 34
    REDUCED_Y = 34
    REDUCED_Z = 34
    VolumeDataSize = 64 * 64 * 64
    
    dual_points_list = pack_dual_points_list()
    problematic_configs = pack_problematic_configs()
    dual_point_map = [-1] * VolumeDataSize
    triangle_output = []
    triangle_counter = [0]
    
    for z in range(36):
        for y in range(36):
            for x in range(36):
                if x >= REDUCED_X or y >= REDUCED_Y or z >= REDUCED_Z:
                    continue
                exec_single(x, y, z, volume_data, iso, dual_points_list, problematic_configs, dual_point_map, triangle_output, triangle_counter)
                
    return triangle_output

def exec_single(x, y, z, volume_data, iso, dual_points_list, problematic_configs, dual_point_map, triangle_output, triangle_counter):
    xyz = (x, y, z)
    xyzX1 = (x-1, y, z)
    xyzY1 = (x, y-1, z)
    xyzZ1 = (x, y, z-1)
    xyzX1Y1 = (x-1, y-1, z)
    xyzX1Z1 = (x-1, y, z-1)
    xyzY1Z1 = (x, y-1, z-1)
    
    dataCellPos = get_data(volume_data, morton3D(xyz))
    
    # X-Edge Processing
    if z > 0 and y > 0:
        dataCellPosX1 = get_data(volume_data, morton3D((x+1, y, z)))
        entering = dataCellPos < iso and dataCellPosX1 >= iso
        exiting = dataCellPos >= iso and dataCellPosX1 < iso
        
        if entering or exiting:
            v0 = get_or_compute_dual_point(xyz, iso, 1, dual_points_list, problematic_configs, dual_point_map)
            v1 = get_or_compute_dual_point(xyzZ1, iso, 4, dual_points_list, problematic_configs, dual_point_map)
            v2 = get_or_compute_dual_point(xyzY1Z1, iso, 64, dual_points_list, problematic_configs, dual_point_map)
            v3 = get_or_compute_dual_point(xyzY1, iso, 16, dual_points_list, problematic_configs, dual_point_map)
            
            start_idx = triangle_counter[0]
            triangle_counter[0] += 2
            
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

    # Y-Edge Processing
    if z > 0 and x > 0:
        dataCellPosY1 = get_data(volume_data, morton3D((x, y+1, z)))
        entering = dataCellPos < iso and dataCellPosY1 >= iso
        exiting = dataCellPos >= iso and dataCellPosY1 < iso
        
        if entering or exiting:
            v0 = get_or_compute_dual_point(xyz, iso, 256, dual_points_list, problematic_configs, dual_point_map)
            v1 = get_or_compute_dual_point(xyzZ1, iso, 2048, dual_points_list, problematic_configs, dual_point_map)
            v2 = get_or_compute_dual_point(xyzX1Z1, iso, 1024, dual_points_list, problematic_configs, dual_point_map)
            v3 = get_or_compute_dual_point(xyzX1, iso, 512, dual_points_list, problematic_configs, dual_point_map)
            
            start_idx = triangle_counter[0]
            triangle_counter[0] += 2
            
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

    # Z-Edge Processing
    if x > 0 and y > 0:
        dataCellPosZ1 = get_data(volume_data, morton3D((x, y, z+1)))
        entering = dataCellPos < iso and dataCellPosZ1 >= iso
        exiting = dataCellPos >= iso and dataCellPosZ1 < iso
        
        if entering or exiting:
            v0 = get_or_compute_dual_point(xyz, iso, 8, dual_points_list, problematic_configs, dual_point_map)
            v1 = get_or_compute_dual_point(xyzX1, iso, 2, dual_points_list, problematic_configs, dual_point_map)
            v2 = get_or_compute_dual_point(xyzX1Y1, iso, 32, dual_points_list, problematic_configs, dual_point_map)
            v3 = get_or_compute_dual_point(xyzY1, iso, 128, dual_points_list, problematic_configs, dual_point_map)
            
            start_idx = triangle_counter[0]
            triangle_counter[0] += 2
            
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

def get_or_compute_dual_point(cell_pos, iso, edge_code, dual_points_list, problematic_configs, dual_point_map):
    cell_code = get_cell_code(cell_pos, iso)
    lookup_code = cell_code ^ 0xFF
    
    idx = find_dual_point(lookup_code, edge_code, dual_points_list)
    
    if idx != 0:
        return dual_point_map[idx]
    
    pos = compute_dual_point_position(cell_pos, iso, edge_code)
    
    morton = morton3D(cell_pos)
    dual_point_map[morton] = pos
    
    return pos

def compute_dual_point_position(cell_pos, iso, edge_code):
    p = [0.0, 0.0, 0.0]
    points = 0
    edge_masks = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048]
    edge_offsets = [
        ((1,0,0), (0,0,0)), ((1,0,1), (1,0,0)), ((0,0,1), (0,0,0)),
        ((0,1,0), (0,0,0)), ((0,1,1), (0,1,0)), ((1,1,1), (0,1,0)),
        ((0,1,1), (0,0,1)), ((1,1,0), (0,1,0)), ((1,0,0), (0,0,0)),
        ((1,1,1), (1,0,1)), ((1,0,1), (1,0,0)), ((0,1,1), (0,0,1))
    ]
    for i, mask in enumerate(edge_masks):
        if edge_code & mask:
            v1_pos = (cell_pos[0] + edge_offsets[i][0][0], cell_pos[1] + edge_offsets[i][0][1], cell_pos[2] + edge_offsets[i][0][2])
            v2_pos = (cell_pos[0] + edge_offsets[i][1][0], cell_pos[1] + edge_offsets[i][1][1], cell_pos[2] + edge_offsets[i][1][2])
            val1 = get_data(None, morton3D(v1_pos)) # Simplified for pseudocode structure
            val2 = get_data(None, morton3D(v2_pos))
            t = (iso - val1) / (val2 - val1) if val2 != val1 else 0.5
            p[0] += v1_pos[0] + t * (v2_pos[0] - v1_pos[0])
            p[1] += v1_pos[1] + t * (v2_pos[1] - v1_pos[1])
            p[2] += v1_pos[2] + t * (v2_pos[2] - v1_pos[2])
            points += 1
    if points > 0:
        return (p[0]/points, p[1]/points, p[2]/points)
    return (0.0, 0.0, 0.0)

def get_data(volume_data, morton_code):
    data_idx = morton_code // 2
    offset = 16 * (morton_code % 2)
    return (volume_data[data_idx] >> offset) & 0xFFFF

def morton3D(v):
    x, y, z = int(v[0]), int(v[1]), int(v[2])
    v = (x, y, z)
    v = tuple(c & 63 for c in v)
    v = tuple((c | (c << 8)) & 0x300F for c in v)
    v = tuple((c | (c << 4)) & 0x30C3 for c in v)
    v = tuple((c | (c << 2)) & 0x9249 for c in v)
    return (v[0]) | ((v[1]) << 1) | ((v[2]) << 2)

def get_cell_code(cell_pos, iso):
    offsets = [(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)]
    code = 0
    for i, off in enumerate(offsets):
        pos = (cell_pos[0]+off[0], cell_pos[1]+off[1], cell_pos[2]+off[2])
        val = get_data(None, morton3D(pos))
        if val >= iso:
            code |= (1 << i)
    return code

def find_dual_point(cell_code, edge_code, dual_points_list):
    for i in range(4):
        idx = (cell_code * 2) + (i // 2)
        packed = dual_points_list[idx]
        shift = (i % 2) * 16
        entry = (packed >> shift) & 0xFFFF
        if entry & edge_code:
            return entry
    return 0

def pack_dual_points_list():
    # Generates 512 uints packing 4 shorts per cube case. Exact table generation is source-specific [G].
    return [0] * 512

def pack_problematic_configs():
    configs = [0] * 64
    for pack in range(64):
        config = pack * 4
        val = 0
        val |= (0 << 0)
        val |= (0 << 8)
        val |= (0 << 16)
        val |= (0 << 24)
        configs[pack] = val
    return configs

def calculate_normal(v0, v1, v2):
    p01 = (v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2])
    p02 = (v2[0]-v0[0], v2[1]-v0[1], v2[2]-v0[2])
    cross = (p01[1]*p02[2]-p01[2]*p02[1], p01[2]*p02[0]-p01[0]*p02[2], p01[0]*p02[1]-p01[1]*p02[0])
    mag = (cross[0]**2 + cross[1]**2 + cross[2]**2)**0.5
    if mag == 0: return (0,0,0)
    return (-cross[0]/mag, -cross[1]/mag, -cross[2]/mag)
```

## Subtopics

### [Dual Marching Cubes with Adaptive Octree and QEF Feature Isolation](2291-dual-marching-cubes-with-adaptive-octree-and-qef-feature-iso.md)
Extracts an isosurface from a scalar field by recursively subdividing space into an adaptive octree based on a Quadratic Error Function (QEF) threshold, then applying a dual marching cubes algorithm on the leaf nodes. It differs from standard Marching Cubes by replacing the fixed grid with an adaptive octree and using QEF-based feature isolation to determine subdivision, extracting the surface from the dual cells formed by these leaves.

### [Dual Isosurfacing with Analytical Trilinear Intersection](3798-dual-isosurfacing-with-analytical-trilinear-intersection.md)
Dual Isosurfacing with Analytical Trilinear Intersection extracts the topology of an analytical isosurface defined by a trilinear function within a voxel cell by determining the optimal set of diagonals to intersect with rays, ensuring one point per disjoint surface component. Unlike standard Marching Cubes which uses linear interpolation and fixed lookup tables, this method uses the trilinear function exhaustively to determine the topology of the analytical isosurface, allowing for more accurate diagonal selection. It builds on [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md) by resolving ambiguous configurations through analytical ray intersection rather than precomputed lookup tables.

### [Curvature/Normal Similarity Loss](4354-curvature-normal-similarity-loss.md)
A differentiable loss function that enforces normal vector alignment between triangles in adjacent grid cells, weighted by the joint probability of their topological configurations. It addresses discontinuity and noise in extracted surfaces by ensuring neighboring triangles share consistent orientations, which is critical for smooth surface reconstruction in differentiable isosurfacing pipelines. It sets itself apart from sibling methods by explicitly modeling the probabilistic nature of topology selection via softmax over 256 cases rather than relying on deterministic hard assignments or purely geometric penalties. It builds on [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md).

### [Geometric Self-Intersection Elimination](4757-geometric-self-intersection-elimination.md)
Geometric Self-Intersection Elimination is a post-processing geometric treatment applied during the connection of dual vertices in adaptive octree constructions to ensure the resulting isosurface is free of polygon self-intersections. It addresses the topological defects inherent in naive dual triangulations of adaptive grids by replacing trivial triangulations with triangle fans or geometrically validated subsets when ambiguous face configurations are detected. Unlike ambiguity treatments for regular (non-adaptive) constructions, which primarily ensure manifoldness without increasing vertex or face counts, this method explicitly increases mesh complexity to guarantee intersection-free geometry.

### [Parallel Processing and Pipeline Optimization](6411-parallel-processing-and-pipeline-optimization.md)
This family covers parallel processing and pipeline optimization for Dual Marching Cubes surface extraction. It addresses topological consistency and parallel reconstruction by encoding neighborhood and connectivity information using a halfedge data structure. Unlike standard parallel extraction that produces unconnected primitives, it handles non-manifold edges (e.g., tunnels spanning two neighbor cells) and enables parallel mesh simplification.

### [Dual Nodes Generation with Connectivity Inversion](7685-dual-nodes-generation-with-connectivity-inversion.md)
This method resolves non-manifold edge configurations in Manifold Dual Marching Cubes (MDMC) by generating two dual nodes instead of one for cells sharing ambiguous faces, thereby recovering topological tunnels. It differs from standard DMC by analyzing the connectivity of both interior (blue) and exterior (white) vertex components to determine node placement, whereas standard DMC relies solely on interior components.

#### Dual Marching Cubes Problematic Config Packing
Initializes the `ProblematicConfigs` array (size 64 uints, 256 bytes) to zero [S5, unofficial]. Iterates `pack` from 0 to 63 [S5, unofficial]. Calculates `config` as `pack * 4` [S5, unofficial]. Packs four consecutive values from the `problematicConfigs` array into the current `ProblematicConfigs[pack]` entry using bitwise OR and shifts: bit 0-7 from `config`, bit 8-15 from `config+1`, bit 16-23 from `config+2`, bit 24-31 from `config+3` [S5, unofficial].

#### Ambiguity Treatment for Regular Constructions
A method to ensure the resulting surface is manifold by identifying and resolving ambiguous faces in adjacent cells during the construction phase, specifically for regular octree constructions [S6, unofficial]. This treatment ensures a manifold surface without increasing the number of vertices and faces, unlike the self-intersection elimination method in adaptive constructions [S6, unofficial]. Processing time impact (Level 6): Increases processing time by only 3 seconds [S6, unofficial].

#### Schaefer et al. Hexahedral Grid Contouring
A method that extracts contours by performing Marching Cubes on a hexahedral grid dual to the octree grid [S7, unofficial]. The hexahedral grid is generated by extending DC to volumetric functions [S7, unofficial]. The resulting surface is guaranteed to be a 2-manifold, but the method is very expensive to compute [S7, unofficial].

#### Surface Diffusion Flow and Tangent Movement Smoothing
A smoothing technique applied to the extracted mesh to improve visual fidelity, utilizing the 1-ring vertex neighborhood gathered during extraction [S8, unofficial]. Gather the 1-ring vertex neighborhood for each vertex directly during the surface extraction process [S8, unofficial]. Apply surface diffusion flow to the mesh vertices [S8, unofficial]. Apply tangent movement to the mesh vertices [S8, unofficial]. Smoothing requires manifold surfaces, connectivity information, and 1-ring neighborhoods, which are typically missing in standard parallel extractions [S8, unofficial]. This method integrates neighbor gathering into extraction to solve this [S8, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is processed independently in the main loop [S5, unofficial]. Precomputation of tables is constant time relative to grid size [S3, unofficial].
**Memory Complexity:** $O(V)$ for volume data and dual point maps, where $V$ is the volume size [S5, unofficial]. The `DualPointsList` and `ProblematicConfigs` tables are constant size (512 uints and 64 uints respectively) [S5, unofficial].
**Measured Performance:** On a $36 \times 36 \times 36$ grid, the theoretical worst-case triangle buffer size is $34^3 \times 18 = 707,472$ triangles [S5, unofficial]. Boundary handling requires checking if neighbor coordinates are within $[0, REDUCED-1]$ to avoid out-of-bounds access [S5, unofficial].
**Strengths:** Guarantees a manifold mesh by resolving C16/C19 ambiguities [S5, unofficial]. Efficient lookup tables allow for fast GPU implementation [S5, unofficial]. Preserves sharp features better than primal methods [S2].
**Limitations:** In flat regions where the adaptive octree is much coarser than the model thickness, the dual partition may fail to generate samples interior to the surface, resulting in topological holes [S1, unofficial]. Ambiguity resolution requires neighbor lookups, which can be costly in parallel implementations if not optimized [S5, unofficial].

## Background
Manifold Dual Marching Cubes was introduced to address the topological inconsistencies of standard Dual Marching Cubes, particularly in ambiguous configurations (C16/C19) [S3, unofficial]. The method relies on precomputed lookup tables to resolve these ambiguities by checking neighbor cells and inverting configurations when necessary [S5, unofficial]. It is used in various surface extraction pipelines, including those for medical imaging (e.g., pelvis reconstruction) and terrain rendering [S1, unofficial][S2]. The algorithm has been implemented in GPU compute frameworks for real-time performance [S5, unofficial].

## Sources
[S1] Unconstrained isosurface extraction on arbitrary octrees, https://doi.org/10.5555/1281991.1282009, fetched 2026-10-27 (unofficial)
[S2] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S3] dominikwodniok/dualmc, https://github.com/dominikwodniok/dualmc, fetched 2026-10-26 (unofficial)
[S4] aleokdev/dmc, https://github.com/aleokdev/dmc, fetched 2026-10-27 (unofficial)
[S5] Markyparky56/dualmc-compute, https://github.com/Markyparky56/dualmc-compute, fetched 2026-10-27 (unofficial)
[S6] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
[S7] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
[S8] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
