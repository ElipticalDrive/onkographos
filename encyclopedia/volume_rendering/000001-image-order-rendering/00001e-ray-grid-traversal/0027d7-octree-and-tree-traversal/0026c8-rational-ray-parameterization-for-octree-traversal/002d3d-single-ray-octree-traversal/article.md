# Single-Ray Octree Traversal

Also called: Octree Traversal Strategy

## Overview
Single-ray octree traversal is a method for rendering implicit surfaces by ray tracing an octree-encapsulated volume directly, using the octree as an acceleration structure to locate cell primitives [S1, unofficial]. It solves the problem of efficiently finding the intersection between a ray and a piecewise $C^0$ isosurface defined by trilinear interpolant patches within octree cells [S1, unofficial]. Unlike coherent traversal techniques that process packets of rays or frustums, this method processes one ray at a time, prioritizing optimized single-ray performance over the modest speedups gained from coherence in highly variant scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; an octree volume where scalar values are stored at leaf nodes and interior nodes maintain pointers [S1, unofficial].
*   **Output:** Intersection point, normal, and depth of the isosurface, or null if no intersection is found.

**Data Structures:**
The octree is an adaptive-resolution, hierarchically compressed scalar field [S1, unofficial].
*   **Leaf Nodes:** Store scalar values. At maximum depth $d_{max}$, these correspond to the finest available data resolution. Scalars at less than maximum depth store coarser resolutions by factors of 8 per depth level [S1, unofficial].
*   **Interior Nodes:** Maintain pointers to children. To avoid explicitly storing a full node for each leaf, the implementation stores nodes corresponding to the parent [S1, unofficial].
*   **Cap Structure:** At depth $d_{max} - 1$, a separate structure called a "cap" is employed, consisting simply of 8 scalar values at $d_{max}$ [S1, unofficial]. All children at $d_{max}$ are guaranteed to be leaves.
*   **Cell Primitive:** A cell is composed of 8 voxels. Values outside the octree data are defined to be zero [S1, unofficial]. The implicit surface inside each cell is a trilinear interpolant patch [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Traversal:** The ray traverses the interior nodes of the octree. This involves determining which child node the ray enters next.
2.  **Cell Retrieval:** When the ray reaches a leaf node (or a node where the primitive is defined), the algorithm retrieves the cell primitive. This primitive is composed of 8 voxels [S1, unofficial].
3.  **Neighbor Finding:** To construct the cell of 8 voxels, the algorithm performs neighbor-finding on the octree structure to retrieve the forward-neighboring voxels relative to the current leaf [S1, unofficial]. This yields a complete cell of 8 scalar values [S1, unofficial].
4.  **Intersection:** The ray intersects the well-defined cell primitive (trilinear interpolant patch) composed of the 8 retrieved voxels [S1, unofficial]. The global surface is piecewise $C^0$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_ray_octree_traversal(ray_origin, ray_direction, octree):
    """
    Traces a single ray through an octree volume to find an isosurface intersection.
    Assumes the octree uses the 'cap' structure at max depth - 1.
    """
    # Start traversal from the root
    node = octree.root
    
    while node is not None:
        # Determine which child the ray enters
        # This requires ray-box intersection tests against the 8 children
        # If the ray does not intersect the current node's bounding box, return None
        if not ray_intersects_node(ray_origin, ray_direction, node.bbox):
            return None
            
        # Check if this is a leaf node or a node requiring cell primitive evaluation
        if node.is_leaf or node.is_cap_level:
            # Retrieve the 8 voxels forming the cell primitive
            # This involves neighbor-finding to get forward neighbors
            voxel_values = get_cell_voxels(node, octree)
            
            # Intersect ray with the trilinear interpolant patch defined by these 8 voxels
            intersection = intersect_trilinear_patch(ray_origin, ray_direction, voxel_values)
            
            if intersection is not None:
                return intersection
            else:
                # No intersection in this cell, continue traversal if there are deeper leaves
                # In a pure leaf representation, we might stop or check neighbors
                # For this specific method, we assume the cell primitive covers the intersection
                break
        else:
            # Interior node: descend into the child that the ray intersects
            child_index = get_ray_child_index(ray_origin, ray_direction, node)
            node = node.children[child_index]
            
    return None

def get_cell_voxels(node, octree):
    """
    Retrieves the 8 voxels forming the cell primitive for a leaf node.
    Uses neighbor-finding to get forward neighbors.
    """
    # Get the scalar value at the current leaf node
    current_value = node.scalar_value
    
    # Perform neighbor-finding to retrieve the 7 forward-neighboring voxels
    # The exact neighbor-finding algorithm is complex and depends on the octree layout
    # [S1, unofficial] mentions details in appendices, but the concept is retrieving
    # the neighborhood to form a 2x2x2 block of voxels.
    neighbors = find_forward_neighbors(node, octree)
    
    # Combine current value and neighbors to form the 8-voxel cell
    # The mapping associates each voxel to the lower-left corner of a cell [S1, unofficial]
    cell_voxels = [current_value] + neighbors
    
    return cell_voxels
```

**Parameters:**
*   **Isosurface Value:** Implicitly zero for the trilinear patch intersection check in the context of the cell primitive construction [S1, unofficial].
*   **Max Depth:** Determines the finest resolution. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Neighbor Finding Complexity:** The worst-case complexity of neighbor-finding is $O(\dots)$ (specific bound not given in the sources) [S1, unofficial].
*   **Coherence Trade-offs:** Aggressively coherent techniques (e.g., frustum-based) entail unnecessary work when a pixel covers multiple voxels, potentially decreasing performance over single-ray techniques [S1, unofficial].
*   **Scene Variance:** Highly variant scenes and large data volumes result in modest speedups from coherent techniques, making single-ray traversal competitive [S1, unofficial].

## Subtopics

### Octree Traversal with Polynomial Coefficient Check
An octree structure where polynomial coefficients are stored only at leaves containing the field isosurface, with empty nodes collapsed to be as large as possible to reduce memory requirements [S2]. During ray traversal, a basic octree traversal algorithm (instead of sphere marching) is used to find if the ray intersects any leaf containing the isosurface [S2]. To determine if a node contains the isosurface, the algorithm inspects polynomial coefficients using the convex property of Bézier polynomials [S2]. The range of control points (convex hull) must contain zero for the surface to be present [S2]. For tricubic polynomials, coefficients are computed from field values and derivatives at 8 vertices and transformed to the 64 control points of the corresponding tricubic Bézier patch via a linear transformation [S2]. If the range of these 64 control values does not contain zero, the zero iso-surface is guaranteed not to be inside the node [S2].

## Operating characteristics
*   **Time Complexity:** The traversal time depends on the depth of the octree and the cost of neighbor-finding. The worst-case complexity of neighbor-finding is not fully specified in the sources [S1, unofficial].
*   **Memory Complexity:** The octree is adaptive and hierarchically compressed. Storing nodes corresponding to parents and using a "cap" structure at $d_{max}-1$ reduces memory overhead compared to storing a full node for each leaf [S1, unofficial]. The polynomial coefficient variant further reduces memory by collapsing empty nodes [S2].
*   **Performance:** The method focuses on optimized single-ray behavior. Coherent techniques (SIMD, packets) are not employed because highly variant scenes and large data volumes offer only modest speedups from coherence, and frustum-based techniques can decrease performance due to unnecessary work on pixels covering multiple voxels [S1, unofficial].
*   **Strengths:** Efficient for large data volumes where coherence gains are minimal; reduces memory usage through adaptive resolution and node collapsing.
*   **Limitations:** Neighbor-finding can be complex; single-ray traversal may not exploit hardware parallelism as effectively as packet traversal in highly coherent scenes.

## Background
This method originates from research on ray tracing implicit surfaces for interactive rendering, specifically focusing on octree-encapsulated volumes [S1, unofficial]. It builds on the duality between voxels and cells, mapping each voxel to the lower-left corner of a cell [S1, unofficial]. The approach was developed to avoid extracting meshes and sending geometry to the GPU, instead ray tracing the octree directly [S1, unofficial]. The polynomial coefficient variant was proposed to improve intersection search at iterations far from the surface in SDF rendering [S2].

## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S2] Rendering piecewise approximations of SDFs through analytic intersections, https://doi.org/10.1016/j.cag.2024.103981, fetched 2026-10-27
