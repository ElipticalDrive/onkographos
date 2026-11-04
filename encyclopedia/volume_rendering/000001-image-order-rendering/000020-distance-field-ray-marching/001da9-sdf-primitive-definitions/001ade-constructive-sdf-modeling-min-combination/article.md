# Constructive SDF Modeling (Min Combination)

Also called: SDF Union, Union of SDFs, Scene SDF Composition

## Overview
Constructive SDF Modeling (Min Combination) is a method for composing multiple Signed Distance Field (SDF) primitives into a single implicit surface representing their geometric union [S1, S2, S3, S5, S6, S7, unofficial]. It solves the problem of defining complex scenes without triangle meshes by evaluating the minimum distance from a query point to any primitive in the scene [S7, unofficial]. Unlike [Boolean Operations on Distance Fields](1961-boolean-operations-on-distance-fields.md), which generalizes to intersection and complement, or [Smooth SDF Composition](2685-smooth-sdf-composition.md), which blends surfaces, this method strictly enforces the nearest-surface control required for safe sphere tracing [S7, unofficial]. It builds on [SDF Primitive Definitions](1578-sdf-primitive-definitions.md) by aggregating their outputs.

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{p} \in \mathbb{R}^3$; a collection of $N$ SDF primitives, where each primitive $i$ provides a function $d_i(\mathbf{p})$ returning the signed distance to that primitive's surface.
*   **Output:** A single scalar value $D_{scene}(\mathbf{p})$ representing the signed distance to the closest surface in the composed scene.

**Data Structures:**
The scene is typically stored as a collection of primitive objects. A common layout, as seen in implementation examples, separates primitives by type into vectors [S4, unofficial]:
*   `world_spheres`: `std::vector<Sphere*>`
*   `world_boxes`: `std::vector<Box*>`
*   `world_mandelbulbs`: `std::vector<MandelBulb*>`
*   `world_toruses`: `std::vector<Torus*>`

Each primitive object implements a method `pointDist(point)` (or similar) that computes its specific SDF [S4, unofficial].

**Formulas:**
The core operation is the minimum function [S2, S3, S4, S5, S6, unofficial]:
$$ D_{union}(\mathbf{p}) = \min_{i=0}^{N-1} (d_i(\mathbf{p})) $$
Or equivalently for two primitives $A$ and $B$ [S2, S6, unofficial]:
$$ S_{union}(\mathbf{p}) = \min(S_A(\mathbf{p}), S_B(\mathbf{p})) $$

**Parameters:**
*   $N$: The number of primitives in the scene.
*   Primitive-specific parameters (e.g., sphere center $\mathbf{c}$, radius $r$; box half-sizes; torus major/minor radii) are encapsulated within the primitive objects [S1, S4, unofficial].

**Algorithm Steps:**
1.  Initialize `min_dist` to positive infinity (or a sufficiently large value, though `std::numeric_limits<double>::max()` is preferred to avoid precision issues with far-away objects) [S4, S7, unofficial].
2.  Iterate through each primitive $i$ in the scene:
    a.  Evaluate the primitive's SDF at the query point $\mathbf{p}$ to get $d_i$.
    b.  If $d_i < \text{min\_dist}$, update $\text{min\_dist} = d_i$.
3.  Return `min_dist`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scene_sdf_union(p: Vector3, primitives: List[Primitive]) -> float:
    """
    Computes the signed distance to the union of all primitives in the list.
    
    Args:
        p: The query point in 3D space.
        primitives: A list of objects, each implementing a 'dist(p)' method.
        
    Returns:
        The minimum signed distance to any primitive surface.
    """
    # Initialize with infinity to ensure the first primitive always sets the value.
    # Using a hardcoded large number like 999 is a pitfall if objects are farther away [S4, unofficial].
    min_dist = float('inf')
    
    for primitive in primitives:
        # Evaluate the specific SDF for this primitive
        d = primitive.dist(p)
        
        # Keep the smallest distance found so far
        if d < min_dist:
            min_dist = d
            
    return min_dist
```

**Edge Cases and Pitfalls:**
*   **Initialization Value:** Initializing `min_dist` to a hardcoded constant (e.g., 999) is incorrect if all scene objects are farther than that distance from the query point, leading to an incorrect return value [S4, unofficial]. It must be initialized to infinity or the distance of the first object [S4, unofficial].
*   **Overshooting:** Using a distance larger than the minimum (e.g., from a far object) causes the ray marcher to take a step that skips over the nearer surface, breaking the convergence guarantee of sphere tracing [S7, unofficial].
*   **Signedness:** The method assumes the input SDFs are correctly signed (negative inside, positive outside) for the union to represent the solid region where *either* primitive is solid [S2, S6, unofficial].

## Subtopics

### [CSG Difference](1238-csg-difference.md)
Subtracts one SDF from another using negation and minimum, carving volume.

### [Object Combination Operators](1801-object-combination-operators.md)
Blends SDFs to create rounded edges or chamfers, preserving Lipschitz continuity.

### [Smooth SDF Composition](2685-smooth-sdf-composition.md)
Uses smooth minimum operators to create organic, transitional volumes between objects.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per query point, where $N$ is the number of primitives. Each primitive must be evaluated to find the global minimum [S4, S7, unofficial].
*   **Memory Complexity:** $O(N)$ to store the primitive definitions and their parameters [S4, unofficial].
*   **Strengths:** Simple to implement; provides exact geometric union for convex primitives; guarantees safe stepping for sphere tracing because the minimum distance is a conservative lower bound [S7, unofficial].
*   **Limitations:** Does not handle intersections or subtractions directly (requires [Boolean Operations on Distance Fields](1961-boolean-operations-on-distance-fields.md)); sharp creases at intersections of primitives; performance degrades linearly with scene complexity [S7, unofficial].
*   **When to use:** Ideal for ray marching scenes composed of distinct, non-intersecting or simply unioned objects where exact distance bounds are required for convergence [S7, unofficial].

## Background
The method is based on the approach described by Iñigo Quilez in his "Distance Functions" tutorials [S7, unofficial]. It relies on the mathematical property that the signed distance function of a union of sets is the minimum of their individual signed distance functions, provided the distance estimators are conservative (do not underestimate the true distance) [S2, S7, unofficial]. This allows sphere tracing to converge without intersecting geometry [S7, unofficial].

## Sources
[S1] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S3] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S4] kuvsamja/lidar-sphere-tracing, https://github.com/kuvsamja/lidar-sphere-tracing, fetched 2026-10-26 (unofficial)
[S5] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S6] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S7] Ray Marching: Signed Distance Fields, Sphere Tracing, and SDF Shading, https://tutorials.tektite.studio/ray-marching/, fetched 2026-10-27 (unofficial)
