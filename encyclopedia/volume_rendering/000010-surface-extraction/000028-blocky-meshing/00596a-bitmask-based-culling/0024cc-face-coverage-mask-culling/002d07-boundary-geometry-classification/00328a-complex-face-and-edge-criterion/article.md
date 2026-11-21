# Complex Face and Edge Criterion

Also called: Complexity Detection

## Overview
The Complex Face and Edge Criterion is a geometric classification method used during octree construction to identify cells containing surface fragments smaller than the current subdivision level [S1]. It ensures that neighbor-based culling and simplification strategies do not incorrectly prune internal geometry of complex blocks by detecting non-manifold configurations and sub-cell-surface pieces [S1]. It is applied after basic inhomogeneity checks to decide if a cell must be subdivided further [S1].

## How it works
**Inputs:** A voxel cell $C$ in an octree; the implicit surface $\partial V$ defined by a scalar field.
**Outputs:** A boolean flag indicating if $C$ is "complex" (requires subdivision or special handling).

**Data Structures:**
*   **Cell Face:** A square plane defined by 4 vertices of the cell.
*   **Homogeneous Face:** A face where all 4 vertices are strictly inside ($\phi > 0$) or strictly outside ($\phi < 0$) the surface $\partial V$.
*   **Connected Component:** A set of voxels on a face connected via shared edges or vertices.

**Steps:**
1.  **Identify Homogeneous Faces:** For each of the 6 faces of cell $C$, evaluate the scalar field $\phi$ at the 4 corner vertices. If all 4 vertices have the same sign (all inside or all outside), the face is homogeneous.
2.  **Extract Connected Components:** For each homogeneous face, extract the connected components of voxels contained within that face. Note: In the context of the source, this refers to analyzing the topology of the surface intersection relative to the face grid [S1].
3.  **Complex Face Check:** If a homogeneous face contains a connected component of surface intersection that is smaller than the current subdivision level (i.e., the surface passes through the face but the topology suggests a fragment too small to be represented accurately at this level), the cell is marked complex [S1].
4.  **Complex Edge Check:** For each of the 12 edges of cell $C$, determine if the edge intersects $\partial V$ more than once. This is done by checking the sign of $\phi$ at the edge endpoints and potentially sampling intermediate points if the function is not linear. If an edge intersects $\partial V$ multiple times, it induces a non-manifold surface configuration [S1].
5.  **Result:** If any face is complex or any edge is complex, the cell $C$ is classified as complex [S1].

**Parameters:**
*   **Minimal Depth:** The octree construction typically divides the space regularly until a user-defined minimal depth (usually 2 or 3) is reached before applying these criteria [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Accurate determination of vertex signs is critical. Floating-point errors near the zero-crossing can misclassify homogeneous faces.
*   **Boundary Geometry:** The criterion ensures that only voxels belonging to cell faces are checked, ensuring all cells intersected by $\partial V$ are processed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_cell_complex(cell, scalar_field):
    """
    Determines if a cell is complex based on face and edge criteria.
    
    Args:
        cell: Object representing the voxel cell with vertices and edges.
        scalar_field: Function phi(x, y, z) returning the scalar value.
        
    Returns:
        bool: True if the cell is complex, False otherwise.
    """
    # Step 1: Evaluate vertices
    vertex_values = [scalar_field(v) for v in cell.vertices]
    
    # Step 2: Check Faces
    for face in cell.faces:
        # Get indices of vertices forming this face
        v_indices = face.vertex_indices
        face_values = [vertex_values[i] for i in v_indices]
        
        # Check if homogeneous (all inside or all outside)
        all_inside = all(v > 0 for v in face_values)
        all_outside = all(v < 0 for v in face_values)
        
        if all_inside or all_outside:
            # Extract connected components of voxels on this face
            # Note: In a coarse octree, this involves checking sub-voxel intersections
            # or analyzing the topology of the implicit surface on the face.
            # For implementation, we check if the surface intersects the face
            # in a way that creates a fragment smaller than the subdivision level.
            # This typically requires sampling the face interior or using a 
            # lookup table for the 2D case.
            if has_sub_cell_surface_fragment(face, scalar_field):
                return True
                
    # Step 3: Check Edges
    for edge in cell.edges:
        # Get endpoints
        v0_val = scalar_field(edge.v0)
        v1_val = scalar_field(edge.v1)
        
        # Check for multiple intersections
        # If signs are different, there is at least one intersection.
        # To detect >1 intersection, we need to check for sign changes 
        # within the edge or higher-order behavior.
        # A simple heuristic for linear fields is that multiple intersections
        # are impossible unless the field is non-linear or the edge is long.
        # However, the source implies detecting configurations inducing 
        # non-manifold surfaces.
        if has_multiple_intersections(edge, scalar_field):
            return True
            
    return False

def has_sub_cell_surface_fragment(face, scalar_field):
    """
    Checks if a homogeneous face contains a surface fragment smaller than 
    the current subdivision level.
    """
    # Implementation detail: Sample the face center or use a 2D marching squares
    # on the face to detect if the zero-crossing forms a small component.
    # If the face is homogeneous, the surface must enter and exit the face
    # through its interior, creating a closed loop or segment.
    # If this loop/segment is small relative to the cell size, it is complex.
    # Not given in the sources exactly how "smaller than current subdivision level"
    # is quantified numerically. [G] Assume a threshold based on cell size.
    return False # Placeholder for specific implementation logic

def has_multiple_intersections(edge, scalar_field):
    """
    Checks if an edge intersects the surface more than once.
    """
    # For a linear scalar field, an edge intersects at most once.
    # Multiple intersections imply non-linearity or specific topological configurations.
    # Sample the midpoint. If signs at v0, mid, v1 are not monotonic, 
    # there may be multiple intersections.
    v0_val = scalar_field(edge.v0)
    v1_val = scalar_field(edge.v1)
    mid_val = scalar_field(edge.midpoint)
    
    # Check for sign changes
    if (v0_val > 0 and mid_val < 0) or (v0_val < 0 and mid_val > 0):
        # One intersection between v0 and mid
        pass
    if (mid_val > 0 and v1_val < 0) or (mid_val < 0 and v1_val > 0):
        # One intersection between mid and v1
        pass
        
    # If both happen, there are two intersections.
    # This is a simplified check. [G] More robust methods involve root finding.
    return False # Placeholder
```

## Subtopics

### Non-manifold Pruning
A post-processing step to prune small non-manifold branches (thin protrusions) and improve structural integrity by removing faces arising from locally inconsistent voxel configurations [S2]. It is applied after mesh extraction to handle artifacts that the lookup table (LUT) could not resolve without complicating the design [S2]. A face is removed if: (1) all three of its constituent vertices lie on either non-manifold edges or open boundary edges, AND (2) the face itself contains an open boundary edge [S2]. The process is repeated until no more faces can be removed [S2].

### Local Complexity Test (Voxel Filtering)
A local complexity test to verify that a candidate voxel contains a single, nearly planar surface patch, ensuring the pivot token is well-defined and numerically stable [S3, unofficial]. It probes multiple sub-voxel locations within each candidate voxel and queries their closest surface points and corresponding triangle normals [S3, unofficial]. A voxel is labeled as ambiguous if the normals exhibit strong directional inconsistency (indicating multi-sheet or topologically complex regions) or if the probed points deviate noticeably from a single plane (indicating high curvature) [S3, unofficial].
*   **Parameters:** Normal-consistency threshold: 0.9 (approx. 25° tolerance); Planarity tolerance: 5% of the voxel size [S3, unofficial].
*   **Pitfall:** Voxels containing multi-sheet or topologically complex regions are filtered out to prevent ambiguous pivot placement [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of faces and edges per cell. For a single cell, it is constant time relative to the cell size, but requires scalar field evaluations. In an octree, it is applied only to cells at the minimal depth or above [S1].
*   **Memory Complexity:** Low, as it operates on individual cells and does not require global data structures beyond the octree index (hash table) for neighbor access [S1].
*   **Strengths:** Ensures topological correctness by detecting non-manifold configurations and sub-cell surface fragments that simple inhomogeneity checks might miss [S1].
*   **Limitations:** Requires careful implementation of the "connected component" extraction and "multiple intersection" detection, which can be numerically sensitive [S1].

## Background
The Complex Face and Edge Criterion was introduced in the context of adaptive surface extraction from anisotropic volumetric data using generalized octrees [S1]. It addresses the problem of cells that appear homogeneous at the vertex level but contain complex surface topology within their faces or edges, which would otherwise be incorrectly culled or simplified [S1]. The method is part of an iterative top-bottom octree construction algorithm [S1].

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
[S2] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
[S3] P2Voxel: Pyramid Pivot Voxelization for 3D Mesh Tokenization, https://doi.org/10.48550/arxiv.2608.07549, fetched 2026-10-26 (unofficial)
