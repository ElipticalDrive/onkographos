# Menger Sponge Fractal Generation

Also called: Fractal sponge voxelization, Menger Sponge Fractal, Menger sponge procedural generation, SDF Menger Sponge

## Overview
Generates a 3D representation of a Menger sponge, either as a voxel grid or a Signed Distance Field (SDF), by iteratively subdividing space into $3 \times 3 \times 3$ subcubes and removing those where two or more coordinates fall into the middle third [S1, S2, unofficial]. It differs from its parent [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md), which applies polar coordinate power transformations, by using discrete coordinate modulo arithmetic and axis-aligned box subtraction [S1, S2, unofficial].

## How it works
**Inputs:**
- `p`: 3D point $(x, y, z)$ in world space.
- `ITERATIONS`: Integer number of fractal iterations (typically 3) [S2, unofficial].

**Outputs:**
- `d`: Signed distance from `p` to the Menger sponge surface.
- `filled`: Boolean indicating if `p` is inside the solid material (voxel method).

**Data Structures:**
- Stateless procedural computation; no persistent data structures are required [G].

**Steps:**
1. Initialize distance `d` to the distance of `p` to the initial bounding box: $d = \text{sdBox}(p, \vec{1.0})$ [S2, unofficial].
2. Initialize scale factor $s = 1.0$ [S2, unofficial].
3. Loop $m$ from $0$ to $ITERATIONS - 1$:
   a. Fold space into the first octant: $p \leftarrow |p| - 1.0$ [G].
   b. Calculate local coordinate: $a = \text{mod}(p \cdot s, 2.0) - 1.0$ [S2, unofficial].
   c. Update scale: $s \leftarrow s \cdot 3.0$ [S2, unofficial].
   d. Calculate absolute local coordinate: $r = |1.0 - 3.0 \cdot |a||$ [S2, unofficial].
   e. Calculate corner distances: $d_a = \max(r_x, r_y)$, $d_b = \max(r_y, r_z)$, $d_c = \max(r_z, r_x)$ [S2, unofficial].
   f. Calculate distance contribution: $c = (\min(d_a, d_b, d_c) - 1.0) / s$ [S2, unofficial].
   g. Update global distance: $d = \max(d, c)$ [S2, unofficial].
4. For voxel occupancy tracking, initialize `filled = true` [S1, unofficial]. For each iteration $k$, evaluate $\lfloor p_x / 3^k \rfloor \bmod 3 = 1 \lor \lfloor p_y / 3^k \rfloor \bmod 3 = 1 \lor \lfloor p_z / 3^k \rfloor \bmod 3 = 1$ [S1, unofficial]. Sum the boolean results; if sum $> 1$, set `filled = false` and break [S1, unofficial].
5. Return `d` for SDF ray marching, or `filled` with color/alpha for voxel traversal [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def menger_sponge_sdf(p, iterations=3):
    """
    Computes the Signed Distance Field for a Menger Sponge.
    p: vec3, point in space
    iterations: int, number of recursive steps
    Returns: float, signed distance
    """
    # Initialize distance to the bounding box (half-extent 1.0)
    d = max(max(abs(p.x) - 1.0, abs(p.y) - 1.0), abs(p.z) - 1.0)
    
    s = 1.0
    for _ in range(iterations):
        # Fold space into the first octant for symmetry [G]
        p = abs(p) - 1.0
        
        # Calculate local coordinate in the 3x3x3 grid
        a = ((p * s) % 2.0) - 1.0
        
        # Update scale for next iteration
        s *= 3.0
        
        # r = abs(1.0 - 3.0 * abs(a))
        r = abs(1.0 - 3.0 * abs(a))
        
        # Calculate corner distances
        da = max(r.x, r.y)
        db = max(r.y, r.z)
        dc = max(r.z, r.x)
        
        # Calculate distance contribution
        c = (min(da, min(db, dc)) - 1.0) / s
        
        # Update distance
        d = max(d, c)
        
    return d
```

**Parameters:**
- `ITERATIONS`: Number of fractal iterations. Typical value: 3 [S2, unofficial].

**Edge Cases:**
- Precision: Floating point modulo operations can accumulate errors at high iteration counts [S1, S2, unofficial].
- Boundaries: Points exactly on the boundary of removed cubes may have distance 0 [S2, unofficial].
- Degenerate Input: High iteration counts increase computational cost linearly with `ITERATIONS` [S2, unofficial].

## Subtopics

### [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md)
Generates a 3D fractal by iteratively applying a polar coordinate power transformation to a complex 3D point.

### [Bristorbrot Fractal Generation](3328-bristorbrot-fractal-generation.md)
Generates a 3D fractal using a different iterative formula.

## Operating characteristics
- **Time Complexity:** $O(ITERATIONS)$ per evaluation step [S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Strengths:** Simple to implement, no pre-computed data structures required, suitable for real-time ray marching [S2, unofficial].
- **Limitations:** High iteration counts lead to visual artifacts due to floating point precision [S1, S2, unofficial].
- **When to use:** When a procedural, infinite-resolution fractal is needed without storing large voxel grids [S2, unofficial].
- **When not to use:** When exact boolean operations or CSG with other primitives are required, as the SDF is an approximation [S2, unofficial].

## Background
The Menger sponge is a classic fractal curve first described by Karl Menger in 1926 [S1, S2, unofficial]. It is a 3D generalization of the Cantor set and the Sierpinski carpet. The procedural generation method described here is a common implementation in computer graphics for real-time rendering [S1, S2, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
