# Screen-Space Ray Casting with Layer Traversal

Also called: Ray Marching through Columns

## Overview
A real-time rendering technique that performs ray casting in screen space by marching pixel by pixel along the projected ray direction to intersect voxel columns and find entry/exit points for volume rendering [S1]. It distinguishes itself from its parent [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md) by explicitly handling multiple surface layers within a column via a layer index, enabling the tracing of multiple refractions and deep ambient occlusion by leveraging full knowledge of surface layers rather than just the front-most surface [S1].

## How it works
**Inputs:** A ray origin and direction (potentially perturbed for refraction), a voxel grid with stored surface layers per column, and a starting depth hint [S1].
**Outputs:** A sequence of intersection events (entry/exit) with their depths and layer indices [S1].

**Data Structures:**
*   **Ray:** Defined by origin $O$ and direction $D$ [G].
*   **Voxel Column:** A vertical stack of surface layers [S1].
*   **Layer Index:** An integer representing the depth layer within a column [S1]. The parity (LSB) of the layer index determines if the ray is entering or exiting the volume material relative to the view direction [S1].

**Parameters:**
*   $z\_sgn$: Sign of the ray direction relative to the view direction. Value is $-1$ if ray marching against view direction, otherwise $1$ [S1].

**Steps:**
1.  **Initialize:** Start at the pixel containing the ray origin. Initialize a `layer` hint variable (typically from the previous pixel's last processed layer) [S1][G].
2.  **Column Intersection:** Determine the current voxel column intersected by the ray [S1].
3.  **Depth Bounds:** Calculate the entry and exit depths of the ray within the current voxel column: $[entry, exit]$ [S1].
4.  **Layer Search:** Find the first layer index after the entry depth, using the `layer` hint as a starting point: $layer = first\_layer\_after(entry, z\_sgn, start\_hint = layer)$ [S1].
5.  **Validity Check:** Determine if a valid layer exists within the column bounds: $layer\_found = (z\_sgn \times layer \le exit \times z\_sgn)$ [S1].
6.  **State Determination:** Determine if the ray is entering the volume at this layer: $entering = ((layer \& 1) \neq (z\_sgn > 0)) == layer\_found$ [S1].
7.  **Marching Loop:** Iterate through layers within the column bounds:
    *   If the current `entering` state differs from the previous `was_inside` state, handle the intersection (e.g., compute contribution, perturb ray for refraction) [S1].
    *   If a layer was found, increment the layer index: $layer += z\_sgn$ [S1].
    *   Toggle the entering state: $entering = !entering$ [S1].
    *   Continue while $z\_sgn \times layer \le exit \times z\_sgn$ [S1].
8.  **Hint Update:** Use the final `layer` value as the `start_hint` for the next pixel to optimize the search [S1].
9.  **Termination:** Terminate when the voxel grid boundary is reached [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# z_sgn = -1 if ray marching against view dir, else 1
layer = initial_hint  # From previous pixel or initialization [G]

column = next_intersected(ray, column)
while column:
    entry, exit = intersection_depths(ray, column)
    layer = first_layer_after(entry, z_sgn, start_hint=layer)
    layer_found = (z_sgn * layer <= exit * z_sgn)
    entering = ((layer & 1) != (z_sgn > 0)) == layer_found
    was_inside = False  # Track state across layers [G]

    # do-while loop simulation
    while True:
        if entering != was_inside:
            # Abort marching, compute ray contribution, perturb ray, restart casting [S1]
            break
        if layer_found:
            layer += z_sgn
        entering = not entering
        if not (z_sgn * layer <= exit * z_sgn):
            break
    column = next_intersected(ray, column)
```

**Edge Cases and Pitfalls:**
*   **Boundary Hits:** Hits may occur on voxel column boundaries in-between stored surface values. These are detected by checking if the ray changes its state between inside and outside when entering a new column (using LSB of layer index) [S1].
*   **Refraction Restart:** When a surface is intersected, marching aborts, the ray's contribution is computed, and ray casting restarts with a new perturbed ray to trace multiple refractions [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of layers intersected per pixel. The use of a depth hint from the previous pixel optimizes the search for the next layer [S1].
*   **Memory Complexity:** Depends on the storage of surface layers per voxel column [S1].
*   **Strengths:** Enables deep ambient occlusion by accessing multiple surface layers. Supports multiple refractions by restarting ray casting with perturbed rays. Efficient layer traversal using hints [S1].
*   **Limitations:** Requires storing multiple layers per voxel column, which increases memory usage compared to single-layer methods [S1].

## Background
This method is described in the context of memory-efficient on-the-fly voxelization of particle data, where it is used to render complex particle systems as volumetric surfaces [S1]. It builds upon the concept of screen-space ray casting and extends it to handle multiple surface layers within a voxel column, facilitating advanced effects like refraction and deep ambient occlusion [S1].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
