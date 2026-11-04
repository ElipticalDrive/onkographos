# Screen-Space Ray Casting with Layer Traversal

Also called: Ray Marching through Columns

## Overview
A real-time rendering technique that performs ray casting in screen space by marching pixel by pixel along the projected ray direction to intersect voxel columns and find entry/exit points for volume rendering [S1, unofficial]. It distinguishes itself from its parent [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md) by explicitly handling multiple surface layers within a column via a layer index, enabling the tracing of multiple refractions and deep ambient occlusion by leveraging full knowledge of surface layers rather than just the front-most surface [S1, unofficial].

## How it works
**Inputs:** A ray origin and direction (potentially perturbed for refraction), a voxel grid with stored surface layers per column, and a starting depth hint.
**Outputs:** A sequence of intersection events (entry/exit) with their depths and layer indices.

**Data Structures:**
*   **Ray:** Defined by origin $O$ and direction $D$.
*   **Voxel Column:** A vertical stack of surface layers. Each layer is identified by an integer index. The parity (LSB) of the layer index determines if the ray is entering or exiting the volume material relative to the view direction.
*   **Layer Index:** An integer representing the depth layer within a column.

**Parameters:**
*   $z\_sgn$: Sign of the ray direction relative to the view direction. Value is $-1$ if ray marching against view direction, otherwise $1$ [S1, unofficial].

**Steps:**
1.  **Initialize:** Start at the pixel containing the ray origin. Initialize a `layer` hint variable (typically from the previous pixel's last processed layer) [S1, unofficial].
2.  **Column Intersection:** Determine the current voxel column intersected by the ray. If no column is intersected, terminate [S1, unofficial].
3.  **Depth Bounds:** Calculate the entry and exit depths of the ray within the current voxel column: $[entry, exit]$ [S1, unofficial].
4.  **Layer Search:** Find the first layer index after the entry depth, using the `layer` hint as a starting point: $layer = first\_layer\_after(entry, z\_sgn, start\_hint = layer)$ [S1, unofficial].
5.  **Validity Check:** Determine if a valid layer exists within the column bounds: $layer\_found = (z\_sgn \times layer \le exit \times z\_sgn)$ [S1, unofficial].
6.  **State Determination:** Determine if the ray is entering the volume at this layer: $entering = ((layer \& 1) \neq (z\_sgn > 0)) == layer\_found$ [S1, unofficial].
7.  **Marching Loop:** Iterate through layers within the column bounds:
    *   If the current `entering` state differs from the previous `was_inside` state, handle the intersection (e.g., compute contribution, perturb ray for refraction) [S1, unofficial].
    *   If a layer was found, increment the layer index: $layer += z\_sgn$ [S1, unofficial].
    *   Toggle the entering state: $entering = !entering$ [S1, unofficial].
    *   Continue while $z\_sgn \times layer \le exit \times z\_sgn$ [S1, unofficial].
8.  **Hint Update:** Use the final `layer` value as the `start_hint` for the next pixel to optimize the search [S1, unofficial].
9.  **Termination:** Terminate when the voxel grid boundary is reached or no more columns are intersected [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# z_sgn = -1 if ray marching against view dir, else 1
layer = initial_hint # From previous pixel or initialization

while column = next_intersected(ray, column):
    entry, exit = intersection_depths(ray, column)
    
    # Find first layer after entry, using current layer as hint
    layer = first_layer_after(entry, z_sgn, start_hint=layer)
    
    # Check if the layer is within the column bounds
    layer_found = (z_sgn * layer <= exit * z_sgn)
    
    # Determine if this layer represents an entry into the volume
    # (layer & 1) != (z_sgn > 0) checks parity against view direction
    entering = ((layer & 1) != (z_sgn > 0)) == layer_found
    
    was_inside = False # Track state across layers
    
    while z_sgn * layer <= exit * z_sgn:
        if entering != was_inside:
            handle_intersection(...)
            # If tracing refractions, perturb ray and restart from this pixel
            # For simple layer traversal, just continue to next layer
        
        if layer_found:
            layer += z_sgn
        
        entering = not entering
        was_inside = entering # Update state for next iteration
        
    # layer is now the hint for the next pixel
```

**Edge Cases and Pitfalls:**
*   **Boundary Hits:** Hits may occur on voxel column boundaries in-between stored surface values. These are detected by checking if the ray changes its state between inside and outside when entering a new column (using LSB of layer index) [S1, unofficial].
*   **Precision:** Depth comparisons must handle floating-point precision issues, especially when entry/exit points are very close to layer boundaries.
*   **Refraction Restart:** When a surface is intersected and refraction is traced, the ray casting must restart with a new perturbed ray, potentially re-evaluating the current pixel's column [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of layers intersected per pixel. The use of a depth hint from the previous pixel optimizes the search for the next layer, reducing the average cost per pixel [S1, unofficial].
*   **Memory Complexity:** Depends on the storage of surface layers per voxel column. The method itself is memory-efficient as it processes columns on-the-fly [S1, unofficial].
*   **Strengths:** Enables deep ambient occlusion by accessing multiple surface layers. Supports multiple refractions by restarting ray casting with perturbed rays. Efficient layer traversal using hints [S1, unofficial].
*   **Limitations:** Requires storing multiple layers per voxel column, which increases memory usage compared to single-layer methods. Performance degrades with high layer density or complex refraction paths [S1, unofficial].

## Background
This method is described in the context of memory-efficient on-the-fly voxelization of particle data, where it is used to render complex particle systems as volumetric surfaces [S1, unofficial]. It builds upon the concept of screen-space ray casting and extends it to handle multiple surface layers within a voxel column, facilitating advanced effects like refraction and deep ambient occlusion [S1, unofficial].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26 (unofficial)
