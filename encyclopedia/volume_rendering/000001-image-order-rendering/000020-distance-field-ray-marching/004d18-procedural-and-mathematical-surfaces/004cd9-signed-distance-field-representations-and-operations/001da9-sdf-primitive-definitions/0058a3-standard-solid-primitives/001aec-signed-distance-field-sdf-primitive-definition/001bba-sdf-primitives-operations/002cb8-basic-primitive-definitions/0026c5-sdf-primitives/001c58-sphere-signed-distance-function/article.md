# Sphere Signed Distance Function

Also called: Sphere SDF, SDF Sphere, Sphere SDF Primitive, SDF Sphere Primitive, Signed Distance Function Sphere, Signed Distance Field (SDF) Sphere

## Overview
The Sphere Signed Distance Function (SDF) computes the exact Euclidean distance from a query point in 3D space to the surface of a sphere, returning a negative value if the point is inside and a positive value if outside [S1, S2, S3, S4, S5, S6, S7, S10, S15, S16, unofficial][S17, unofficial][S18][S19, unofficial]. It serves as the fundamental building block for distance-field ray marching, enabling adaptive step sizes that guarantee surface intersection without overshooting when the step size equals the computed distance [S1, S10, S15, unofficial]. It is the 3D analog of the Circle SDF [G].

## How it works

### Inputs and Outputs
*   **Input**: A query point $p \in \mathbb{R}^3$, a sphere center $c \in \mathbb{R}^3$, and a sphere radius $r \in \mathbb{R}_{\ge 0}$ [S16, unofficial]. Optionally, a transformation matrix $M$ if the sphere is defined in a local coordinate system [S8, unofficial][S17, unofficial].
*   **Output**: A scalar signed distance $d \in \mathbb{R}$.
    *   $d < 0$: Point is inside the sphere.
    *   $d = 0$: Point is on the surface.
    *   $d > 0$: Point is outside the sphere [S1, S2, S3, S4, S5, S6, S7, S10, S15, S16, unofficial][S17, unofficial][S18][S19, unofficial].

### Data Structures
The sphere is defined by a center vector and a scalar radius [S1, S3, S4, S5, S10, S11, S12, S14, S15, S19, unofficial]. In Rust implementations, the sphere is packed into a struct with a 4x4 transform matrix (64 bytes), a radius (4 bytes), and padding (12 bytes), totaling 80 bytes aligned to 16 bytes [S8, unofficial].

**Sphere Struct (Rust/Layout)**
*   `transform`: `Mat4` (64 bytes) [S8, unofficial].
*   `radius`: `f32` (4 bytes) [S8, unofficial].
*   `_padding`: `[u32; 3]` (12 bytes) [S8, unofficial].
*   **Total Size**: 80 bytes, `#[repr(C, align(16))]` [S8, unofficial].

In CPU or shader implementations, the center $c$ and radius $r$ are passed as function arguments or uniform variables [S1, S3, S4, S5, S10, S11, S12, S14, S15, S19, unofficial].

### Formula
The signed distance $d(p)$ from a point $p$ to a sphere with center $c$ and radius $r$ is:

$$ d(p) = \| p - c \| - r $$

Where $\| \cdot \|$ denotes the Euclidean norm (L2 norm) [S1, S2, S3, S4, S5, S6, S7, S10, S11, S12, S15, S16, S19, unofficial].

If the point $p$ is given in a local coordinate system and the sphere is defined by a transformation matrix $M$ (where $M$ transforms local space to world space), the point is first transformed:

$$ p_{world} = M \cdot \begin{bmatrix} p_x \\ p_y \\ p_z \\ 1 \end{bmatrix} $$

Then the distance is computed using $p_{world}$ and the sphere's world-space center and radius [S8, unofficial]. Note that if $M$ includes non-uniform scaling, the resulting function is no longer a true Euclidean SDF [S17, unofficial].

### Parameters
*   **Center ($c$)**: The 3D coordinates of the sphere's center. Default is $(0, 0, 0)$ [S1, S2, S7, S10, S11, S16, unofficial]. In learnable models (e.g., NeRFs), this is a trainable parameter [S4, S5, S6, unofficial].
*   **Radius ($r$)**: The radius of the sphere. Default is often $1.0$ or $0.5$ in examples [S1, S7, S9, S10, S11, unofficial]. In learnable models, this is also a trainable parameter [S4, S5, S6, unofficial].
*   **Transformation Matrix ($M$)**: A 4x4 matrix used to position, rotate, and scale the sphere in world space [S8, unofficial][S17, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sphere_sdf(p: tuple, center: tuple, radius: float) -> float:
    """Computes the signed distance from point p to a sphere defined by center and radius."""
    dx = p[0] - center[0]
    dy = p[1] - center[1]
    dz = p[2] - center[2]
    dist_to_center = math.sqrt(dx*dx + dy*dy + dz*dz)
    return dist_to_center - radius

def sphere_sdf_transformed(p: tuple, transform_matrix: list, center_local: tuple, radius: float) -> float:
    """Computes SDF for a sphere defined in local space, transformed by a 4x4 matrix."""
    px, py, pz = p
    w = 1.0
    M = transform_matrix
    wx = M[0][0]*px + M[1][0]*py + M[2][0]*pz + M[3][0]*w
    wy = M[0][1]*px + M[1][1]*py + M[2][1]*pz + M[3][1]*w
    wz = M[0][2]*px + M[1][2]*py + M[2][2]*pz + M[3][2]*w
    
    cx = M[0][0]*center_local[0] + M[1][0]*center_local[1] + M[2][0]*center_local[2] + M[3][0]
    cy = M[0][1]*center_local[0] + M[1][1]*center_local[1] + M[2][1]*center_local[2] + M[3][1]
    cz = M[0][2]*center_local[0] + M[1][2]*center_local[1] + M[2][2]*center_local[2] + M[3][2]
    
    dx = wx - cx
    dy = wy - cy
    dz = wz - cz
    dist_to_center = math.sqrt(dx*dx + dy*dy + dz*dz)
    return dist_to_center - radius

def sphere_trace(origin: tuple, direction: tuple, center: tuple, radius: float, 
                 max_steps: int = 100, max_dist: float = 100.0, eps: float = 1e-4) -> float:
    """
    Marches a ray using the Sphere SDF to find the closest surface intersection.
    Returns the distance along the ray, or float('inf') if no intersection is found.
    """
    t = 0.0
    for _ in range(max_steps):
        p = (origin[0] + t*direction[0],
             origin[1] + t*direction[1],
             origin[2] + t*direction[2])
        d = sphere_sdf(p, center, radius)
        if d < eps:
            return t
        if t > max_dist:
            return float('inf')
        t += max(d, eps)
    return float('inf')
```

### Edge Cases and Pitfalls
*   **Precision**: Near the surface ($d \approx 0$), floating-point precision errors can cause jitter in normal estimation or ray marching termination. Using a small epsilon threshold for termination is standard [S11, S13, unofficial].
*   **Non-Uniform Scaling**: If the transformation matrix $M$ includes non-uniform scaling, the resulting function is no longer a true Euclidean SDF. The distance values will be distorted, leading to incorrect step sizes in ray marching [S17, unofficial].
*   **Zero Radius**: A sphere with radius $0$ is a point. The SDF returns the distance to the center. This is valid but may cause infinite loops in ray marching if not handled (the ray might never "hit" a point unless it passes exactly through it) [G].
*   **Definition Order**: In some scripting languages (e.g., Fennel/Lua), the SDF function must be defined before the object constructor that uses it, due to top-down definition order [S19, unofficial].

## Subtopics

### [Circle Signed Distance Function](2942-circle-signed-distance-function.md)
The Circle SDF computes the signed distance from a 2D query point to the perimeter of a circle. It is the 2D analog of the Sphere SDF [G]. It returns negative values inside, positive outside, and zero on the boundary. It is used when rendering 2D scenes or cross-sections of 3D volumes.

### Distance Estimator
A Distance Estimator is a function that computes the minimum distance from a query point to *any* object in a scene, often returning the distance along with the identifier or color of the closest object [S20, unofficial]. It initializes the minimum distance to a maximum draw distance (e.g., 1000) and iterates through scene primitives, updating the minimum if a closer object is found [S20, unofficial]. This avoids recomputing color lookups for the endpoint later [S20, unofficial]. While the Sphere SDF returns the distance to a *single* sphere, a Distance Estimator aggregates multiple SDFs (including the Sphere SDF) to find the global minimum [S20, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per evaluation. It involves a constant number of arithmetic operations (subtractions, multiplications, one square root) [S1, S3, S7, S12, S19, unofficial].
*   **Memory Complexity**: $O(1)$ if parameters are passed as arguments. $O(1)$ if stored in a struct (80 bytes on GPU as per [S8, unofficial]).
*   **Performance**: Extremely fast. The square root operation is the most expensive part, but modern GPUs have dedicated hardware for `length()` or `sqrt()` [S19, unofficial].
*   **Strengths**:
    *   **Exactness**: Provides the exact Euclidean distance, unlike grid-based SDFs which are approximations [S3, S13, unofficial].
    *   **Adaptive Stepping**: Enables sphere tracing, where the ray step size is equal to the distance to the surface, guaranteeing no surface is missed and minimizing the number of evaluations [S1, S10, S15, unofficial].
    *   **Normal Estimation**: The gradient of the SDF provides the surface normal, which can be computed analytically or via finite differences [S1, S11, unofficial].
*   **Limitations**:
    *   **No Non-Uniform Scaling**: As noted, non-uniform scaling breaks the SDF property [S17, unofficial].
    *   **Single Primitive**: Only represents a sphere. Complex scenes require combining multiple SDFs using Boolean operations (Union, Intersection) [S15, unofficial].
*   **When to Use**: Ideal for ray marching simple geometric shapes, procedural modeling, and collision detection where exact distances are required [G].
*   **When Not to Use**: Not suitable for highly complex, organic shapes where grid-based SDFs or mesh-based ray tracing are more appropriate [G].

## Background
The Sphere SDF is one of the most basic and fundamental Signed Distance Functions, originating from the field of geometric modeling and ray marching [G]. Inigo Quilez compiled a comprehensive list of distance functions for various primitives [S1, S10, S14, unofficial]. The mathematical basis for sphere tracing using SDFs was established by T. A. Nylander in "Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces" (1999) [S17, unofficial]. It is used in neural implicit representations (e.g., NeRFs) as a learnable primitive [S4, S5, S6, unofficial].

## Sources

[S1] Raymarching Distance Fields, http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S3] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S4] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S5] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S6] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S7] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S8] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S9] GitHub - narnia-ai-mason/iso-surface-rendering, https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S10] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S11] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S12] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S13] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S14] Ray Marching with Three.js Guide, https://www.scribd.com/document/882338593/Ray-Marching-in-Three-js, fetched 2026-10-28 (unofficial)
[S15] Volumetric Rendering: Signed Distance Functions - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
[S16] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
[S17] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29 (unofficial)
[S18] Implementing Dual Contouring, http://ngildea.blogspot.com.br/2014/11/implementing-dual-contouring.html, fetched 2026-10-30
[S19] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
