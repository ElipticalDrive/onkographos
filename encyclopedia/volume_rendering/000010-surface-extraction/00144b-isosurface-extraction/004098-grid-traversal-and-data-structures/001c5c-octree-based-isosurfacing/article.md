# Octree-based Isosurfacing

Also called: Contouring using Octrees, Isosurfacing with Octree discretization

## Overview
Octree-based isosurfacing replaces the uniform Cartesian grid with an octree spatial subdivision to discretize the domain for surface extraction algorithms such as Marching Cubes or Dual Contouring [S1]. This approach enables adaptive resolution, concentrating computational effort near the isosurface while coarsening the representation in empty or uniform regions [S1]. It sets itself apart from standard grid-based methods by decoupling spatial resolution from the global grid density, though this adaptivity sacrifices the topological guarantees of uniform grids, potentially introducing cracks if the octree is not sufficiently adapted to the surface geometry [S1].

## How it works
The method constructs a hierarchical spatial domain and applies a standard isosurfacing kernel to the leaf nodes of the octree [S1].

**Inputs:**
*   `bbox`: A 3D bounding box defined by minimum and maximum corner points [S1].
*   `split_predicate`: A function determining whether an octree node should be subdivided [S1].
*   `value_function`: A function evaluating the scalar field at any point in space [S1].
*   `gradient_function`: A function evaluating the gradient of the scalar field at any point in space [S1].
*   `isovalue`: The scalar threshold for surface extraction [S1].

**Outputs:**
*   `points`: A collection of vertices forming the isosurface [S1].
*   `triangles`: A collection of triangular faces forming the isosurface mesh [S1].

**Data Structures:**
*   **Octree**: A hierarchical structure where each node has up to 8 children, recursively subdividing the initial `bbox` [S1].
*   **Domain Object**: An abstraction that binds the `octree`, `values`, and `gradients` to provide the isosurfacing algorithm with access to scalar values and gradients at arbitrary locations within the octree's leaf cells [S1, G].

**Steps:**
1.  **Initialize Octree**: Create an octree instance initialized with the `bbox` defined by two 3D points (min and max corners) [S1].
2.  **Refine Octree**: Call the `refine` method on the octree, passing the `split_predicate` [S1].
3.  **Traverse and Collect**: Traverse the octree to collect the range of leaf nodes for processing [S1].
4.  **Construct Value/Gradient Functions**: Bind the `value_function` and `gradient_function` to the octree structure [S1].
5.  **Create Domain**: Instantiate the domain object by passing the `octree`, `values`, and `gradients` [S1].
6.  **Run Isosurfacing**: Execute the chosen isosurfacing algorithm (e.g., `dual_contouring` or `marching_cubes`) on the `domain` with the `isovalue` [S1].
    *   For Dual Contouring, parameters such as `do_not_triangulate_faces` (boolean) and `constrain_to_cell` (boolean) can be passed to control mesh generation details [S1].
7.  **Output**: Return the generated `points` and `triangles` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def run_octree_isosurfacing(bbox_min, bbox_max, split_predicate, value_func, gradient_func, isovalue, algorithm="dual_contouring"):
    # 1. Initialize Octree with bounding box defined by two 3D points
    octree = {"bbox": [bbox_min, bbox_max], "children": []}

    # 2. Refine the octree using the split predicate
    def refine_node(node, predicate):
        if predicate(node["bbox"]):
            min_pt, max_pt = node["bbox"]
            mid = [(min_pt[i] + max_pt[i]) / 2.0 for i in range(3)]
            node["children"] = []
            for i in range(8):
                child_min = [min_pt[j] if (i & (1 << j)) == 0 else mid[j] for j in range(3)]
                child_max = [mid[j] if (i & (1 << j)) == 0 else max_pt[j] for j in range(3)]
                child = {"bbox": [child_min, child_max], "children": []}
                node["children"].append(child)
                refine_node(child, predicate)
        else:
            node["children"] = []
    refine_node(octree, split_predicate)

    # 3. Traverse to collect leaf nodes
    def collect_leaves(node):
        if not node["children"]:
            return [node]
        leaves = []
        for child in node["children"]:
            leaves.extend(collect_leaves(child))
        return leaves
    leaf_nodes = collect_leaves(octree)

    # 4. Construct value and gradient functions bound to the octree
    values = lambda p: value_func(p)
    gradients = lambda p: gradient_func(p)

    # 5. Create the domain object
    domain = {"octree": octree, "values": values, "gradients": gradients}

    # 6. Run the isosurfacing algorithm
    points = []
    triangles = []
    if algorithm == "dual_contouring":
        # Execute dual contouring on domain with isovalue
        # Parameters: do_not_triangulate_faces=True, constrain_to_cell=False
        pass # Algorithm implementation depends on external kernel
    elif algorithm == "marching_cubes":
        # Execute marching cubes on domain with isovalue
        pass # Algorithm implementation depends on external kernel

    # 7. Return results
    return points, triangles
```

**Parameters:**
*   `isovalue`: The scalar threshold. Must be positive in some implementations (e.g., CGAL example requires `offset_value > 0`) [S1].
*   `do_not_triangulate_faces`: Boolean parameter for Dual Contouring. If `true`, skips face triangulation during execution [S1].
*   `constrain_to_cell`: Boolean parameter for Dual Contouring. Set to `false` in standard examples to allow vertices to move freely [S1].

**Edge Cases and Pitfalls:**
*   **Cracks**: If the octree is not sufficiently adapted to the surface geometry, cracks can appear in the resulting Marching Cubes surface [S1]. This occurs because adjacent leaf nodes of different sizes may not share vertices consistently [G].
*   **Loss of Guarantees**: Using an octree as a spatial discretization structure causes standard isosurfacing methods to lose their topological guarantees, which are inherent to uniform Cartesian grids [S1].

## Subtopics

### [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md)
A spatial indexing method that identifies active cells using a balanced binary tree of scalar value intervals, decoupling spatial locality from value-based filtering; choose when output-sensitive query complexity is prioritized over spatial coherence.

### [IsoOctree](1744-isooctree.md)
An unconstrained method that adapts an arbitrary octree to sample the Euclidean Distance Transform of a mesh for simplified representation; choose when mesh simplification and adaptive sampling of distance fields are required.

### [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md)
A hierarchical method using rectilinear block-based octrees where nodes store value intervals to prune branches that cannot contain the target isovalue; choose when efficient spatial pruning of empty regions is needed.

### [Ashida and Badla Octree Contouring](2117-ashida-and-badla-octree-contouring.md)
Extracts contours directly from octrees by identifying cycles of contour faces, avoiding uniform grids but potentially producing non-manifold edges; choose when direct octree contouring is preferred over grid-based polygonization.

### [Unconstrained Isosurface Extraction on Arbitrary Octrees](2413-unconstrained-isosurface-extraction-on-arbitrary-octrees.md)
Guarantees watertightness on arbitrary octrees using binary edge-trees to resolve inconsistencies at face boundaries; choose when topological correctness on non-conforming octrees is mandatory.

### [Consistent Isovertex Definition via Edge-Trees](2427-consistent-isovertex-definition-via-edge-trees.md)
Defines isovertex positions consistently by traversing edge-trees to resolve ambiguity in coarse edges with multiple sign changes; choose when vertex placement consistency across varying octree depths is critical.

### [Adaptive Octree Generation with Flatness Criterion](2482-adaptive-octree-generation-with-flatness-criterion.md)
Refines octree nodes based on geometric flatness (curvature-to-area ratios) to ensure high curvature regions are resolved densely; choose when curvature-driven adaptive refinement is required before extraction.

### [Sampled DMC](3813-sampled-dmc.md)
Adapts Dual Marching Cubes to hashed octrees using Morton keys, constructing a dual grid with vertices at octree leaf centers; choose when extracting meshes from sampled points or complex SDFs stored in hashed octrees.

### [Point-based Isosurface Extraction with Octree Pruning](6707-point-based-isosurface-extraction-with-octree-pruning.md)
A view-dependent method that subdivides cells until sub-pixel resolution is achieved, rendering geometry as point primitives; choose when interactive, visibility-driven rendering of large unstructured grids is needed.

### [Isotopic Approximation of Implicit Curves and Surfaces](10102-isotopic-approximation-of-implicit-curves-and-surfaces.md)
Uses interval arithmetic to adaptively subdivide space via octrees, guaranteeing the mesh is regularly isotopic to the implicit surface; choose when topological correctness and fast visualization of implicit manifolds are required.

### [Adaptive Sampling-based Isosurface Extraction](11618-adaptive-sampling-based-isosurface-extraction.md)
Constructs meshes by adaptively sampling the implicit field using octrees, relying on topological reconstruction algorithms; choose when discrete sampling and reconstruction are preferred over analytic evaluation.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Adaptive resolution allows for high detail where needed and coarse representation elsewhere, reducing computational cost for large volumes with localized features [S1].
*   **Limitations**: Loss of topological guarantees compared to uniform grids; potential for cracks in Marching Cubes if the octree is not adapted to the surface [S1].
*   **When to use**: Not given in the sources.
*   **When not to use**: Not given in the sources.

## Background
The method is implemented in libraries such as CGAL, which provides examples of using octrees for isosurfacing with Dual Contouring and Marching Cubes [S1]. The approach leverages the hierarchical nature of octrees to optimize the discretization of the domain for surface extraction tasks [S1].

## Sources
[S1] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
