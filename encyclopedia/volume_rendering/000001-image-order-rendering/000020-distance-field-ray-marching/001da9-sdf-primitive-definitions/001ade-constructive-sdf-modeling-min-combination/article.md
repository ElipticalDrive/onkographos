# Constructive SDF Modeling (Min Combination)

Also called: SDF Scene Composition, SDF Union, Scene SDF Composition, Union of SDFs

## Overview
Constructive SDF Modeling (Min Combination) constructs a single Signed Distance Function (SDF) representing the geometric union of multiple primitives by evaluating the minimum distance to any object in the scene at a query point [S1, S2, S3, S5, S7, S8, unofficial]. It solves the problem of describing complex scenes without triangle meshes, enabling ray marching to step safely toward the nearest surface by guaranteeing that the returned distance is a lower bound on the true distance to the scene geometry [S2, S8, unofficial]. It differs from its parent, [SDF Primitive Definitions](1578-sdf-primitive-definitions.md), by combining individual primitives into a composite field, and from [Boolean Operations on Distance Fields](1961-boolean-operations-on-distance-fields.md) by specifically implementing the union operation (intersection and complement are handled via max and negation respectively) [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A query point $\mathbf{p} \in \mathbb{R}^3$ in world space, and a scene description containing a set of $N$ primitives. Each primitive $i$ has an associated local SDF function $d_i(\mathbf{p})$ and transformation parameters (center, orientation, scale) [S1, S4, S6, unofficial].
- **Output:** A scalar value $d_{scene}(\mathbf{p})$ representing the signed distance to the nearest surface in the union of all primitives [S4, S8, unofficial]. Optionally, the index or material ID of the primitive providing this minimum distance [S6, unofficial].

**Data Structures:**
The scene is typically stored as a collection of primitive objects. A common layout separates primitives by type for iteration [S4, unofficial]:
- `world_spheres`: Array of Sphere objects, each storing center $\mathbf{c}_i$ and radius $r_i$.
- `world_boxes`: Array of Box objects, each storing center $\mathbf{c}_i$ and half-sizes $\mathbf{b}_i$.
- `world_toruses`: Array of Torus objects, each storing center $\mathbf{c}_i$, major radius $R_i$, and minor radius $r_i$.
- `world_mandelbulbs`: Array of Mandelbulb objects, each storing center $\mathbf{c}_i$, scale, and power parameters.
- Global arrays for uniform properties, such as `sphereCenters` (array of vec3) and `sphereRadii` (array of float) [S1, unofficial].

**Formulas:**
The core operation is the minimum function [S2, S3, S5, S7, unofficial]:
$$ d_{scene}(\mathbf{p}) = \min_{i=0}^{N-1} (d_i(\mathbf{p})) $$
where $d_i(\mathbf{p})$ is the signed distance from $\mathbf{p}$ to the $i$-th primitive [S4, S5, unofficial].

Common primitive SDFs used in this composition include [S1, S4, S6, unofficial]:
- **Sphere:** $d_{sphere}(\mathbf{p}, \mathbf{c}, r) = \|\mathbf{p} - \mathbf{c}\| - r$
- **Plane (Floor):** $d_{plane}(\mathbf{p}) = \mathbf{p} \cdot \mathbf{n} - d$, often simplified to $p.y - y_{offset}$ for horizontal planes [S1, S6, unofficial].
- **Box (AABB):** $d_{box}(\mathbf{p}, \mathbf{c}, \mathbf{b}) = \|\max(\mathbf{p} - \mathbf{c} - \mathbf{b}, \mathbf{0})\| + \min(\max((\mathbf{p} - \mathbf{c} - \mathbf{b}).x, (\mathbf{p} - \mathbf{c} - \mathbf{b}).y, (\mathbf{p} - \mathbf{c} - \mathbf{b}).z), 0.0)$ [S4, S6, unofficial].
- **Torus:** $d_{torus}(\mathbf{p}, \mathbf{c}, R, r) = \|\mathbf{q}\| - r$, where $\mathbf{q} = (\|\mathbf{p}_{xz} - \mathbf{c}_{xz}\| - R, p_y - c_y)$ [S4, S6, unofficial].

**Parameters:**
- $N$: The number of primitives in the scene [S1, unofficial].
- $r_i, R_i$: Radii parameters for spheres and tori [S1, S4, unofficial].
- $\mathbf{c}_i$: Center positions for all primitives [S1, S4, unofficial].

**Steps:**
1. Initialize `min_dist` to positive infinity (or a sufficiently large value like `std::numeric_limits<double>::max()`) [S4, S8, unofficial].
2. For each primitive $i$ in the scene:
   a. Transform the query point $\mathbf{p}$ into the local coordinate space of primitive $i$ (if the primitive is not centered at the origin or aligned with axes) [G].
   b. Evaluate the local SDF $d_i(\mathbf{p}_{local})$ using the primitive's specific formula [S4, S6, unofficial].
   c. If $d_i(\mathbf{p}_{local}) < \text{min\_dist}$, update `min_dist = d_i(\mathbf{p}_{local})` [S4, unofficial].
3. Return `min_dist` [S4, S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sphere_sdf(p, center, radius):
    """Signed distance to a sphere."""
    return math.sqrt(sum((p[j] - center[j])**2 for j in range(3))) - radius

def box_sdf(p, center, half_size):
    """Signed distance to an axis-aligned box."""
    # Transform to local space
    q = [p[j] - center[j] for j in range(3)]
    # Calculate distance to box surface
    outside = math.sqrt(sum(max(q[j], 0.0)**2 for j in range(3)))
    inside = min(max(q[0], max(q[1], q[2])), 0.0)
    return outside + inside

def torus_sdf(p, center, big_r, small_r):
    """Signed distance to a torus aligned with Y axis."""
    # Transform to local space
    px = p[0] - center[0]
    py = p[1] - center[1]
    pz = p[2] - center[2]
    # Project to XZ plane
    q_x = math.sqrt(px*px + pz*pz) - big_r
    q_y = py
    return math.sqrt(q_x*q_x + q_y*q_y) - small_r

def plane_sdf(p, normal, offset):
    """Signed distance to a plane defined by normal and distance from origin."""
    return sum(p[j] * normal[j] for j in range(3)) - offset

def scene_sdf(p, primitives):
    """
    Evaluate the constructive SDF union for a scene.
    
    Args:
        p: Query point (vec3)
        primitives: List of tuples (type, params...)
                    type: 'sphere', 'box', 'torus', 'plane'
                    params: primitive-specific parameters
    
    Returns:
        min_dist: The signed distance to the nearest surface
    """
    min_dist = float('inf')
    
    for prim in primitives:
        ptype = prim[0]
        
        if ptype == 'sphere':
            center = prim[1]
            radius = prim[2]
            d = sphere_sdf(p, center, radius)
        elif ptype == 'box':
            center = prim[1]
            half_size = prim[2]
            d = box_sdf(p, center, half_size)
        elif ptype == 'torus':
            center = prim[1]
            big_r = prim[2]
            small_r = prim[3]
            d = torus_sdf(p, center, big_r, small_r)
        elif ptype == 'plane':
            normal = prim[1]
            offset = prim[2]
            d = plane_sdf(p, normal, offset)
        else:
            # Handle other primitives or raise error
            d = float('inf')
            
        if d < min_dist:
            min_dist = d
            
    return min_dist
```

**Edge Cases and Pitfalls:**
- **Initialization:** Initializing `min_dist` to a hardcoded small value (e.g., 999) can cause incorrect results if all scene objects are farther away than this value. It must be initialized to infinity or the distance to the first object [S4, unofficial].
- **Overshooting:** Using a distance larger than the minimum (e.g., from a far object) instead of the true minimum can cause the ray marching step to skip across the nearer surface, leading to rendering artifacts or failure to converge [S8, unofficial].
- **Lipschitz Continuity:** The `min` operator preserves the Lipschitz condition with constant 1, ensuring the distance estimate is a valid lower bound for sphere tracing [G]. However, the resulting field is not differentiable at the boundaries where two primitives are equidistant, which may affect normal estimation if not handled via finite differences or alternative methods [S2, S3, unofficial].

## Subtopics

### [CSG Difference](1238-csg-difference.md)
Performs boolean subtraction using max and negation.

### [Object Combination Operators](1801-object-combination-operators.md)
Merges SDFs with rounded edges using a radius parameter.

### [Smooth SDF Composition](2685-smooth-sdf-composition.md)
Blends SDFs using a smooth minimum operator for organic transitions.

## Operating characteristics
- **Time Complexity:** $O(N)$ per query point, where $N$ is the number of primitives in the scene [S4, S8, unofficial]. Each primitive must be evaluated to find the minimum.
- **Memory Complexity:** $O(N)$ to store the primitive parameters [S4, unofficial].
- **Performance:** Performance scales linearly with the number of primitives. For scenes with many objects, spatial partitioning (e.g., BVH) can be used to accelerate the search, but the basic constructive modeling approach evaluates all primitives [S4, unofficial].
- **Strengths:** Simple to implement; guarantees correct sphere tracing steps; allows for complex scene composition from simple primitives [S2, S8, unofficial].
- **Limitations:** Sharp creases at intersections; no smooth blending by default; performance degrades with large numbers of primitives without acceleration structures [S2, S8, unofficial].
- **When to use:** Ideal for scenes with a moderate number of primitives where sharp geometric unions are desired and performance is not critically constrained by $N$ [S8, unofficial].
- **When not to use:** When smooth transitions between objects are required (use [Smooth SDF Composition](2685-smooth-sdf-composition.md)) or when the scene contains a very large number of primitives without acceleration (use spatial partitioning or other techniques).

## Background
The method is based on the approach described by Iñigo Quilez in his "Distance Functions" resources, which established the use of minimum, maximum, and negation operations for constructive solid geometry with SDFs [S8, unofficial]. It is a foundational technique in ray marching, allowing artists and programmers to create complex 3D scenes using simple mathematical functions rather than triangle meshes [S2, S8, unofficial]. The technique is widely used in real-time graphics, demoscene productions, and procedural generation [S1, S6, S8, unofficial].

## Sources
[S1] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S3] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S4] kuvsamja/lidar-sphere-tracing, https://github.com/kuvsamja/lidar-sphere-tracing, fetched 2026-10-26 (unofficial)
[S5] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S6] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S7] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S8] Ray Marching: Signed Distance Fields, Sphere Tracing, and SDF Shading, https://tutorials.tektite.studio/ray-marching/, fetched 2026-10-27 (unofficial)
