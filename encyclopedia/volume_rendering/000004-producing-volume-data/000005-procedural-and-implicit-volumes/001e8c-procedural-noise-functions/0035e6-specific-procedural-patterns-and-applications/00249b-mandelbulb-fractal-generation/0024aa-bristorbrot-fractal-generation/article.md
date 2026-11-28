# Bristorbrot Fractal Generation

Also called: Bristorbrot

## Overview
Bristorbrot is a 3D fractal generation method that produces a volumetric structure using a specific iterative polynomial mapping in Cartesian coordinates [S1, unofficial]. It differs from the Mandelbulb [3327-mandelbulb-fractal-generation.md] by employing a distinct algebraic iteration rule that does not rely on polar coordinate power transformations [S1, unofficial].

## How it works
**Inputs:** A 3D point $p$ (voxel center or ray origin), `max_iterations` (integer), `max_distance` (float) [S1, unofficial].
**Outputs:** An integer indicating the iteration count at which the point escapes, or `max_iterations` if the point remains inside the set [S1, unofficial].

**Data Structures:**
* `point3d`: A structure containing three floating-point fields: `x`, `y`, `z` [G].
* `float`: Standard 32-bit or 64-bit floating-point number [G].

**Parameters:**
* `max_iterations`: Default 30 [S1, unofficial].
* `max_distance`: Default 200 [S1, unofficial].

**Steps:**
1. Initialize the iteration state $z$ to the input point $p_{in}$:
   $$z = p_{in}$$ [S1, unofficial].
2. Initialize the squared distance accumulator $z_{dist}$:
   $$z_{dist} = z.x^2 + z.y^2 + z.z^2$$ [S1, unofficial].
3. Loop $i$ from 0 to `max_iterations` - 1:
   a. Compute the new coordinates $p_{new}$ based on the current $z$:
   $$p_{new}.x = z.x^2 - z.y^2 - z.z^2$$
   $$p_{new}.y = z.y \cdot (2.0 \cdot z.x - z.z)$$
   $$p_{new}.z = z.z \cdot (2.0 \cdot z.x + z.y)$$ [S1, unofficial].
   b. Update $z$ by adding the input point $p_{in}$ to the new coordinates:
   $$z = p_{new} + p_{in}$$ [S1, unofficial].
   c. Update the squared distance:
   $$z_{dist} = z.x^2 + z.y^2 + z.z^2$$ [S1, unofficial].
   d. Check for escape: If $z_{dist} > \text{max\_distance}$, break the loop [S1, unofficial].
4. If the loop completes all `max_iterations` without breaking, the point is considered part of the fractal set [S1, unofficial].

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
        
        # Update distance
        z_dist = z.x**2 + z.y**2 + z.z**2
        
        # Check escape
        if z_dist > max_distance:
            return i
            
    return max_iterations
```

**Edge Cases and Pitfalls:**
* **Degenerate Input:** If $p_{in}$ is $(0,0,0)$, $z$ remains $(0,0,0)$ and the point is always inside [G].
* **Precision:** Not given in the sources. [S1, unofficial]
* **Boundaries:** Not given in the sources. [S1, unofficial]

## Subtopics

### Knot Fractal Generation
A fractal distance function representing a knot shape, constructed by iterating and transforming a base curve [S2, unofficial]. It differs from Bristorbrot by using iterative angular transformations and vector offsets to compute a distance field rather than a Cartesian polynomial mapping [S2, unofficial]. Choose this when a knotted topology is required [S2, unofficial].

**Definition:**
Generates a distance field for a knot fractal using iterative angular transformations [S2, unofficial].

**Steps:**
1. Initialize `r = length(p.xz)`, `ang = atan(p.z, p.x)`, `y = p.y`, `d = HUGEVAL` [S2, unofficial].
2. Loop `n` from 0 to `ITERATIONS` - 1:
   a. Create `p = vec3(r, y, ang + 2.0 * PI * float(n))` [S2, unofficial].
   b. Subtract `offset.z` from `p.x` [S2, unofficial].
   c. Calculate `ra = (p.z + time) * clamp.x / clamp.z` and `raz = p.z * clamp.y / clamp.z` [S2, unofficial].
   d. Update distance `d = min(d, length(p.xy - vec2(offset.y*cos(ra)+offset.z, offset.y*sin(raz)+offset.z)) - offset.x)` [S2, unofficial].

**Parameters:**
* `ITERATIONS`: 3 [S2, unofficial].
* `offset`: `vec3(0.07, 0.29, 0.43)` [S2, unofficial].
* `clamp`: `vec3(-2.0, -4.0, 3.0)` [S2, unofficial].

## Operating characteristics
* **Time Complexity:** Not given in the sources. [S1, unofficial]
* **Memory Complexity:** Not given in the sources. [S1, unofficial]
* **Performance:** Not given in the sources. [S1, unofficial]
* **Strengths:** Not given in the sources. [S1, unofficial]
* **Limitations:** Not given in the sources. [S1, unofficial]

## Background
The Bristorbrot fractal is defined in the context of procedural volume rendering experiments, specifically within the `holographic_fractals` repository [S1, unofficial].

## Sources
[S1] jtomori/holographic_fractals, https://github.com/jtomori/holographic_fractals, fetched 2026-10-26 (unofficial)
[S2] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
