# Plane Signed Distance Function

Also called: Plane SDF, Plane distance, Signed Distance Function (SDF) for Plane, Signed Plane, sdPlane

## Overview
The Plane Signed Distance Function (SDF) computes the exact Euclidean distance from a query point to an infinite plane defined by a normal vector and an offset [S1, S3, unofficial]. It serves as a fundamental building block in distance-field ray marching, often representing ground planes or infinite boundaries [S5]. Unlike its siblings such as the [Sphere SDF](1237-sphere-signed-distance-function.md) or [Box SDF](1092-signed-box-distance-function.md), which define bounded volumes, the plane is unbounded [S3, unofficial]. It is distinguished by its computational simplicity, requiring only a single dot product and addition, making it the most efficient primitive for infinite planar surfaces [S2, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A query point $\mathbf{p} \in \mathbb{R}^3$ and plane definition parameters [S3, unofficial].
- **Output**: A scalar signed distance $d \in \mathbb{R}$ [S3, unofficial].

### Data Structures
The plane is defined by two components:
1. **Normal ($\mathbf{n}$)**: A 3D unit vector $(n_x, n_y, n_z)$ perpendicular to the plane [S5].
2. **Offset ($h$ or $d_0$)**: A scalar representing the distance from the origin to the plane along the normal [S1, unofficial].
Alternatively, the plane can be represented as a 4D vector $\mathbf{n}_{eq} = (n_x, n_y, n_z, n_w)$, where $n_w$ corresponds to the offset term in the plane equation [S2, unofficial].

### Numbered Steps
1. Compute the dot product of the query point $\mathbf{p}$ and the plane normal $\mathbf{n}$ [S1, unofficial].
2. Add the scalar offset $h$ to the dot product result [S3, unofficial].

### Formula
The signed distance $d$ is calculated as:
$$ d = \mathbf{p} \cdot \mathbf{n} + h $$
Where:
- $\mathbf{p}$ is the query point.
- $\mathbf{n}$ is the unit normal vector of the plane [S5].
- $h$ is the offset. The sign convention varies by source:
  - In [S2, unofficial] and [S3, unofficial], $h$ is added directly ($d = \mathbf{p} \cdot \mathbf{n} + h$).
  - In [S4, unofficial], the formula is $f(\mathbf{x}) = \mathbf{x} \cdot \mathbf{n} - h$, where $h$ defines the relative placement with respect to the plane perpendicular to $\mathbf{n}$ intersecting the origin.
  - In [S5], the distance is given as $\mathbf{x} \cdot \mathbf{n} - \mathbf{r} \cdot \mathbf{n}$, where $\mathbf{r}$ is a point on the plane [S5]. This is equivalent to $\mathbf{x} \cdot \mathbf{n} - h'$ where $h' = \mathbf{r} \cdot \mathbf{n}$ [S5].

**Sign Convention**:
- $d > 0$: The point is on the side of the plane towards which the normal $\mathbf{n}$ points.
- $d < 0$: The point is on the opposite side of the normal.
- $d = 0$: The point lies exactly on the plane [S2, unofficial].

### Parameters
- **$\mathbf{n}$**: Must be a unit vector ($\|\mathbf{n}\| = 1$) [S2, S3, S4, unofficial][S5].
- **$h$**: Scalar offset. Typical values depend on scene scale. For a ground plane at $y=0$ with normal $(0,1,0)$, $h=0$ [G]. For a plane at $y=-5$ with normal $(0,1,0)$, $h=5$ (using $d = \mathbf{p} \cdot \mathbf{n} + h$) [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sd_plane(p: Vector3, n: Vector3, h: float) -> float:
    """
    Computes the signed distance from point p to a plane defined by unit normal n and offset h.
    
    Args:
        p: Query point (x, y, z).
        n: Plane normal vector. MUST be normalized.
        h: Offset. The plane equation is effectively n.x*x + n.y*y + n.z*z + h = 0.
           Note: Some sources use minus h. This implementation follows [S2, S3, unofficial].
           
    Returns:
        Signed Euclidean distance. Positive if p is on the side of n, negative otherwise.
    """
    # Guard: Ensure normal is normalized to guarantee Euclidean distance [S2, unofficial]
    # [G] In production, assert or normalize n if not guaranteed by caller.
    
    # Calculate dot product of point and normal
    dot_product = p.x * n.x + p.y * n.y + p.z * n.z
    
    # Add offset
    distance = dot_product + h
    
    return distance
```

### Edge Cases and Pitfalls
1. **Unnormalized Normal**: If $\mathbf{n}$ is not a unit vector, the result is scaled by $\|\mathbf{n}\|$, breaking the "distance" property required for correct ray marching step sizes [S2, unofficial].
2. **Offset Sign Convention**: Sources [S3, unofficial] and [S4, unofficial] use different signs for the offset ($+h$ vs $-h$). The implementer must align the offset definition with the rest of the scene graph or SDF library [S3, S4, unofficial].
3. **Infinite Extent**: The plane is infinite [S3, unofficial]. It does not have edges. If a bounded plane is required, it must be combined with other primitives (e.g., via min/max operations for union/intersection) or clipped by other SDFs [S5].

## Operating characteristics
- **Time Complexity**: $O(1)$ [G].
- **Memory Complexity**: $O(1)$ [G].
- **Strengths**: Extremely fast; exact distance; no convergence issues for infinite surfaces [G].
- **Limitations**: Cannot represent bounded planar shapes (like a square tile) on its own; requires combination with other SDFs for finite planes [S5].
- **When to use**: Infinite ground planes, skyboxes, or as a clipping boundary [G].
- **When not to use**: When a finite planar surface is needed without additional SDF logic [G].
- **Measured Performance**: Not given in the sources [G].
- **Scaling**: Not given in the sources [G].

## Background
The plane SDF is one of the most basic primitives in implicit surface modeling [S5]. It appears in foundational ray marching literature, such as Hart's "Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces" [S5], where it is listed alongside spheres and cylinders. It is also widely implemented in GLSL-based SDF libraries, such as those by Inigo Quilez (referenced in [S2, unofficial]), which popularized the `sdPlane` function in real-time graphics [S2, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S3] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S4] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S5] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
