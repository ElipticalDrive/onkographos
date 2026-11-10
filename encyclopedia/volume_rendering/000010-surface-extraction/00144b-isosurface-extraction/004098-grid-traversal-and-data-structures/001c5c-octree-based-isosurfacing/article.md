# Octree-based Isosurfacing

Also called: Contouring using Octrees, Isosurfacing with Octree discretization

## Overview
Octree-based isosurfacing replaces the uniform Cartesian grid with an octree spatial subdivision to discretize the domain for surface extraction algorithms such as Marching Cubes or Dual Contouring [S1]. This approach enables adaptive resolution, concentrating computational effort near the isosurface while coarsening the representation in empty or uniform regions [S1]. It sets itself apart from standard grid-based methods by decoupling spatial resolution from the global grid density, though this adaptivity sacrifices the topological guarantees of uniform grids, potentially introducing cracks if the octree is not sufficiently adapted to the surface geometry [S1].

## How it works
The method constructs a hierarchical spatial domain and applies a standard isosurfacing kernel (e.g., Dual Contouring or Marching Cubes) to the leaf nodes of the octree.

**Inputs:**
*   `bbox`: A 3D bounding box defined by minimum and maximum corner points.
*   `split_predicate`: A function determining whether an octree node should be subdivided.
*   `value_function`: A function evaluating the scalar field at any point in space.
*   `gradient_function`: A function evaluating the gradient of the scalar field at any point in space.
*   `isovalue`: The scalar threshold for surface extraction.

**Outputs:**
*   `points`: A collection of vertices forming the isosurface.
*   `triangles`: A collection of triangular faces forming the isosurface mesh.

**Data Structures:**
*   **Octree**: A hierarchical structure where each node has up to 8 children. The root represents the initial `bbox`. Nodes are refined recursively based on `split_predicate`.
*   **Domain Object**: An abstraction that binds the `octree`, `values`, and `gradients` to provide the isosurfacing algorithm with access to scalar values and gradients at arbitrary locations within the octree's leaf cells.

**Steps:**
1.  **Initialize Octree**: Create an octree instance initialized with the `bbox` defined by two 3D points (min and max corners) [S1].
2.  **Refine Octree**: Call the `refine` method on the octree, passing the `split_predicate`. The predicate determines which nodes to subdivide based on the scalar field's behavior (e.g., presence of the isovalue) [S1].
3.  **Traverse and Collect**: Traverse the octree to collect the range of leaf nodes. This step identifies the active cells where surface extraction will occur [S1].
4.  **Construct Value/Gradient Functions**: Bind the `value_function` and `gradient_function` to the octree structure. These functions allow the domain to query scalar values and gradients at any point, interpolating or evaluating as necessary within the octree's spatial hierarchy [S1].
5.  **Create Domain**: Instantiate the domain object by passing the `octree`, `values`, and `gradients` [S1].
6.  **Run Isosurfacing**: Execute the chosen isosurfacing algorithm (e.g., `dual_contouring` or `marching_cubes`) on the `domain` with the `isovalue` [S1].
    *   For Dual Contouring, parameters such as `do_not_triangulate_faces` (boolean) and `constrain_to_cell` (boolean) can be passed to control mesh generation details [S1].
7.  **Output**: Return the generated `points` and `triangles` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def run_octree_isosurfacing(bbox_min, bbox_max, split_predicate, value_func, gradient_func, isovalue, algorithm="dual_contouring"):
    # 1. Initialize Octree with bounding box
    bbox_points = [bbox_min, bbox_max]
    octree = Octree(bbox_points)
    
    # 2. Refine the octree using the split predicate
    octree.refine(split_predicate)
    
    # 3. Traverse to collect leaf nodes (optional explicit step if domain handles traversal)
    # leaf_range = octree.traverse(Leaves_traversal(octree))
    
    # 4. Construct value and gradient functions bound to the octree
    values = ValueFunction(value_func, octree)
    gradients = GradientFunction(gradient_func, octree)
    
    # 5. Create the domain object
    domain = Domain(octree, values, gradients)
    
    # 6. Run the isosurfacing algorithm
    if algorithm == "dual_contouring":
        points, triangles = dual_contouring(
            domain, 
            isovalue, 
            do_not_triangulate_faces=True, 
            constrain_to_cell=False
        )
    elif algorithm == "marching_cubes":
        points, triangles = marching_cubes(
            domain, 
            isovalue
        )
    else:
        raise ValueError("Unsupported algorithm")
        
    # 7. Return results
    return points, triangles
```

**Parameters:**
*   `isovalue`: The scalar threshold. Must be positive in some implementations (e.g., CGAL example requires `offset_value > 0`) [S1].
*   `do_not_triangulate_faces`: Boolean parameter for Dual Contouring. If `true`, skips face triangulation during execution [S1].
*   `constrain_to_cell`: Boolean parameter for Dual Contouring. Set to `false` in standard examples to allow vertices to move freely [S1].

**Edge Cases and Pitfalls:**
*   **Cracks**: If the octree is not sufficiently adapted to the surface geometry, cracks can appear in the resulting Marching Cubes surface [S1]. This occurs because adjacent leaf nodes of different sizes may not share vertices consistently.
*   **Loss of Guarantees**: Using an octree as a spatial discretization structure causes standard isosurfacing methods to lose their topological guarantees, which are inherent to uniform Cartesian grids [S1].

## Subtopics

### [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md)
Uses a binary tree of value intervals to identify active cells, decoupling spatial locality from value-based filtering.

### [IsoOctree](1744-isooctree.md)
Adapts an arbitrary octree to sample the Euclidean Distance Transform of a mesh for simplified representation.

### [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md)
Uses a rectilinear block-based octree where nodes store value intervals to prune branches that cannot contain the target isovalue.

### [Ashida and Badla Octree Contouring](2117-ashida-and-badla-octree-contouring.md)
Extracts contours directly from octrees by identifying cycles of contour faces, avoiding uniform grids but potentially producing non-manifold edges.

### [Unconstrained Isosurface Extraction on Arbitrary Octrees](2413-unconstrained-isosurface-extraction-on-arbitrary-octrees.md)
Guarantees watertightness on arbitrary octrees using binary edge-trees to resolve inconsistencies at face boundaries.

### [Consistent Isovertex Definition via Edge-Trees](2427-consistent-isovertex-definition-via-edge-trees.md)
Defines isovertex positions consistently by traversing edge-trees to resolve ambiguity in coarse edges with multiple sign changes.

### [Adaptive Octree Generation with Flatness Criterion](2482-adaptive-octree-generation-with-flatness-criterion.md)
Refines octree nodes based on geometric flatness (curvature-to-area ratios) to ensure high curvature regions are resolved densely.

### [Sampled DMC](3813-sampled-dmc.md)
Adapts Dual Marching Cubes to hashed octrees using Morton keys, constructing a dual grid with vertices at octree leaf centers.

### [Point-based Isosurface Extraction with Octree Pruning](6707-point-based-isosurface-extraction-with-octree-pruning.md)
A view-dependent method that subdivides cells until sub-pixel resolution is achieved, rendering geometry as point primitives.

### [Isotopic Approximation of Implicit Curves and Surfaces](10102-isotopic-approximation-of-implicit-curves-and-surfaces.md)
Uses interval arithmetic to adaptively subdivide space via octrees, guaranteeing the mesh is regularly isotopic to the implicit surface.

### [Adaptive Sampling-based Isosurface Extraction](11618-adaptive-sampling-based-isosurface-extraction.md)
Constructs meshes by adaptively sampling the implicit field using octrees, relying on topological reconstruction algorithms.

## Operating characteristics
*   **Time Complexity**: Depends on the number of leaf nodes in the octree and the complexity of the split predicate. Traversal is proportional to the number of leaves. Surface extraction is linear in the number of active cells.
*   **Memory Complexity**: Proportional to the number of octree nodes. Adaptive refinement can significantly reduce memory usage compared to uniform grids for sparse or complex surfaces.
*   **Strengths**: Adaptive resolution allows for high detail where needed and coarse representation elsewhere, reducing computational cost for large volumes with localized features.
*   **Limitations**: Loss of topological guarantees compared to uniform grids; potential for cracks in Marching Cubes if the octree is not adapted to the surface [S1]. Requires careful implementation of the split predicate and potentially additional steps to ensure watertightness.
*   **When to use**: When the volume data is sparse or has varying levels of detail, and adaptive resolution is beneficial.
*   **When not to use**: When strict topological guarantees are required without additional post-processing, or when the surface is uniformly distributed and a uniform grid would be more efficient.

## Background
The method is implemented in libraries such as CGAL, which provides examples of using octrees for isosurfacing with Dual Contouring and Marching Cubes [S1]. The approach leverages the hierarchical nature of octrees to optimize the discretization of the domain for surface extraction tasks.

## Sources
[S1] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
