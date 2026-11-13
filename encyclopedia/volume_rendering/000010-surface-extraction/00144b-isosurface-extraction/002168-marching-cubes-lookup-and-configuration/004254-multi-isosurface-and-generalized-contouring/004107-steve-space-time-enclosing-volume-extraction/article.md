# STEVE (Space-Time-Enclosing Volume Extraction)

Also called: 4D protomesh-based iso-hypersurface construction algorithm

## Overview
STEVE is a 4D protomesh-based algorithm that generates iso-hypersurfaces from 4D volumetric data by transforming boundary cube octants into tetrahedrons [S1, unofficial]. It sets itself apart from sibling methods like [Generalized Marching Cubes](8320-generalized-marching-cubes.md) by using a single 4D building block and a directed vector path system rather than a massive lookup table of configurations, thereby avoiding the exponential storage requirements of template-based approaches [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D scalar field (toxels) defined on a grid.
*   **Output:** A set of tetrahedrons representing the iso-hypersurface.

**Data Structures:**
The algorithm relies on the storage of 192 directed vector paths for 4D space [S1, unofficial]. This number represents half of the total paths due to $\oplus/\ominus$-symmetry [S1, unofficial]. The paths are used to define the geometry of the final tetrahedrons.

**Steps:**
1.  **Identify Boundary:** Identify the boundary cubes where the scalar field crosses the isovalue.
2.  **Initial Octants:** Identify the octants of these boundary cubes that contain the iso-surface.
3.  **Protomesh Construction:** Transform the set of initially identified octants into a "protomesh" using a single 4D building block [S1, unofficial].
4.  **Path Generation:** Generate cyclic vector paths. The algorithm uses normalized normal vectors attached to the centers of oriented $(N-1)$-simplices. Each component of these normal vectors is larger than zero, pointing to the exterior of the enclosed $N$-dimensional regions [S1, unofficial].
5.  **Tetrahedron Formation:** Continuously transform the protomesh into the final set of tetrahedrons [S1, unofficial].
6.  **Propagation:** Propagate information about the interior/exterior of the enclosed 4D regions to the final results [S1, unofficial].

**Parameters:**
*   **Resolution Mode:** The user can choose between high resolution and lower resolution decomposition of the generated polytopes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Topological Ambiguities:** Discretized 4D spaces allow for two different global treatments of topological ambiguities. The algorithm must properly treat these to prevent accidental rifts in the final hypersurfaces [S1, unofficial].
*   **Weak Connectivity:** Toxels in direct contact only through a single vertex or edge result in two disjunct hypersurface segments (two tetrahedrons each) [S1, unofficial]. In these cases, the initial cyclic vector paths form two disjunct sets with four cyclic paths each [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# STEVE Algorithm Pseudocode
# Note: Specific implementation details of the "single 4D building block" 
# and the exact transformation of octants to tetrahedrons are not provided 
# in the source text beyond the high-level description [S1, unofficial].

def steve_extract_4d_hypersurface(data, isovalue, resolution_mode):
    """
    Extracts 4D iso-hypersurface from 4D volumetric data.
    
    Args:
        data: 4D numpy array of scalar values.
        isovalue: The threshold value for extraction.
        resolution_mode: 'high' or 'low' for decomposition of polytopes.
        
    Returns:
        list of tetrahedrons.
    """
    # 1. Identify boundary cubes and initial octants
    # [G] Implementation detail: Iterate over grid, check sign changes relative to isovalue.
    boundary_octants = identify_boundary_octants(data, isovalue)
    
    # 2. Initialize directed vector paths
    # Storage requirement: 192 directed vector paths for 4D [S1, unofficial]
    # [G] Implementation detail: Define the 192 paths based on normalized normal vectors
    # pointing to the exterior (all components > 0).
    paths = initialize_192_directed_paths()
    
    # 3. Construct protomesh and transform to tetrahedrons
    # The algorithm uses a single 4D building block [S1, unofficial]
    tetrahedrons = []
    for octant in boundary_octants:
        # [G] Implementation detail: Apply the single 4D building block transformation
        # to the octant using the directed paths.
        tetra = transform_octant_to_tetrahedron(octant, paths, resolution_mode)
        tetrahedrons.append(tetra)
        
    # 4. Handle weak connectivity (vertex/edge contact)
    # [G] Implementation detail: Check for toxels connected only by vertex or edge.
    # If found, ensure they result in two disjunct sets of tetrahedrons [S1, unofficial].
    tetrahedrons = resolve_weak_connectivity(tetrahedrons)
    
    # 5. Propagate interior/exterior information
    # [G] Implementation detail: Assign orientation/normal direction to each tetrahedron
    # based on the exterior-pointing normal vectors [S1, unofficial].
    tetrahedrons = propagate_interior_exterior_info(tetrahedrons)
    
    return tetrahedrons
```

## Subtopics

### 4D Shape Characterization
Used for visualizing 4D transverse phase space in ion beam transport [S1, unofficial].

### Freezeout Hypersurface Extraction
Used in relativistic heavy-ion physics to visualize fireball expansion [S1, unofficial].

### Intersection Studies
Used to study intersections of low and high resolution hypersurfaces [S1, unofficial].

## Operating characteristics
*   **Storage Complexity:** The algorithm requires the storage of 192 directed vector paths for 4D [S1, unofficial]. This is significantly less than the $2^{16}$ (65,536) entries required for a full lookup table in template-based methods like MCA [S1, unofficial].
*   **Scalability:** The number of directed paths grows with dimension $N$ as $\prod_{k=2}^N 2k$, while the number of templates for MCA grows as $2^{2^N}$ [S1, unofficial]. This makes STEVE more scalable to higher dimensions than template-based methods.
*   **Performance:** For a $50 \times 50 \times 50 \times 50$ toxel data set with an iso-value of 1, STEVE in high resolution 'mixed' mode generated 129,563 tetrahedrons [S1, unofficial].
*   **Strengths:** Minimal data redundancy; avoids multiple counting of manifold supporting points; handles 4D specifically without the exponential blowup of general $N$-D template tables.
*   **Limitations:** Specific to 4D (or requires extension for higher $N$); topological ambiguities require careful handling to prevent rifts.

## Background
STEVE (Space-Time-Enclosing Volume Extraction) was introduced as a 4D protomesh-based iso-hypersurface construction algorithm [S1, unofficial]. It was developed to address the limitations of template-based hypersurface generators (like VESTA or MCA) in 4D, where the number of configurations becomes prohibitively large ($2^{16}$ for 4D) [S1, unofficial]. The method was applied to problems in accelerator physics (ion beam transport) and relativistic heavy-ion physics (freezeout hypersurfaces) [S1, unofficial].

## Sources
[S1] STEVE - Space-Time-Enclosing Volume Extraction, https://arxiv.org/abs/1302.5683, fetched 2026-10-27 (unofficial)
