# Woodcock Tracking with Macrocell Majorant

Also called: Woodcock tracker, stochastic ray marching with majorant

## Overview
This method performs volume rendering using Woodcock tracking (null-collision) within coarse mesh elements, utilizing macrocell densities to determine the majorant extinction value [S1, unofficial]. It builds on [Ray-grid traversal](17-ray-grid-traversal.md) by integrating the traversal of a coarse mesh structure to manage the underlying adaptive mesh refinement (AMR) data [S1, unofficial]. Using macrocell majorant values provides a tighter majorant than using only transfer function opacity, reducing null collisions and improving performance [S1, unofficial].

## How it works
The algorithm advances a ray through a coarse mesh element (e.g., a tetrahedron) using stochastic steps determined by a majorant extinction coefficient. It handles sub-element traversal by detecting when a step exits the current coarse element, re-traversing to find the new leaf element, and updating the exit distance.

**Inputs:**
*   `ray`: Ray structure with origin `ray.org`, direction `ray.dir`, and current distance `ray.t` (initialized to 0 or entry point).
*   `tMax`: Maximum ray distance for traversal. If exceeded, the ray is terminated [S1, unofficial].
*   `coarseTetID`: Identifier for the current coarse mesh element (e.g., tetrahedron) [S1, unofficial].
*   `macrocellMajorant`: Precomputed majorant value for the coarse element [S1, unofficial].
*   `tf`: Transfer function object providing `opacity`, `getColor(scalar)`, and `inRange(scalar)` [S1, unofficial].

**Outputs:**
*   `color`: The accumulated color if a collision is accepted; otherwise, the function returns `false` (transparent/background) [S1, unofficial].

**Data Structures:**
*   `sampleTetID`: Integer identifier for the current leaf tetrahedron [S1, unofficial].
*   `tetVertices`: Array of 4 vertices defining the current leaf tetrahedron [S1, unofficial].
*   `tExit`: Floating-point distance along the ray to the exit face of the current leaf tetrahedron [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize Majorant:**
    Compute the effective majorant for the coarse element:
    $$ \text{majorant} \leftarrow \text{tf.opacity} \times \text{macrocellMajorant} $$
    If $\text{majorant} \le 0$, return `false` (skip ray) [S1, unofficial].

2.  **Initialize Traversal State:**
    Set the exit threshold for the current tetrahedron to slightly less than the current ray distance to ensure the first sample is processed:
    $$ t_{\text{Exit}} \leftarrow \text{nextafterf}(ray.t, -\infty) $$
    Initialize `sampleTetID` to 0 and `tetVertices` to an empty set [S1, unofficial].

3.  **Stochastic Step Loop:**
    Iterate indefinitely:
    a.  **Advance Ray:** Generate a random number $u \in (0, 1)$. Update the ray distance:
        $$ ray.t \leftarrow ray.t - \frac{\ln(1 - u)}{\text{majorant}} $$
        If $ray.t \ge t_{\text{Max}}$, return `false` [S1, unofficial].
    b.  **Sample Point:** Calculate the 3D sample position:
        $$ \text{samplePoint} \leftarrow ray.org + ray.t \times ray.dir $$
    c.  **Boundary Check:** If $ray.t > t_{\text{Exit}}$:
        i.  Traverse the mesh to find the new leaf element containing `samplePoint`:
            $$ (\text{sampleTetID}, \text{tetVertices}) \leftarrow \text{traverse}(\text{samplePoint}, \text{coarseTetID}) $$
        ii. Calculate the new exit distance for the new leaf element:
            $$ t_{\text{Exit}} \leftarrow \text{intersectFaces}(ray.t, \text{samplePoint}, ray.dir, \text{tetVertices}) $$
    d.  **Sample Data:** Retrieve the scalar value at `samplePoint` using `sampleData(samplePoint, sampleTetID, tetVertices)` [S1, unofficial].
    e.  **Range Check:** If the scalar is not in the transfer function range (`!tf.inRange(scalar)`), `continue` to the next iteration [S1, unofficial].
    f.  **Get Color:** Compute the color from the transfer function:
        $$ \text{color} \leftarrow \text{tf.getColor}(scalar) $$
    g.  **Acceptance Test:** Generate a random number $v \in (0, 1)$. If:
        $$ \text{color.opacity} \times \text{tf.opacity} \ge v \times \text{majorant} $$
        Return `color` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def woodcock_elem(ray, tMax, coarseTetID, macrocellMajorant, tf):
    # 1. Initialize Majorant
    majorant = tf.opacity * macrocellMajorant
    if majorant <= 0:
        return False

    # 2. Initialize Traversal State
    # nextafterf returns the next representable floating-point value below ray.t
    tExit = nextafterf(ray.t, -float('inf'))
    sampleTetID = 0
    tetVertices = {}

    # 3. Stochastic Step Loop
    while True:
        # a. Advance Ray
        u = random()
        ray.t = ray.t - log(1 - u) / majorant
        
        if ray.t >= tMax:
            return False

        # b. Sample Point
        samplePoint = ray.org + ray.t * ray.dir

        # c. Boundary Check
        if ray.t > tExit:
            # Traverse to find new leaf element
            sampleTetID, tetVertices = traverse(samplePoint, coarseTetID)
            # Update exit distance for the new leaf
            tExit = intersect_faces(ray.t, samplePoint, ray.dir, tetVertices)

        # d. Sample Data
        scalar = sample_data(samplePoint, sampleTetID, tetVertices)

        # e. Range Check
        if not tf.in_range(scalar):
            continue

        # f. Get Color
        color = tf.get_color(scalar)

        # g. Acceptance Test
        v = random()
        if color.opacity * tf.opacity >= v * majorant:
            return color
```

**Parameters:**
*   `tMax`: Maximum ray distance. Typical values depend on the scene bounds [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Majorant:** If `majorant` is zero or negative, the ray is skipped immediately to avoid division by zero or infinite steps [S1, unofficial].
*   **Boundary Crossing:** When `ray.t` exceeds `tExit`, the algorithm must re-traverse the mesh. Failure to update `tExit` correctly can lead to missing collisions or processing the same element multiple times [S1, unofficial].
*   **Performance Degradation:** On datasets with shallow trees or specific structures (e.g., Agulhas), macrocell optimizations can degrade performance. This occurs because processing must restart from the beginning of the next element to avoid overstepping into denser regions and missing collisions [S1, unofficial].

## Subtopics

### [Woodcock Tracking with Coarse Mesh and Macrocells](1048-woodcock-tracking-with-coarse-mesh-and-macrocells.md)
This method performs volume rendering using Woodcock tracking (null-collision) within coarse mesh elements, utilizing macrocell densities to determine the majorant extinction value. It builds on [Woodcock Tracking with Macrocell Majorant](986-woodcock-tracking-with-macrocell-majorant.md) by integrating the traversal of a coarse mesh structure to manage the underlying adaptive mesh refinement (AMR) data. Using macrocell majorant values provides a tighter majorant than using only transfer function opacity, reducing null collisions and improving performance.

## Operating characteristics
*   **Performance:** Macrocell optimization improves Woodcock performance on most datasets by providing tighter majorants, reducing rejected samples [S1, unofficial].
*   **Limitations:** On the Agulhas dataset, using macrocell optimizations for Woodcock tracking decreases performance [S1, unofficial]. This is because processing must restart from the beginning of the next element when using macrocell values to avoid overstepping into denser regions and missing collisions [S1, unofficial].
*   **Comparison:** Turning on the local majorant optimization almost always increases performance for the Woodcock renderer, as it allows for tighter majorants, decreasing the number of null collisions [S1, unofficial].

## Background
The method is described in the context of direct volume rendering of tree-based tetrahedral adaptive mesh refinement (AMR) data [S1, unofficial]. It addresses the challenge of efficiently rendering unstructured mesh data by combining stochastic ray marching with hierarchical mesh traversal [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
