# Octree-based Isosurfacing

Also called: Contouring using Octrees, Isosurfacing with Octree discretization

## Overview
Octree-based isosurfacing replaces the uniform Cartesian grid with an octree spatial subdivision to discretize the domain for surface extraction algorithms like Marching Cubes or Dual Contouring [S1]. This approach enables adaptive resolution, refining the mesh density only where necessary, but it sacrifices the topological guarantees of uniform grids, potentially causing cracks in the resulting surface if the octree is not sufficiently adapted to the geometry [S1].

## How it works
The method involves constructing an octree from a bounding box, refining it based on a predicate, and then passing the resulting structure to a standard isosurfacing algorithm [S1].

**Inputs:**
*   A 3D bounding box defined by minimum and maximum corner points.
*   A split predicate function that determines whether an octree node should be subdivided.
*   A scalar value function that evaluates the scalar field at any point in space.
*   A gradient function that evaluates the gradient vector at any point in space (required for Dual Contouring).
*   An isovalue threshold.

**Outputs:**
*   A set of vertices and triangles representing the extracted isosurface.

**Data Structures:**
*   **Octree:** A hierarchical spatial partitioning structure. The root node represents the initial bounding box. Each node has up to 8 children representing sub-cubes. The tree is refined until the split predicate returns false for all leaf nodes [S1].
*   **Domain Object:** A wrapper that binds the octree structure with the value and gradient functions. It provides the interface expected by the isosurfacing algorithm [S1].

**Steps:**
1.  **Initialize Octree:** Create an octree instance using the bounding box defined by two 3D points (min and max corners) [S1].
2.  **Refine Octree:** Call the refinement method on the octree, passing the split predicate. The predicate is evaluated for each node to decide if it should be split into 8 children. This process continues recursively until the desired resolution or criteria are met [S1].
3.  **Traverse Octree:** Traverse the octree to collect all leaf nodes. This step identifies the active cells where the surface may exist [S1].
4.  **Construct Value and Gradient Functions:** Bind the scalar value function and the gradient function to the octree structure. These functions allow the algorithm to query scalar values and gradients at arbitrary points, typically interpolated or evaluated within the context of the octree cells [S1].
5.  **Create Domain Object:** Instantiate a domain object that encapsulates the octree, the value function, and the gradient function. This object serves as the input domain for the isosurfacing algorithm [S1].
6.  **Run Isosurfacing Algorithm:** Execute the chosen isosurfacing algorithm (e.g., Dual Contouring or Marching Cubes) on the domain object. The algorithm iterates over the leaf nodes of the octree to generate vertices and faces [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octree_isosurfacing(bbox_min, bbox_max, split_predicate, value_func, gradient_func, isovalue, algorithm="dual_contouring"):
    # Step 1: Initialize Octree
    # Define bounding box points
    bbox_points = [bbox_min, bbox_max]
    octree = Octree(bbox_points)
    
    # Step 2: Refine Octree
    # Recursively split nodes based on the predicate
    octree.refine(split_predicate)
    
    # Step 3: Traverse Octree
    # Get all leaf nodes
    leaf_range = octree.traverse(Leaves_traversal(octree))
    leaf_count = len(leaf_range)
    
    # Step 4 & 5: Construct Domain
    # Bind value and gradient functions to the octree
    values = ValueFunction(value_func, octree)
    gradients = GradientFunction(gradient_func, octree)
    domain = Domain(octree, values, gradients)
    
    # Step 6: Run Algorithm
    if algorithm == "dual_contouring":
        # Dual Contouring parameters
        do_not_triangulate_faces = True
        constrain_to_cell = False
        points, triangles = dual_contouring(domain, isovalue, 
                                            do_not_triangulate_faces=do_not_triangulate_faces,
                                            constrain_to_cell=constrain_to_cell)
    elif algorithm == "marching_cubes":
        # Marching Cubes does not use gradient function in this context
        points, triangles = marching_cubes(domain, isovalue)
    else:
        raise ValueError("Unsupported algorithm")
        
    return points, triangles
```

**Parameters:**
*   `isovalue`: The scalar threshold for surface extraction.
*   `do_not_triangulate_faces`: A parameter for Dual Contouring. If true, face triangulation is skipped during execution [S1].
*   `constrain_to_cell`: A parameter for Dual Contouring. If false, the method does not constrain vertices to their parent cells [S1].

**Edge Cases and Pitfalls:**
*   **Cracks:** If the octree is not sufficiently adapted to the surface geometry, cracks can appear in the Marching Cubes surface [S1]. This is because the uniform grid topology guarantees are lost when adjacent cells have different depths [S1].
*   **Loss of Guarantees:** Using an octree as a spatial discretization structure causes all methods to lose their topological guarantees [S1].

## Subtopics

### [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md)
Uses a binary tree of value intervals to identify active cells, decoupling spatial locality from value-based filtering.

### [IsoOctree](1744-isooctree.md)
Adapts an arbitrary octree to sample the Euclidean Distance Transform of a mesh for simplified representation.

### [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md)
Accelerates extraction by pruning octree branches that cannot contain the target isovalue using stored value intervals.

### [Ashida and Badla Octree Contouring](2117-ashida-and-badla-octree-contouring.md)
Extracts contours directly from octrees by identifying cycles of contour faces, avoiding uniform grids.

### [Unconstrained Isosurface Extraction on Arbitrary Octrees](2413-unconstrained-isosurface-extraction-on-arbitrary-octrees.md)
Guarantees watertightness on arbitrary octrees using binary edge-trees to resolve inconsistencies at face boundaries.

### [Consistent Isovertex Definition via Edge-Trees](2427-consistent-isovertex-definition-via-edge-trees.md)
Defines isovertex positions consistently using edge-trees to resolve ambiguity in coarse edges with multiple sign changes.

### [Adaptive Octree Generation with Flatness Criterion](2482-adaptive-octree-generation-with-flatness-criterion.md)
Refines octree nodes based on geometric flatness (curvature-to-area ratios) to ensure high curvature regions are resolved.

### [Sampled DMC](3813-sampled-dmc.md)
Adapts Dual Marching Cubes to hashed octrees using Morton keys, constructing a dual grid with vertices at leaf node centers.

### [Point-based Isosurface Extraction with Octree Pruning](6707-point-based-isosurface-extraction-with-octree-pruning.md)
A view-dependent method that recursively subdivides the field until sub-pixel resolution is achieved, rendering points directly.

### [Isotopic Approximation of Implicit Curves and Surfaces](10102-isotopic-approximation-of-implicit-curves-and-surfaces.md)
Uses interval arithmetic to adaptively subdivide space via an octree, guaranteeing the approximation has the same topology as the original manifold.

## Operating characteristics
*   **Time Complexity:** Depends on the split predicate and the depth of the octree. Traversal is proportional to the number of leaf nodes.
*   **Memory Complexity:** Proportional to the number of nodes in the octree, which is typically much less than a uniform grid for sparse or smoothly varying data.
*   **Strengths:** Adaptive resolution allows for high detail where needed and coarse representation elsewhere, reducing memory usage and computation time for large datasets.
*   **Limitations:** Loss of topological guarantees compared to uniform grids. Cracks can appear in Marching Cubes if the octree is not adapted to the surface [S1].
*   **When to use:** When the data has varying levels of detail or when memory constraints prevent the use of a uniform grid.
*   **When not to use:** When topological correctness and watertightness are critical and cannot be ensured by the refinement strategy, or when using algorithms that rely on uniform grid properties.

## Background
The method is implemented in libraries such as CGAL, which provides examples of using an octree as a discretization structure for both Dual Contouring and Marching Cubes [S1]. The approach allows for adaptive resolution but requires careful handling of the octree refinement to avoid artifacts.

## Sources
[S1] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
