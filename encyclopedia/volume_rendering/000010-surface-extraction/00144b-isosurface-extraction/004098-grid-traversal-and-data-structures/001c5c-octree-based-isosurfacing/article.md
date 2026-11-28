# Octree-based Isosurfacing

Also called: Contouring using Octrees, Isosurfacing with Octree discretization

## Overview
Octree-based isosurfacing replaces the uniform Cartesian grid with an octree spatial discretization structure to extract isosurfaces from scalar fields [S1]. This approach enables adaptive resolution, refining the mesh density only where necessary [S1]. Unlike sibling methods that may enforce strict conforming grids or specific topological constraints, this method operates on arbitrary octree structures, though it sacrifices the topological guarantees of uniform grid algorithms, potentially leading to cracks if the octree is not sufficiently adapted to the surface [S1].

## How it works
The method involves constructing an octree domain, defining value and gradient functions bound to that domain, and executing an isosurfacing algorithm (such as Dual Contouring or Marching Cubes) on the resulting structure [S1].

**Inputs:**
- A bounding box defined by minimum and maximum corner points [S1].
- A split predicate function (refinement criterion) that determines which octree nodes to subdivide [S1].
- A scalar value function $V(p)$ that returns the scalar value at point $p$ [S1].
- A gradient function $G(p)$ that returns the gradient vector at point $p$ [S1].
- An isovalue $I$ defining the surface threshold [S1].

**Outputs:**
- A set of vertices $P$ and a set of triangles $T$ representing the extracted mesh [S1].

**Data Structures:**
- **Octree:** A hierarchical spatial partitioning structure where each node represents a cubic region [S1]. Leaf nodes represent the finest resolution cells used for extraction [S1].
- **Domain Object:** An abstraction that encapsulates the octree, the value function, and the gradient function, providing a unified interface for the isosurfacing algorithm to query scalar data [S1].

**Steps:**
1. **Initialize Octree:** Create an octree instance using the bounding box defined by the min and max corners [S1].
2. **Refine Octree:** Apply the split predicate function to recursively subdivide nodes [S1].
3. **Traverse Octree:** Traverse the octree to collect all leaf nodes [S1].
4. **Bind Functions:** Create value and gradient function objects bound to the octree structure [S1].
5. **Create Domain:** Instantiate the domain object, passing the octree, the value function, and the gradient function [S1].
6. **Execute Isosurfacing:** Run the chosen isosurfacing algorithm (e.g., Dual Contouring or Marching Cubes) on the domain object with the specified isovalue [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octree_isosurfacing(bbox_min, bbox_max, split_predicate, value_func, gradient_func, isovalue, algorithm="dual_contouring"):
    """
    Extracts an isosurface using an octree discretization.
    
    Args:
        bbox_min: Point (x, y, z) defining the minimum corner of the bounding box.
        bbox_max: Point (x, y, z) defining the maximum corner of the bounding box.
        split_predicate: Function taking a bounding box and returning True if it should be subdivided.
        value_func: Function taking a Point and returning a scalar value.
        gradient_func: Function taking a Point and returning a Vector (gradient).
        isovalue: The scalar threshold for the isosurface.
        algorithm: String specifying the extraction method ("dual_contouring" or "marching_cubes").
        
    Returns:
        points: List of vertex coordinates.
        triangles: List of triangle indices.
    """
    # Step 1: Initialize Octree
    class Octree:
        def __init__(self, bbox):
            self.bbox = bbox
            self.children = []
            self.is_leaf = True
            
        def refine(self, predicate):
            if not self.is_leaf or not predicate(self.bbox):
                return
            self.is_leaf = False
            dx, dy, dz = (self.bbox[1][0]-self.bbox[0][0])/2, (self.bbox[1][1]-self.bbox[0][1])/2, (self.bbox[1][2]-self.bbox[0][2])/2
            for i in range(2):
                for j in range(2):
                    for k in range(2):
                        child_min = [self.bbox[0][0]+i*dx, self.bbox[0][1]+j*dy, self.bbox[0][2]+k*dz]
                        child_max = [child_min[0]+dx, child_min[1]+dy, child_min[2]+dz]
                        self.children.append(Octree([child_min, child_max]))
            for child in self.children:
                child.refine(predicate)
                
        def traverse_leaves(self):
            if self.is_leaf:
                yield self
            else:
                for child in self.children:
                    yield from child.traverse_leaves()

    octree = Octree([bbox_min, bbox_max])
    octree.refine(split_predicate)

    # Step 2 & 3: Traverse Octree to collect leaves
    leaves = list(octree.traverse_leaves())

    # Step 4: Bind Functions
    class ValueFunction:
        def __init__(self, func, octree):
            self.func = func
            self.octree = octree
        def evaluate(self, point):
            return self.func(point)
            
    class GradientFunction:
        def __init__(self, func, octree):
            self.func = func
            self.octree = octree
        def evaluate(self, point):
            return self.func(point)

    values = ValueFunction(value_func, octree)
    gradients = GradientFunction(gradient_func, octree)

    # Step 5: Create Domain
    class Domain:
        def __init__(self, octree, values, gradients):
            self.octree = octree
            self.values = values
            self.gradients = gradients
    domain = Domain(octree, values, gradients)

    # Step 6: Execute Isosurfacing
    points = []
    triangles = []

    if algorithm == "dual_contouring":
        do_not_triangulate_faces = True
        constrain_to_cell = False
        # Standard dual contouring processes domain leaves to find isovertices and build polygons
        for leaf in domain.octree.traverse_leaves():
            # Evaluate scalar values at cell corners
            corner_values = [domain.values.evaluate(corner) for corner in leaf.bbox_corners()]
            # Find edges crossing the isovalue
            crossing_edges = [i for i in range(12) if (corner_values[i] - isovalue) * (corner_values[(i+1)%8] - isovalue) < 0]
            # Compute isovertex positions and build mesh primitives
            build_dual_contour_mesh(leaf, crossing_edges, isovalue, points, triangles, do_not_triangulate_faces, constrain_to_cell)
    elif algorithm == "marching_cubes":
        for leaf in domain.octree.traverse_leaves():
            corner_values = [domain.values.evaluate(corner) for corner in leaf.bbox_corners()]
            build_marching_cubes_mesh(leaf, corner_values, isovalue, points, triangles)
    else:
        raise ValueError("Unsupported algorithm")

    return points, triangles

def build_dual_contour_mesh(leaf, crossing_edges, isovalue, points, triangles, do_not_triangulate, constrain):
    # Standard operation: projects gradients to a plane, solves a Quadratic Error Function (QEF) for the isovertex, and triangulates the resulting polygon.
    # Implementation details omitted; standard QEF polygonization [G]
    pass

def build_marching_cubes_mesh(leaf, values, isovalue, points, triangles):
    # Standard operation: looks up the cube configuration case, interpolates vertex positions along crossing edges, and outputs triangles.
    # Implementation details omitted; standard table lookup [G]
    pass
```

**Parameters:**
- `do_not_triangulate_faces`: A boolean parameter for Dual Contouring. If true, face triangulation is skipped during execution [S1].
- `constrain_to_cell`: A boolean parameter for Dual Contouring. If false, vertices are not constrained to their parent cells [S1].

**Edge Cases and Pitfalls:**
- **Cracks:** If the octree is not sufficiently adapted to the surface geometry, cracks can appear in the resulting Marching Cubes surface [S1]. This occurs because adjacent leaf nodes of different depths may not share vertices consistently [S1].
- **Loss of Guarantees:** Using an octree instead of a Cartesian grid causes the method to lose the topological guarantees provided by uniform grid algorithms [S1].

## Subtopics

### [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md)
A spatial indexing method that organizes cells by scalar value intervals for efficient query, distinct from the spatial subdivision of octrees. Choose when output-sensitive query complexity $O(\log n + k)$ is required, accepting higher memory overhead.

### [IsoOctree](1744-isooctree.md)
An unconstrained method operating on arbitrary octrees by computing the Euclidean Distance Transform, enabling mesh simplification. Choose when working with distance fields or requiring direct mesh simplification without topological constraints.

### [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md)
A hierarchical method that prunes octree branches based on value intervals to accelerate extraction. Choose when static value-range pruning is sufficient and strict octree hierarchy is preferred over general interval trees.

### [Ashida and Badla Octree Contouring](2117-ashida-and-badla-octree-contouring.md)
Extracts contours directly from octrees by identifying cycles of contour faces, avoiding uniform grids. Choose when avoiding dual hexahedral meshes is desired, accepting higher computational cost and potential non-manifold edges.

### [Unconstrained Isosurface Extraction on Arbitrary Octrees](2413-unconstrained-isosurface-extraction-on-arbitrary-octrees.md)
Guarantees watertightness on arbitrary octrees using binary edge-trees to resolve inconsistencies. Choose when provably watertight surfaces are required without tree refinement or vertex value modification.

### [Consistent Isovertex Definition via Edge-Trees](2427-consistent-isovertex-definition-via-edge-trees.md)
Defines isovertex positions consistently using edge-trees to ensure watertightness. Choose when resolving ambiguous isovertex positions on coarse edges with multiple sign changes is necessary.

### [Adaptive Octree Generation with Flatness Criterion](2482-adaptive-octree-generation-with-flatness-criterion.md)
Generates adaptive octrees based on geometric flatness, serving as a subdivision stage for extraction. Choose when curvature-to-area ratios should drive refinement density for subsequent algorithms like Dual Marching Cubes.

### [Sampled DMC](3813-sampled-dmc.md)
Adapts Dual Marching Cubes to hashed octrees using Morton keys for adaptive resolution. Choose when extracting meshes from sampled points, complex SDFs, or destructible environments without requiring uniform grids.

### [Point-based Isosurface Extraction with Octree Pruning](6707-point-based-isosurface-extraction-with-octree-pruning.md)
A view-dependent method using octree subdivision and visibility pruning for interactive rendering. Choose when interactive GPU rendering of large unstructured grids is required, rather than static mesh generation.

### [Isotopic Approximation of Implicit Curves and Surfaces](10102-isotopic-approximation-of-implicit-curves-and-surfaces.md)
Uses interval arithmetic and octrees/quadtree to generate isotopic approximations without critical point computation. Choose when guaranteed topology and fast visualization/morphing are prioritized over grid-based schemes.

### [Adaptive Sampling-based Isosurface Extraction](11618-adaptive-sampling-based-isosurface-extraction.md)
Constructs meshes by adaptively sampling the implicit field using octrees for spatial refinement. Choose when computational efficiency and ease of implementation on adaptive structures outweigh the need for analytic intersection logic.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Adaptive resolution allows for high detail where needed and coarse representation elsewhere [S1].
- **Limitations:** Loss of topological guarantees compared to uniform grid methods. Cracks may occur if the octree refinement is not aligned with the surface features [S1].
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The use of octrees for isosurfacing is implemented in libraries such as CGAL, which provides examples for both Dual Contouring and Marching Cubes using octree discretization [S1]. This approach allows developers to leverage the flexibility of hierarchical spatial partitioning for surface extraction tasks [S1].

## Sources
[S1] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
