# Point-Plane Reflection

Also called: Reflect Point Across Plane, Space Reflection

## Overview
Point-Plane Reflection reflects a point $\mathbf{p}$ across a plane defined by a normal vector $\mathbf{n}$ and a scalar offset $c$, modifying $\mathbf{p}$ in place and returning the sign of the signed distance to the plane [S1, unofficial]. It serves as a geometric primitive for spatial transformations and specular path calculations [S1, unofficial][S2, unofficial]. Unlike ray-primitive intersection methods that compute hit points along a ray, this operation transforms a single point relative to an infinite plane [G].

## How it works
**Inputs:**
- $\mathbf{p}$: The point to reflect (3D vector) [S1, unofficial].
- $\mathbf{n}$: The plane normal (3D vector) [S1, unofficial].
- $c$: The plane offset (scalar) [S1, unofficial].

**Outputs:**
- $\mathbf{p}$: The reflected point (modified in place) [S1, unofficial].
- $s$: The sign of the signed distance from the original point to the plane [S1, unofficial].

**Data Structures:**
- Standard 3D floating-point vectors for $\mathbf{p}$ and $\mathbf{n}$ [G].
- Standard floating-point scalar for $c$ and $t$ [G].

**Steps:**
1. Compute the signed distance $t$ from point $\mathbf{p}$ to the plane: $t = \mathbf{p} \cdot \mathbf{n} + c$ [S1, unofficial].
2. Determine the sign of the distance: $s = \text{sign}(t)$ [S1, unofficial].
3. If the point is on the negative side of the plane ($t < 0$), reflect it across the plane: $\mathbf{p} \leftarrow \mathbf{p} - 2t\mathbf{n}$ [S1, unofficial].
4. Return $s$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reflect_point_across_plane(p, n, c):
    # Step 1: Compute signed distance t
    t = p[0]*n[0] + p[1]*n[1] + p[2]*n[2] + c
    
    # Step 2: Determine sign
    if t > 0:
        s = 1
    elif t < 0:
        s = -1
    else:
        s = 0
        
    # Step 3: Reflect if on negative side
    if t < 0:
        p[0] = p[0] - 2 * t * n[0]
        p[1] = p[1] - 2 * t * n[1]
        p[2] = p[2] - 2 * t * n[2]
        
    return s, p
```

**Parameters:**
- $\mathbf{n}$: Typically normalized ($\|\mathbf{n}\| = 1$) so that $t$ represents the true Euclidean distance [G].

**Edge Cases and Pitfalls:**
- **Non-normalized normal:** If $\mathbf{n}$ is not unit length, $t$ scales incorrectly and the reflection magnitude is wrong [G].
- **Precision:** For points very close to the plane ($t \approx 0$), floating-point errors may cause jitter in the sign or position [G].
- **Degenerate Input:** If $\mathbf{n}$ is the zero vector, the plane is undefined [G].

## Subtopics

### Image Method for Specular Reflection
A geometric technique to determine the specular hit point and path length by mirroring the receiver (or source) across the reflecting surface, converting the broken path into a straight line [S2, unofficial]. Unlike angle search methods, it provides a deterministic, closed-form solution for the hit point and angles without iterative optimization [S2, unofficial]. The path length is given by $\ell(h) = \sqrt{\Delta x^2 + (2h - y_S - y_R)^2}$, where $h$ is the wall height, $y_S$ and $y_R$ are source and receiver y-coordinates, and $\Delta x$ is the horizontal distance [S2, unofficial]. The hit point $x_B$ is derived as $x_B = x_S + \frac{(h - y_S)(x_R - x_S)}{2h - y_S - y_R}$ [S2, unofficial]. A key pitfall is that the derivative $\frac{\partial \ell}{\partial h} = \frac{2(2h - y_S - y_R)}{\ell(h)}$ applies only to a nonzero-length path while the same valid reflection branch is retained; if the path topology changes (e.g., hit point moves off the surface), the derivative is invalid [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The method is a fundamental geometric transformation used in computer graphics and ray tracing [G]. It is often employed in the context of specular reflection calculations, where the path of light is modeled by reflecting the receiver or source across the reflecting surface [S2, unofficial]. The specific implementation described, which returns the sign of the distance and conditionally updates the point, appears in the context of marching cubes and signed distance field operations in GLSL [S1, unofficial].

## Sources
[S1] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
