# VESTA
Also called: Marching VESTA, Original VESTA

## Overview
VESTA (Volume-Enclosing Surface exTraction) is an isosurface extraction algorithm that constructs non-degenerate, closed 3D surfaces from voxel data by generating N-cycles from 3-cell configurations [S1, unofficial]. Unlike template-based methods such as Marching Cubes, VESTA uses a single building block based on vector decomposition to resolve 3D cell ambiguities at voxel face junctures, guaranteeing surfaces that are hole-free, non-self-intersecting, and volume-enclosing [S1, unofficial]. It operates by treating 2x2x2 neighborhoods (marching VESTA) or the entire dataset (original VESTA) to form closed vector cycles which are then triangulated [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field defined on a regular cubic lattice. Each voxel has a scalar value (e.g., density, temperature).
*   **Parameters:**
    *   `iso_value`: Threshold separating active (value $\ge$ iso_value) and inactive voxels [S2, unofficial].
    *   `connectivity_mode`: Global or local mode for handling Points of Ambiguity (POA). Options: `disconnect`, `connect`, or `mixed` [S1, unofficial].
    *   `resolution_mode`: Determines triangulation detail. Options: `Low` (L, no extra points) or `High` (H, uses average points) [S1, unofficial].
*   **Output:** A mesh of oriented triangles (2-simplices) forming the isosurface.

**Data Structures:**
1.  **Voxel Grid:** 3D array of scalar values.
2.  **Boundary Face Vectors (VFVs):** For each boundary face between an active and inactive voxel, four vectors connect the face center to the four edge midpoints (junctures) in counter-clockwise order relative to the range vector [S1, unofficial].
3.  **Vector Path Table:** A lookup table mapping a quadrant ID and orientation ($\oplus$ or $\ominus$) to a path sequence of junctures. The table stores 24 directed paths (half due to symmetry) [S1, unofficial].
    *   Columns: `Center ID`, `Orientation` ($\oplus/\ominus$), `Path` (sequence of juncture indices).
    *   Example (ID 0): $\oplus$ Path: $13 \to 0 \to 12$; $\ominus$ Path: $12 \to 0 \to 13$ [S1, unofficial].
4.  **Support Points:** Initially the centers of boundary faces. May be displaced within range vectors for isosurface accuracy [S1, unofficial].
5.  **Junctures:** Endpoints of VFVs (edge midpoints). Points where multiple VFVs meet.
6.  **Points of Ambiguity (POA):** Junctures where two incoming and two outgoing VFVs meet (typically where voxels share only an edge or corner) [S2, unofficial].

**Algorithm Steps:**

1.  **Segmentation:** Classify voxels as active ($v \ge \text{iso\_value}$) or inactive ($v < \text{iso\_value}$) [S2, unofficial].
2.  **Identify Boundary Faces:** For each active voxel, check its 6 neighbors. If a neighbor is inactive, record the shared face as a boundary face [S1, unofficial].
3.  **Define VFVs:** For each boundary face, define the face center (support point) and four VFVs connecting the center to the four edge midpoints (junctures) in counter-clockwise order [S1, unofficial].
4.  **Connect VFVs (Cycle Formation):**
    *   Traverse the boundary faces. At each juncture, connect the origin of a VFV to the tip of a VFV from a neighboring boundary face [S1, unofficial].
    *   **Ambiguity Resolution:** If a juncture is a POA (multiple valid connections exist), apply the `connectivity_mode`:
        *   `disconnect`: Connect VFVs belonging to the *same* voxel (prevents linking shapes touching at a single point/edge) [S1, unofficial].
        *   `connect`: Connect VFVs belonging to *different* voxels (links shapes) [S1, unofficial].
        *   `mixed`: Use local gray-level thresholds to decide mode per POA [S1, unofficial].
    *   Replace connected VFV pairs with **VESTA cycle vectors** connecting support points [S1, unofficial].
    *   Group vectors into closed cycles (N-cycles) [S1, unofficial].
5.  **Displacement (Optional for Isosurfaces):** Displace support points (face centers) along the range vector (from active voxel center to inactive neighbor center) based on linear interpolation of scalar values [S1, unofficial].
6.  **Triangulation (Decomposition):**
    *   Decompose N-cycles into triangles (3-cycles).
    *   **Low Resolution (L):** For N > 3, insert antiparallel vector pairs to break cycles into 3-cycles without adding new vertices [S1, unofficial].
    *   **High Resolution (H):** For N > 3, calculate an additional point (average of all cycle support points) and connect it to the cycle vertices to form triangles [S1, unofficial].
7.  **Filtering:** Discard unphysical triangles (e.g., if corner points have negative radial/temporal values) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vesta_extract(volume, iso_value, connectivity_mode, resolution_mode):
    """
    volume: 3D array of scalars
    iso_value: float
    connectivity_mode: 'disconnect', 'connect', or 'mixed'
    resolution_mode: 'L' or 'H'
    """
    active_voxels = find_active_voxels(volume, iso_value)
    boundary_faces = []
    
    # Step 2 & 3: Identify boundary faces and define VFVs
    for voxel in active_voxels:
        for neighbor in voxel.neighbors:
            if neighbor not in active_voxels:
                face = voxel.get_shared_face(neighbor)
                face_center = face.center
                # Define 4 VFVs from center to edge midpoints (junctures)
                vfvs = face.get_vfv_pairs() 
                boundary_faces.append({'face': face, 'center': face_center, 'vfvs': vfvs})
                
    # Step 4: Connect VFVs into cycles
    cycles = []
    visited_vfvs = set()
    
    for face in boundary_faces:
        for vfv in face['vfvs']:
            if vfv.id in visited_vfvs:
                continue
            
            current_vfv = vfv
            cycle = [current_vfv]
            visited_vfvs.add(current_vfv.id)
            
            while True:
                # Find next VVF at the tip of current_vfv
                tip_juncture = current_vfv.tip
                next_vfv = find_next_vfv(tip_juncture, current_vfv, boundary_faces, connectivity_mode)
                
                if next_vfv is None:
                    break # Should not happen in valid closed surface
                
                if next_vfv.id in visited_vfvs:
                    # Cycle closed
                    break
                
                cycle.append(next_vfv)
                visited_vfvs.add(next_vfv.id)
                current_vfv = next_vfv
                
            if len(cycle) > 0:
                cycles.append(cycle)
                
    # Step 5: Displace support points (if isosurface)
    if resolution_mode == 'H' or isosurface_mode:
        for cycle in cycles:
            for vfv in cycle:
                vfv.support_point.displace_along_range_vector(volume)
                
    # Step 6: Triangulate
    triangles = []
    for cycle in cycles:
        if len(cycle) == 3:
            triangles.extend(decompose_3_cycle(cycle))
        else:
            if resolution_mode == 'L':
                triangles.extend(decompose_n_cycle_low_res(cycle))
            else: # 'H'
                avg_point = calculate_average_point(cycle)
                triangles.extend(decompose_n_cycle_high_res(cycle, avg_point))
                
    return triangles

def find_next_vfv(juncture, current_vfv, boundary_faces, mode):
    """
    Resolves ambiguity at juncture.
    Returns the next VVF to connect to current_vfv.tip.
    """
    # Get all VFVs ending at this juncture (incoming) and starting from it (outgoing)
    # In VESTA, we connect tip of current to origin of next.
    # We look for VFVs originating at this juncture.
    
    candidates = get_outgoing_vfvs(juncture, boundary_faces)
    
    if len(candidates) == 1:
        return candidates[0]
    
    # POA detected
    if mode == 'disconnect':
        # Select successor belonging to same voxel as current_vfv
        return select_same_voxel_vfv(candidates, current_vfv)
    elif mode == 'connect':
        # Select successor belonging to different voxel
        return select_diff_voxel_vfv(candidates, current_vfv)
    elif mode == 'mixed':
        # Use local gray level threshold
        return select_mixed_vfv(candidates, current_vfv, juncture)
        
    return None

def decompose_n_cycle_low_res(cycle):
    """Break N-cycle into 3-cycles using antiparallel vector pairs."""
    # Insert edges between non-adjacent vertices to triangulate
    # Specifically, insert antiparallel pairs to preserve orientation
    triangles = []
    n = len(cycle)
    for i in range(1, n - 1):
        # Form triangle using first vertex, i-th, and (i+1)-th support points
        # Note: Exact triangulation logic depends on specific N-cycle geometry
        # Standard fan triangulation from first vertex
        p0 = cycle[0].support_point
        p1 = cycle[i].support_point
        p2 = cycle[i+1].support_point
        triangles.append(Triangle(p0, p1, p2))
    return triangles

def decompose_n_cycle_high_res(cycle, avg_point):
    """Break N-cycle into 3-cycles using an average point."""
    triangles = []
    n = len(cycle)
    for i in range(n):
        p1 = cycle[i].support_point
        p2 = cycle[(i + 1) % n].support_point
        triangles.append(Triangle(avg_point, p1, p2))
    return triangles
```

## Subtopics

### [STEVE Hypersurface Generation](9503-steve-hypersurface-generation.md)
STEVE (Space-Time-Enclosing Volume Extraction) is an algorithm for extracting iso-valued hypersurfaces from 4D data sets, guaranteeing that the resulting hypersurface is free from accidental rifts and fully encloses a region in 4D space. It generates simplex-based manifolds of codimension 1, specifically consisting purely of 3-simplices (tetrahedrons) embedded in 4D space. Unlike methods that extrude 4D data to 5D, STEVE works directly in 4D. It builds on the VESTA framework, using a similar flowchart for mesh features, data segmentation, and ambiguity handling, but extends the vector path logic to four dimensions.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active voxels and boundary faces. The marching variant processes 2x2x2 neighborhoods independently, allowing for parallelization [S1, unofficial].
*   **Memory Complexity:** Original VESTA requires storing the entire dataset and all cycles in memory, which can be high for large volumes. Marching VESTA reduces memory usage by processing local 3-cells [S1, unofficial].
*   **Performance:** In benchmarks on 200 x-ray microtomography images (256x256 pixels, ~512k active voxels), Original VESTA L took 5.34s, Mixed L took 4.62s, and MCA L took 4.71s [S1, unofficial]. VESTA generates significantly fewer support points than MCA (e.g., 932k points vs 5.5M points for the same triangle count), reducing redundancy [S1, unofficial].
*   **Strengths:** Guarantees hole-free, non-self-intersecting, volume-enclosing surfaces. No template management required. Robust against topological ambiguities [S1, unofficial].
*   **Limitations:** Does not connect voxels sharing only a single point (corner contact) by default, which may require post-processing or template insertion for weak connectivity [S1, unofficial]. High-resolution mode requires calculating average points, adding computational overhead [S1, unofficial].

## Background
VESTA was introduced as a novel volume-enclosing surface extraction technique for 3D data sets on cubic lattices [S1, unofficial]. It was developed to address the limitations of the Marching Cubes Algorithm (MCA), particularly the generation of holes and the need for extensive template sets to handle ambiguities [S1, unofficial]. The method is analogous to the 2D DICONEX algorithm [S1, unofficial]. It has been applied to computer tomographic scan data and relativistic hydrodynamic simulation data [S1, unofficial].

## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
[S2] STEVE - Space-Time-Enclosing Volume Extraction, https://arxiv.org/abs/1302.5683, fetched 2026-10-27 (unofficial)
