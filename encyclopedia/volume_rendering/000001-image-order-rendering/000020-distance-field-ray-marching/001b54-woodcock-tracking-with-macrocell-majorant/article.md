# Woodcock Tracking with Macrocell Majorant

Also called: Woodcock tracker, stochastic ray marching with majorant

## Overview
This method performs volume rendering using Woodcock tracking (null-collision) within coarse mesh elements, utilizing macrocell densities to determine the majorant extinction value [S1, unofficial]. It builds on [Distance-field ray marching](21-distance-field-ray-marching.md) by replacing deterministic step sizes with stochastic jumps governed by a majorant, and integrates the traversal of a coarse mesh structure to manage the underlying adaptive mesh refinement (AMR) data [S1, unofficial]. Using macrocell majorant values provides a tighter majorant than using only transfer function opacity, reducing null collisions and improving performance [S1, unofficial].

## How it works
The algorithm processes a ray through a coarse tetrahedral element. It maintains the ray's parametric distance $t$ along the direction vector. The core loop advances the ray by a stochastic step size determined by the majorant, samples the scalar field, and performs an acceptance test against the majorant to determine if a collision (interaction) occurs.

**Inputs:**
- `ray`: Ray structure with origin `ray.org`, direction `ray.dir`, and current parametric distance `ray.t`.
- `tMax`: Maximum ray distance for traversal.
- `coarseTetID`: Identifier for the current coarse tetrahedral element.
- `macrocellMajorant`: Pre-computed majorant value for the macrocell containing the element.
- `tf`: Transfer function object providing `inRange(scalar)`, `getColor(scalar)`, and `opacity`.

**Outputs:**
- `color`: The accumulated color if a collision is accepted; otherwise, the function returns `false` (no hit).

**Data Structures:**
- `tetVertices`: Array of 4 vertex positions for the current tetrahedron.
- `sampleTetID`: Identifier for the specific leaf tetrahedron within the coarse element where sampling occurs.

**Steps:**
1. **Initialize Majorant:** Calculate the effective majorant for the current step.
   $$ M = \text{tf.opacity} \times \text{macrocellMajorant} $$
   If $M \le 0$, the ray skips this element; return `false` [S1, unofficial].

2. **Initialize Traversal State:**
   - Set `tExit` to the next representable floating-point number below `ray.t` to ensure strict inequality checks for exit conditions: `tExit ← nextafterf(ray.t, -INFINITY)` [S1, unofficial].
   - Initialize `sampleTetID ← 0` and `tetVertices ← {}` [S1, unofficial].

3. **Stochastic Loop:**
   - **Advance Ray:** Generate a random number $u \in (0, 1)$. Update the ray's parametric distance:
     $$ t_{new} = t_{current} - \frac{\ln(1 - u)}{M} $$
     Update `ray.t` to $t_{new}$ [S1, unofficial].
   - **Check Bounds:** If `ray.t` $\ge$ `tMax`, the ray has exceeded the maximum range; return `false` [S1, unofficial].
   - **Calculate Sample Point:** Compute the 3D position:
     $$ p = \text{ray.org} + \text{ray.t} \times \text{ray.dir} $$
     [S1, unofficial].
   - **Check Element Exit:** If `ray.t` > `tExit`, the ray has exited the current tetrahedron boundary.
     - Traverse the mesh to find the new tetrahedron containing $p$: `sampleTetID, tetVertices ← traverse(p, coarseTetID)` [S1, unofficial].
     - Calculate the exit distance for the new tetrahedron: `tExit ← intersectFaces(ray.t, p, ray.dir, tetVertices)` [S1, unofficial].
   - **Sample Data:** Retrieve the scalar value at $p$ using the current tetrahedron: `scalar ← sampleData(p, sampleTetID, tetVertices)` [S1, unofficial].
   - **Transfer Function Check:** If `scalar` is not in the transfer function's valid range (`!tf.inRange(scalar)`), continue to the next iteration of the loop (null collision due to empty space) [S1, unofficial].
   - **Get Color:** Retrieve the color and opacity from the transfer function: `color ← tf.getColor(scalar)` [S1, unofficial].
   - **Acceptance Test:** Generate a new random number $u' \in (0, 1)$. If:
     $$ \text{color.opacity} \times \text{tf.opacity} \ge u' \times M $$
     then a collision is accepted. Return `color` [S1, unofficial]. Otherwise, continue to the next iteration (null collision) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def woodcock_elem(ray, t_max, coarse_tet_id, macrocell_majorant, tf):
    # 1. Initialize Majorant
    majorant = tf.opacity * macrocell_majorant
    if majorant <= 0:
        return False

    # 2. Initialize Traversal State
    # nextafterf(x, -inf) returns the largest representable floating point number less than x
    t_exit = nextafterf(ray.t, -float('inf'))
    sample_tet_id = 0
    tet_vertices = {}

    # 3. Stochastic Loop
    while True:
        # Advance Ray
        u = random()
        ray.t = ray.t - log(1 - u) / majorant

        # Check Bounds
        if ray.t >= t_max:
            return False

        # Calculate Sample Point
        sample_point = ray.org + ray.t * ray.dir

        # Check Element Exit
        if ray.t > t_exit:
            sample_tet_id, tet_vertices = traverse(sample_point, coarse_tet_id)
            t_exit = intersect_faces(ray.t, sample_point, ray.dir, tet_vertices)

        # Sample Data
        scalar = sample_data(sample_point, sample_tet_id, tet_vertices)

        # Transfer Function Check
        if not tf.in_range(scalar):
            continue

        # Get Color
        color = tf.get_color(scalar)

        # Acceptance Test
        u_prime = random()
        if color.opacity * tf.opacity >= u_prime * majorant:
            return color
```

**Parameters:**
- `tMax`: Maximum ray distance. If exceeded, the ray is terminated [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Agulhas Dataset Performance:** On the Agulhas dataset, using macrocell optimizations for Woodcock tracking decreases performance [S1, unofficial]. This is because processing must restart from the beginning of the next element when using macrocell values to avoid overstepping into denser regions and missing collisions [S1, unofficial].
- **Floating Point Precision:** The use of `nextafterf` for `tExit` is critical to prevent immediate re-entry or infinite loops due to floating-point equality issues at boundaries [S1, unofficial].

## Subtopics

### [Woodcock Tracking with Coarse Mesh and Macrocells](1048-woodcock-tracking-with-coarse-mesh-and-macrocells.md)
This method performs volume rendering using Woodcock tracking (null-collision) within coarse mesh elements, utilizing macrocell densities to determine the majorant extinction value. It builds on [Woodcock Tracking with Macrocell Majorant](986-woodcock-tracking-with-macrocell-majorant.md) by integrating the traversal of a coarse mesh structure to manage the underlying adaptive mesh refinement (AMR) data. Using macrocell majorant values provides a tighter majorant than using only transfer function opacity, reducing null collisions and improving performance.

## Operating characteristics
- **Performance:** Macrocell optimization improves Woodcock performance on most datasets by providing tighter majorants, reducing rejected samples [S1, unofficial]. However, it degrades performance on the Agulhas dataset due to the overhead of restarting element processing to prevent missing collisions in shallow trees [S1, unofficial].
- **Comparison:** In tests on unstructured mesh data, the Woodcock tracker was significantly faster than the standard Ray Marcher (e.g., 29.07s vs 112.51s on one dataset, 26.30s vs 115.71s on another) [S1, unofficial].
- **VRAM Usage:** The Woodcock tracker generally uses less VRAM than the Ray Marcher (e.g., 6,078 MB vs 2,352 MB on one dataset, 762 MB vs 1,524 MB on another) [S1, unofficial].
- **Scaling:** Rendering times for the Ray Marcher grow with smaller step sizes [S1, unofficial]. The Woodcock tracker's performance is less dependent on fixed step sizes but depends on the tightness of the majorant.

## Background
The method is described in the thesis "Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data" [S1, unofficial]. It applies the Woodcock tracking algorithm, a standard technique in Monte Carlo radiation transport, to the context of volume rendering on AMR tetrahedral meshes. The macrocell majorant optimization is a specific contribution to reduce the variance of the null-collision method by providing a tighter upper bound on the extinction coefficient.

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
