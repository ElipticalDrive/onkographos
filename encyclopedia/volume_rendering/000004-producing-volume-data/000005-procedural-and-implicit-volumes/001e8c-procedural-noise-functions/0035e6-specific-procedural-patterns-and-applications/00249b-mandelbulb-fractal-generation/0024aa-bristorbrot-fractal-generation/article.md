# Bristorbrot Fractal Generation

Also called: Bristorbrot

## Overview
Bristorbrot generates a 3D fractal using a specific iterative polynomial mapping in Cartesian coordinates [S1, unofficial]. It differs from its parent, the Mandelbulb, which uses polar coordinate power transformations, by employing a distinct set of quadratic and cubic-like mixing terms that do not rely on spherical harmonics [S1, unofficial].

## How it works
**Inputs:** A 3D point $p$ (the voxel center or ray origin).
**Outputs:** A boolean indicating whether the point belongs to the fractal set (escape time or distance).

**Data Structures:**
- `point3d`: A structure containing three floating-point fields `x`, `y`, `z`.
- `float`: Standard 32-bit or 64-bit floating-point number.

**Parameters:**
- `max_iterations`: Integer, default 30 [S1, unofficial].
- `max_distance`: Float, default 200 [S1, unofficial].

**Algorithm Steps:**
1. Initialize the iteration variable $z$ to the input point $p_{in}$:
   $$z = p_{in}$$
2. Initialize the squared distance $z_{dist}$ to the squared length of $z$:
   $$z_{dist} = x^2 + y^2 + z^2$$
3. Loop $i$ from 0 to `max_iterations` - 1:
   a. Compute the new coordinates $p_{new}$ based on the current $z$:
      $$p_{new}.x = z.x^2 - z.y^2 - z.z^2$$
      $$p_{new}.y = z.y \cdot (2.0 \cdot z.x - z.z)$$
      $$p_{new}.z = z.z \cdot (2.0 \cdot z.x + z.y)$$
   b. Update $z$ by adding the original input point $p_{in}$ to the new coordinates:
      $$z = p_{new} + p_{in}$$
   c. Update the squared distance:
      $$z_{dist} = z.x^2 + z.y^2 + z.z^2$$
   d. Check for escape: If $z_{dist} >$ `max_distance`, break the loop.
4. If the loop completes all `max_iterations` without breaking, the point is considered part of the fractal set.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bristorbrot(p_in, max_iterations=30, max_distance=200):
    z = p_in
    z_dist = z.x**2 + z.y**2 + z.z**2
    
    for i in range(max_iterations):
        # Compute new coordinates
        new_x = z.x**2 - z.y**2 - z.z**2
        new_y = z.y * (2.0 * z.x - z.z)
        new_z = z.z * (2.0 * z.x + z.y)
        
        # Update z = new_p + p_in
        z.x = new_x + p_in.x
        z.y = new_y + p_in.y
        z.z = new_z + p_in.z
        
        # Check escape
        z_dist = z.x**2 + z.y**2 + z.z**2
        if z_dist > max_distance:
            return False # Escaped
            
    return True # Did not escape
```

**Edge Cases and Pitfalls:**
- Precision: Squaring large coordinates can lead to overflow if `max_distance` is set excessively high. The default 200 is safe for standard float ranges.
- Boundaries: The method assumes the input point $p$ is within the bounding box of the fractal. Points far outside will escape immediately.

## Subtopics

### Knot Fractal Generation
A fractal distance function representing a knot shape, constructed by iterating and transforming a base curve [S2, unofficial]. It differs from Bristorbrot by operating on a 2D cross-section (r, ang) extended into 3D, rather than a direct 3D polynomial map. It is chosen when a knotted topology is desired rather than a bulbous fractal structure.

**Definition:**
Iterates $n$ times to refine the distance to a knot shape [S2, unofficial].

**Steps:**
1. Initialize $r = \sqrt{p.x^2 + p.z^2}$, $ang = \text{atan2}(p.z, p.x)$, $y = p.y$, and distance $d = \infty$ [S2, unofficial].
2. Loop $n$ from 0 to `ITERATIONS` - 1:
   a. Create a temporary point $p_{temp} = (r, y, ang + 2.0 \cdot \pi \cdot n)$ [S2, unofficial].
   b. Subtract `offset.z` from $p_{temp}.x$ [S2, unofficial].
   c. Calculate intermediate values:
      $$ra = (p_{temp}.z + \text{time}) \cdot \frac{\text{clamp}.x}{\text{clamp}.z}$$
      $$raz = p_{temp}.z \cdot \frac{\text{clamp}.y}{\text{clamp}.z}$$
   d. Update distance:
      $$d = \min\left(d, \sqrt{(p_{temp}.x - (\text{offset}.y \cdot \cos(ra) + \text{offset}.z))^2 + (p_{temp}.y - (\text{offset}.y \cdot \sin(raz) + \text{offset}.z))^2} - \text{offset}.x\right)$$

**Parameters:**
- `ITERATIONS`: 3 [S2, unofficial].
- `offset`: `vec3(0.07, 0.29, 0.43)` [S2, unofficial].
- `clamp`: `vec3(-2.0, -4.0, 3.0)` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(I)$ per voxel, where $I$ is `max_iterations`.
- **Memory Complexity:** $O(1)$ auxiliary space per voxel evaluation.
- **Performance:** The Bristorbrot iteration involves several multiplications and additions per step. It is generally faster than high-power Mandelbulbs (e.g., power 8+) but slower than simple quadratic Julia sets due to the 3D mixing terms.
- **Strengths:** Produces intricate, organic structures with distinct "bristle" or "spike" features compared to the smoother Mandelbulb.
- **Limitations:** The escape radius of 200 is arbitrary; tuning it may reveal more detail or cause overflow. The specific polynomial mapping may not converge for all initial points in the same way as standard Mandelbrot sets.

## Background
The Bristorbrot is a variant of 3D fractal generation explored in the context of procedural volume rendering and voxel-based visualization [S1, unofficial]. It was implemented in the `holographic_fractals` repository as an alternative to the Mandelbulb, offering a different topological structure through its specific Cartesian polynomial iteration [S1, unofficial].

## Sources
[S1] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
[S2] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
