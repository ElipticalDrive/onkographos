# Unsigned Box Distance Function

Also called: Unsigned Box, udBox

## Overview
The Unsigned Box Distance Function computes the Euclidean distance from a query point to the surface of an axis-aligned box centered at the origin, returning only non-negative values regardless of whether the point lies inside or outside the volume [S1, unofficial]. Unlike its sibling, the Signed Box Distance Function, which distinguishes interior from exterior space, this method treats the box as a solid boundary where internal points are measured by their distance to the nearest face [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: `p` (vec3), the query point in 3D space [S1, unofficial].
- Input: `b` (vec3), the half-extents of the box along the X, Y, and Z axes [S1, unofficial]. The box is centered at $(0,0,0)$ and spans $[-b_x, b_x] \times [-b_y, b_y] \times [-b_z, b_z]$ [S1, unofficial].
- Output: `d` (float), the unsigned Euclidean distance from `p` to the box surface [S1, unofficial].

**Data structures:**
No complex data structures are required. The function operates on standard 3D vectors [G].

**Formula:**
The distance is calculated using the following formula [S1, unofficial]:

$$ d = \|\max(|p| - b, 0)\| $$

Where:
- $|p|$ denotes the component-wise absolute value of vector $p$, i.e., $(|p_x|, |p_y|, |p_z|)$ [G].
- $\max(v, 0)$ denotes the component-wise maximum of vector $v$ and the scalar $0$, i.e., $(\max(v_x, 0), \max(v_y, 0), \max(v_z, 0))$ [G].
- $\|\cdot\|$ denotes the Euclidean norm (length) of the resulting vector [G].

**Step-by-step derivation:**
1. Compute the absolute coordinates of the point: $q = |p|$ [S1, unofficial].
2. Subtract the box half-extents: $r = q - b$ [S1, unofficial].
3. Clamp negative values to zero: $s = \max(r, 0)$ [S1, unofficial]. This step effectively ignores the interior of the box, treating any negative distance (interior) as zero distance to the surface [S1, unofficial].
4. Compute the Euclidean length of the remaining vector: $d = \sqrt{s_x^2 + s_y^2 + s_z^2}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def udBox(p_x, p_y, p_z, b_x, b_y, b_z):
    """
    Computes the unsigned distance from point p to an axis-aligned box
    centered at the origin with half-extents b.
    
    Args:
        p_x, p_y, p_z: Query point coordinates
        b_x, b_y, b_z: Box half-extents
        
    Returns:
        Non-negative Euclidean distance to the box surface.
    """
    # Step 1: Component-wise absolute value of p
    abs_x = abs(p_x)
    abs_y = abs(p_y)
    abs_z = abs(p_z)
    
    # Step 2: Subtract half-extents
    diff_x = abs_x - b_x
    diff_y = abs_y - b_y
    diff_z = abs_z - b_z
    
    # Step 3: Component-wise max with 0 (clamp negatives to zero)
    clamped_x = max(diff_x, 0.0)
    clamped_y = max(diff_y, 0.0)
    clamped_z = max(diff_z, 0.0)
    
    # Step 4: Euclidean norm (length)
    distance = (clamped_x**2 + clamped_y**2 + clamped_z**2)**0.5
    
    return distance
```

**Parameters and typical values:**
- `b`: Must be positive. If any component is zero, the box degenerates into a plane, line, or point [G].
- `p`: Any real-valued 3D coordinate [G].

**Edge cases and pitfalls:**
- **Inside the box:** If the point is strictly inside the box, the intermediate `diff` vector will have negative components [G]. The clamping step sets these to 0 [G]. If the point is inside, the distance returned is the distance to the nearest face (if only one component is negative) or 0 (if the point is strictly inside all boundaries, though the formula $\max(|p|-b, 0)$ yields a vector of zeros only if $|p| \le b$ component-wise, resulting in distance 0) [S1, unofficial]. The formula provided returns 0 if the point is inside the box volume because $\max(|p_i| - b_i, 0) = 0$ for all $i$, leading to a zero vector and thus distance 0 [S1, unofficial].
- **Precision:** Near the box edges and corners, the function is continuous but not differentiable [G]. Floating-point precision issues may arise when $|p_i| \approx b_i$ [G].
- **Degenerate input:** If $b$ contains negative values, the geometric interpretation is invalid [G].

## Subtopics

### Signed Cuboid Distance Function
Evaluates the signed distance from a point to an axis-aligned cuboid, handling both outside and inside distances [S2, unofficial]. It transforms the point using the cuboid's 4x4 matrix, calculates distances to faces, and combines outside and inside distance components [S2, unofficial]. The cuboid struct layout is: `transform` (Mat4, 64 bytes), `dimensions` (Vec3, 12 bytes), `_padding` (u32, 4 bytes), totaling 80 bytes with 16-byte alignment [S2, unofficial]. Construction uses `Cuboid::new(dimensions).translate(v).rotate(v)` [S2, unofficial].

## Operating characteristics
- **Time complexity:** $O(1)$. The calculation involves a fixed number of arithmetic operations (absolute value, subtraction, comparison, square root) [G].
- **Memory complexity:** $O(1)$. No auxiliary data structures are allocated [G].
- **Strengths:** Extremely fast evaluation, suitable for real-time ray marching and collision detection where interior/exterior distinction is not required [G].
- **Limitations:** Does not provide information about whether the point is inside or outside the volume [G]. Not suitable for operations requiring signed distance properties (e.g., smooth unions, boolean operations with SDFs) [G].

## Background
The Unsigned Box Distance Function is a standard primitive in Signed Distance Field (SDF) rendering, often used in GLSL-based ray marchers [S1, unofficial]. It is part of the collection of SDF primitives popularized by the `glsl-sdf-primitives` library [S1, unofficial]. While the signed variant is more common in constructive solid geometry (CSG) operations, the unsigned variant is useful for simple boundary checks or when the interior is treated as empty space [G].

## Sources
[S1] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S2] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
