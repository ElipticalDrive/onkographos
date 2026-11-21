# Boolean Operations on Distance Fields

Also called: Boolean Operations on SDFs, SDF Boolean Operations, SDF Boolean Operators, SDF Boolean Union, SDF Min, Boolean Union on Distance Fields

## Overview
Boolean operations on Signed Distance Fields (SDFs) are Constructive Solid Geometry (CSG) primitives that combine the scalar fields of two or more shapes to create complex geometries [S2, S4, S11, unofficial]. By applying component-wise minimum or maximum functions to the distance values, these operations define the union, intersection, and difference of volumes [S4, S11, unofficial]. Unlike [Constructive SDF Modeling (Min Combination)](871-constructive-sdf-modeling-min-combination.md) which may focus on specific blending or hierarchical structures, this method provides the fundamental algebraic operators for exact CSG composition, ensuring the resulting field remains a valid SDF (satisfying the Lipschitz condition) for use in sphere tracing [S16, unofficial].

## How it works
The method evaluates two SDFs, $f_1(\mathbf{p})$ and $f_2(\mathbf{p})$, at a query point $\mathbf{p}$ and combines their scalar outputs. The result is a new SDF $f_{result}(\mathbf{p})$ representing the combined shape.

### Inputs and Outputs
*   **Inputs**: Two SDF functions $f_1, f_2: \mathbb{R}^3 \to \mathbb{R}$, and a query point $\mathbf{p} \in \mathbb{R}^3$.
*   **Outputs**: A scalar distance value $d_{result} \in \mathbb{R}$. Optionally, a material ID or normal vector may be returned alongside the distance [S6, S13, unofficial].

### Data Structures
The core data structure is the scalar distance value. When tracking material identity (e.g., for shading different primitives differently), the SDF evaluation returns a pair:
*   `vec2` or struct `{ distance: float, material_id: int }` [S6, S13, unofficial].
*   The distance component (`.x`) is used for ray marching.
*   The material ID component (`.y`) is used for shading logic [S6, S13, unofficial].

### Formulas
Let $d_1 = f_1(\mathbf{p})$ and $d_2 = f_2(\mathbf{p})$.

1.  **Union**: The volume occupied by either shape. The distance is the minimum of the two, as the surface is defined by the closest boundary [S4, unofficial][S8][S11, S16, unofficial].
    $$ d_{union}(\mathbf{p}) = \min(d_1, d_2) $$
2.  **Intersection**: The volume occupied by both shapes. The distance is the maximum of the two, as the surface is defined by the farthest boundary that still keeps the point inside both volumes [S4, S5, S11, S16, unofficial].
    $$ d_{intersect}(\mathbf{p}) = \max(d_1, d_2) $$
3.  **Subtraction (Difference)**: The volume of the first shape minus the second. The second shape's distance is negated (inverting its inside/outside status) before taking the maximum [S1, S4, unofficial][S8][S11, S14, unofficial].
    $$ d_{sub}(\mathbf{p}) = \max(d_1, -d_2) $$
    *Note: Some sources define subtraction as $A - B$ using $\max(d_A, -d_B)$ [S11, S14, unofficial], while others describe it as removing object 1 from object 2 using $\max(-d_1, d_2)$ [S3, unofficial]. The standard CSG interpretation $A \setminus B$ uses $\max(d_A, -d_B)$.*

### Pseudocode
The following implementation handles both scalar distance and material ID tracking, consistent with [S6, unofficial] and [S13, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_sdf_union(p, sdf1, sdf2):
    """
    Computes the union of two SDFs.
    Returns (distance, material_id).
    """
    d1, id1 = sdf1(p)
    d2, id2 = sdf2(p)
    
    if d1 < d2:
        return d1, id1
    else:
        return d2, id2

def evaluate_sdf_intersection(p, sdf1, sdf2):
    """
    Computes the intersection of two SDFs.
    Returns (distance, material_id).
    """
    d1, id1 = sdf1(p)
    d2, id2 = sdf2(p)
    
    if d1 > d2:
        return d1, id1
    else:
        return d2, id2

def evaluate_sdf_subtraction(p, sdf1, sdf2):
    """
    Computes the difference (subtraction) of two SDFs: sdf1 - sdf2.
    Returns (distance, material_id).
    """
    d1, id1 = sdf1(p)
    d2, id2 = sdf2(p)
    
    # Negate the distance of the subtracted object
    neg_d2 = -d2
    
    if d1 > neg_d2:
        return d1, id1
    else:
        return neg_d2, id2

def scene_sdf(p, primitives):
    """
    Combines multiple primitives using union.
    primitives: list of (sdf_func, material_id)
    """
    # Initialize with a large distance (empty space)
    # and a background material ID (e.g., 0)
    current_dist = float('inf')
    current_id = 0
    
    for sdf_func, mat_id in primitives:
        d, _ = sdf_func(p)
        # Union operation: take the minimum distance
        if d < current_dist:
            current_dist = d
            current_id = mat_id
            
    return current_dist, current_id
```

### Parameters
*   **Operation Type**: Enum or string indicating 'union', 'intersection', or 'difference' [S10, unofficial].
*   **Smoothness (Optional)**: Standard boolean operations produce sharp edges. Smooth variants (e.g., Smooth Union) use a blending parameter $k$ to interpolate between min/max, but this breaks the exact SDF property (Lipschitz constant 1) and is not a strict boolean operation [S13, S9, unofficial].

### Edge Cases and Pitfalls
*   **Lipschitz Condition**: The min/max of 1-Lipschitz functions is also 1-Lipschitz, preserving the validity of sphere tracing [S2, 9643, unofficial].
*   **Normals**: The normal at the surface is the gradient of the resulting SDF. For boolean operations, the normal is simply the normal of the primitive that determined the minimum/maximum distance [S2, S10, unofficial]. If $d_{union} = d_1$, then $\mathbf{n} = \nabla f_1(\mathbf{p})$. If $d_{union} = d_2$, then $\mathbf{n} = \nabla f_2(\mathbf{p})$ [S10, unofficial].
*   **Degenerate Intersection**: If two shapes touch exactly, the distance is 0. The normal may be undefined or ambiguous if the gradients are opposite. In practice, sphere tracing handles this by stopping at the surface.
*   **Precision**: When $d_1 \approx d_2$, floating-point precision errors may cause flickering or incorrect normal selection. Using a small epsilon for comparison can help, but strict equality is rare in continuous space.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per primitive pair. For $N$ primitives combined via union, the complexity is $O(N)$ per ray step, as each primitive must be evaluated to find the minimum distance [S13, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory per operation. No additional data structures are required beyond the primitive definitions [S11, unofficial].
*   **Performance**: The operations are extremely fast, consisting of basic arithmetic comparisons. They are highly parallelizable on GPUs [S12, unofficial].
*   **Strengths**: Exact CSG representation; no tessellation required; infinite resolution; easy to animate and transform [S2, S11, unofficial].
*   **Limitations**: Sharp edges only (without smooth blending); complex scenes with many primitives can be slow due to $O(N)$ evaluation per step; no support for non-manifold geometry or holes that require topological changes beyond CSG [S12, S16, unofficial].

## Background
Boolean operations on SDFs were popularized by Inigo Quilez in his articles on distance functions and raymarching [S11, S13, unofficial]. The technique leverages the mathematical properties of distance fields to allow constructive solid geometry without explicit mesh representation. It is a cornerstone of procedural raymarching, enabling the creation of complex scenes from simple primitives like spheres, boxes, and tori [S2, unofficial][S8]. The method is widely used in real-time graphics, demoscene productions, and educational raymarching tutorials [S4, S6, S14, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S3] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
[S4] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S5] ChadMcKinney/Dirac-Sea-Engine, https://github.com/ChadMcKinney/Dirac-Sea-Engine, fetched 2026-10-26 (unofficial)
[S6] 6 – Raymarching: Floor + Union of Rounded Box and Sphere, http://www.geeks3d.com/20130524/building-worlds-with-distance-functions-in-glsl-raymarching-glslhacker-tutorial-opengl/6/, fetched 2026-10-26 (unofficial)
[S7] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S8] Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere ..., https://diglib.eg.org/items/7f21e58c-639e-4359-bf94-414738612006, fetched 2026-10-27
[S9] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S10] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S11] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S12] Dmajster/surface_nets, https://github.com/Dmajster/surface_nets, fetched 2026-10-27 (unofficial)
[S13] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S14] jk9g13/LaFortuna-SDF-Ray-marching, https://github.com/jk9g13/LaFortuna-SDF-Ray-marching, fetched 2026-10-27 (unofficial)
[S15] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S16] Volumetric Rendering: Signed Distance Functions - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
