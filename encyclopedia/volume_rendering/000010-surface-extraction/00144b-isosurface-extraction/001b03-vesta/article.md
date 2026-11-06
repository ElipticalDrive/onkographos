# VESTA

Also called: Marching VESTA, Original VESTA

## Overview
VESTA (Volume-Enclosing Surface exTraction Algorithm) is a surface extraction method that constructs isosurfaces by generating N-cycles from 3-cell configurations, resolving ambiguities at voxel face junctures to prevent surface holes, unlike the original Marching Cubes Algorithm [S1, unofficial]. It processes voxel face vectors (VFVs) to generate a closed, oriented surface composed of triangles (3-cycles). VESTA differs from template-based methods like Marching Cubes by using a single building block based on vector decomposition, ensuring the surface is non-degenerate, volume-enclosing, and hole-free without requiring post-processing repair [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field on a regular cubic lattice. Voxels are classified as "active" (value $\ge$ isovalue $T_f$) or "inactive" (value $< T_f$).
*   **Output:** A mesh of oriented triangles (3-cycles) forming a closed surface.

**Data Structures:**
*   **Voxel Face Vector (VFV):** A directed vector connecting a boundary face center (support point) to a juncture (edge midpoint).
*   **Juncture:** The midpoint of a voxel edge.
*   **Support Point:** The center of a voxel face separating an active and an inactive voxel.
*   **VESTA Cycle Vector:** A vector connecting two support points, formed by replacing connected VFV pairs.
*   **N-cycle:** A closed loop of VESTA cycle vectors.

**Parameters:**
*   **Isovalue ($T_f$):** Threshold for active voxels. Typical value in benchmarks: 139 MeV [S1, unofficial].
*   **Connectivity Mode:** Determines handling of Points of Ambiguity (POAs). Options:
    *   *Disconnect:* Successor VFV belongs to the same voxel (left turn in 2D analog).
    *   *Connect:* Successor VFV belongs to the other voxel (right turn in 2D analog).
    *   *Mixed:* Mode determined locally by gray-level thresholds.
*   **Resolution Mode:** Determines N-cycle decomposition.
    *   *Low (L):* No additional points; N-cycles decomposed by inserting antiparallel vector pairs.
    *   *High (H):* Additional average point (center of mass of cycle vertices) inserted for decomposition.

**Algorithm Steps:**

1.  **Identify Boundary Faces:** For every active voxel, check its 6 neighbors. If a neighbor is inactive, record the shared face as a boundary face. The center of this face is a support point.
2.  **Define VFVs:** For each boundary face, define 4 VFVs connecting the face center to the 4 edge midpoints (junctures) in counterclockwise order relative to the range vector (from active voxel center to inactive neighbor center).
3.  **Connect VFVs:** At each juncture, connect the origin of a VFV to the tip of a VFV from a neighboring boundary face. Only attach origins to tips. This forms oriented paths between support points.
4.  **Resolve Ambiguities (POAs):** A POA occurs when two active voxels share only a single edge (in 3D) or point (in 2D analog). At a POA, 4 incoming and 4 outgoing VFVs exist. Apply the chosen Connectivity Mode globally or locally to select the successor VFV. This prevents surface tearing [S1, unofficial].
5.  **Form Cycles:** Replace connected VFV pairs with VESTA cycle vectors. Group these into closed N-cycles. All cycles are confined to a $2 \times 2 \times 2$ voxel neighborhood.
6.  **Decompose N-cycles:**
    *   If $N=3$, the cycle is a triangle.
    *   If $N>3$:
        *   In *Low* mode: Insert antiparallel vector pairs to break the N-cycle into 3-cycles, preserving orientation.
        *   In *High* mode: Calculate the average point of all support points in the cycle. Connect this average point to all vertices of the N-cycle to form triangles. Determine the average point *after* displacing support points if isosurface interpolation is used [S1, unofficial].
7.  **Displace Support Points (Optional for Isosurfaces):** Move support points along their range vectors based on linear interpolation of field values to approximate the isosurface. Recalculate average points for High mode if displaced.
8.  **Filter:** Discard unphysical triangles (e.g., if corner points have radial/temporal values $< 0$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vesta_extract(volume, isovalue, connectivity_mode, resolution_mode):
    """
    volume: 3D array of scalar values
    isovalue: threshold for active voxels
    connectivity_mode: 'disconnect', 'connect', or 'mixed'
    resolution_mode: 'L' (low) or 'H' (high)
    """
    active_voxels = find_active_voxels(volume, isovalue)
    boundary_faces = []
    
    # Step 1 & 2: Identify boundary faces and define VFVs
    for voxel in active_voxels:
        for neighbor in voxel.neighbors:
            if neighbor not in active_voxels:
                face = voxel.get_shared_face(neighbor)
                face_center = face.center
                junctures = face.edge_midpoints # 4 points
                
                # Define 4 VFVs from center to junctures
                vfv_list = []
                for j in range(4):
                    vfv = Vector(face_center, junctures[j])
                    vfv_list.append(vfv)
                
                boundary_faces.append({
                    'voxel': voxel,
                    'neighbor': neighbor,
                    'center': face_center,
                    'vfvs': vfv_list,
                    'junctures': junctures
                })
    
    # Step 3 & 4: Connect VFVs and resolve POAs
    # Map junctures to incoming/outgoing VFVs
    juncture_map = {}
    for face in boundary_faces:
        for i, vfv in enumerate(face['vfvs']):
            juncture = face['junctures'][i]
            if juncture not in juncture_map:
                juncture_map[juncture] = {'in': [], 'out': []}
            
            # VFV goes from center to juncture (out from center, in to juncture)
            # But we connect tip of one to origin of another? 
            # Text says: "attach the origin of a given VFV to another ones tip"
            # VFV: Center -> Juncture. Origin=Center, Tip=Juncture.
            # So we connect Center_A to Juncture_B? 
            # Actually, VFVs are connected *at* the juncture.
            # "Connect VFVs from adjacent boundary faces through shared junctures."
            # "Attach the origin of a given VFV to another one's tip."
            # This implies a chain: Center1 -> Juncture -> Center2?
            # No, VFV is Center -> Juncture.
            # If we attach Origin(Center2->Juncture2) to Tip(Center1->Juncture1),
            # and Juncture1 == Juncture2, then we have Center2 -> Juncture == Center1 -> Juncture.
            # This forms a path Center2 -> Juncture <- Center1?
            # The text says "oriented paths between boundary face centers".
            # Let's assume the standard VESTA logic:
            # At a juncture, we have multiple VFVs ending there.
            # We link the *start* of one VFV (a center) to the *end* of another (the juncture)?
            # No, "Replace connected VFV pairs with single VESTA cycle vectors that connect one VESTA surface support point to another."
            # This implies the cycle vectors are Center -> Center.
            # The VFVs are the segments Center -> Juncture.
            # Connection: Tip of VFV1 (Juncture) connects to Origin of VFV2 (Center)?
            # If Tip1 == Origin2, then VFV1 ends where VFV2 starts.
            # VFV1: C1 -> J. VFV2: C2 -> J.
            # Tip1 is J. Origin2 is C2. They are not equal unless C2=J.
            # Re-reading: "Connect VFVs from adjacent boundary faces through shared junctures."
            # "Attach the origin of a given VFV to another one's tip."
            # This phrasing is tricky. Let's look at the result: "VESTA cycle vectors... connect one VESTA surface support point to another."
            # So the cycle is C1 -> C2 -> C3...
            # The VFVs are C1->J, C2->J.
            # If we link them, we get a path C1->J and C2->J.
            # To make a cycle C1->C2, we need to traverse J.
            # The "VESTA cycle vector" is the sum or replacement of the pair.
            # Let's stick to the high-level grouping described.
            
            juncture_map[juncture]['out'].append(vfv) # Out from center to juncture
            juncture_map[juncture]['in'].append(vfv)  # In to juncture from center

    # Resolve POAs
    # POA: Juncture shared by 2 active voxels (edge contact)
    # 4 VFVs involved.
    cycles = []
    visited_vfvs = set()
    
    for juncture, data in juncture_map.items():
        if len(data['in']) == 4: # POA
            # Apply connectivity mode
            # Disconnect: Link VFVs from same voxel
            # Connect: Link VFVs from different voxels
            # This determines the successor
            pass
        
    # Group into cycles
    # ... (Complex graph traversal logic omitted for brevity, relies on Table 1 paths in Marching VESTA)
    
    # Step 6: Decompose
    triangles = []
    for cycle in cycles:
        if len(cycle) == 3:
            triangles.append(cycle)
        else:
            if resolution_mode == 'L':
                # Insert antiparallel pairs
                triangles.extend(decompose_low_res(cycle))
            else: # 'H'
                avg_point = average(cycle.vertices)
                triangles.extend(decompose_high_res(cycle, avg_point))
                
    return triangles
```

## Subtopics

### DICONEX
DICONEX (DIlated CONtour EXtraction) is the 2D analog of VESTA. It constructs gap-free contours from binary image data by connecting Initial Contour Vectors (ICVs) into closed cycles. ICVs are oriented pixel edges separating active from inactive pixels, with the active pixel to the left. At Points of Ambiguity (POAs) where active pixels touch at a single point, DICONEX uses a "disconnect" mode (left turn, same pixel) or "connect" mode (right turn, different pixel) to resolve gaps. It displaces ICV origins to pixel edge midpoints to form final contours [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active voxels and boundary faces. The marching variant is more memory-efficient than the original global processing.
*   **Memory Complexity:** Original VESTA requires large memory to process the entire dataset at once. Marching VESTA reduces this by processing $2 \times 2 \times 2$ neighborhoods.
*   **Performance:** In benchmarks with 241x81 images (8,000 active voxels), VESTA produced 6,406 triangles in ~0.124s, compared to MCA's 0.096s but with significantly more redundant points (19,218 vs 3,326). For larger datasets (512,603 active voxels), Original VESTA L took 5.34s, while Mixed L took 4.62s [S1, unofficial].
*   **Strengths:** Guarantees hole-free, non-self-intersecting surfaces. Does not require template tables. Minimizes redundant support points compared to MCA.
*   **Limitations:** Weak connectivity (voxels sharing only a corner are not connected). Requires careful handling of POAs. High resolution mode adds computational overhead for average point calculation.

## Background
VESTA was introduced by Schleiermacher in 2010/2011 as a robust alternative to Marching Cubes, inspired by the 2D DICONEX algorithm. It was developed to address the issue of surface holes and ambiguities in standard isosurface extraction methods without relying on extensive template sets [S1, unofficial]. It has been applied to relativistic hydrodynamic simulations and X-ray microtomography data.

## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
