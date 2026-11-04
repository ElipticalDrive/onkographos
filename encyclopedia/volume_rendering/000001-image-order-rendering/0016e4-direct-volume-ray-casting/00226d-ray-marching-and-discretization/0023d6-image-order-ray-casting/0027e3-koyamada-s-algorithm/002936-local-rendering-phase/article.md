# Local Rendering Phase
Also called: Local subvolume rendering, Processor-local rendering

## Overview
The Local Rendering Phase is the core computation step in the Object-Space (OS) parallel implementation of Koyamada's Algorithm [S1, unofficial]. It solves the problem of redundant ray-segment generation that occurs when local clusters on a single processor are treated as independent subvolumes. By treating the set of local clusters assigned to a processor as a single **local subvolume**, it ensures that ray-segments crossing pseudo-boundary faces (shared between local clusters) are composed into a single continuous segment rather than being recomputed [S1, unofficial]. This phase builds on [Koyamada's Algorithm](4148-koyamada-s-algorithm.md) by handling the spatial decomposition constraints within a single processing unit.

## How it works
**Inputs:**
*   The set of local clusters $C_p$ assigned to processor $P_i$.
*   The `CellArray` structures containing cell geometry and connectivity for these clusters.
*   The `RayBuffer` structure for the current pixel row/column being processed.

**Outputs:**
*   Composited color and opacity values for ray-segments exiting the local subvolume, stored in the `RayBuffer`.

**Data Structures:**
*   **CellArray**: Contains cell data. Each cell $c_i$ has neighbor references.
*   **RayBuffer**: A linked-list structure indexed by pixel location. Each entry stores a list of ray-segments sorted by their exit $z$-value.

**Steps:**
1.  **Concurrent Traversal**: Each processor $P_i$ concurrently traverses the cells of the `CellArray` structures of its local clusters $C_p$ [S1, unofficial].
2.  **Face Classification**: For each cell $c_i$ in $C_p$, check each face $f$:
    *   Identify $f$ as an **external face** if cell $c_i$ does not have a neighbor cell through face $f$ [S1, unofficial].
    *   Identify $f$ as a **boundary face** if cell $c_i$ is neighbor to a cell $c_j$ through face $f$ such that $c_j$ belongs to a non-local cluster of $P_i$ (i.e., $c_j \in C_q$ where $C_q$ is not in $P_i$'s set) [S1, unofficial].
    *   Faces shared between two local clusters of $P_i$ are **pseudo-boundary faces** and are *not* treated as external or boundary faces for ray generation purposes [S1, unofficial].
3.  **Ray-Segment Generation**: Scan-convert each identified external and boundary face. Generate a ray-segment for each pixel covered by the face [S1, unofficial].
4.  **Ray Traversal**: Follow each generated ray-segment through the local subvolume according to Koyamada's algorithm [S1, unofficial].
    *   The traversal uses the connectivity information preserved in `ClusterData` structures, allowing the ray to pass through multiple local clusters seamlessly [S1, unofficial].
    *   The ray continues until it exits from a **bf-external face** (boundary face that is also external) or a **bf-boundary face** (boundary face that is not external, i.e., exits the local subvolume to a non-local cluster) [S1, unofficial].
5.  **Buffer Insertion**: When a ray-segment exits the local subvolume, insert its composited color and opacity values into the linked-list of the respective pixel location in the `RayBuffer` structure.
    *   Insertion must be in **sorted order** according to the ray-segment's exit $z$-value [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_rendering_phase(processor_id, local_clusters, ray_buffer):
    """
    Renders the local subvolume defined by local_clusters for processor_id.
    """
    # Step 1 & 2: Traverse cells and identify ray origins
    ray_origins = []
    
    for cluster in local_clusters:
        for cell in cluster.cells:
            for face in cell.faces:
                # Check if face is external (no neighbor)
                if not face.has_neighbor():
                    ray_origins.append((cell, face, 'external'))
                # Check if face is boundary (neighbor exists but is non-local)
                elif face.has_neighbor():
                    neighbor_cluster = face.neighbor.cluster_id
                    if neighbor_cluster not in local_clusters:
                        ray_origins.append((cell, face, 'boundary'))
                    # Else: pseudo-boundary, skip ray generation
    
    # Step 3 & 4: Generate and trace ray-segments
    for cell, face, face_type in ray_origins:
        # Scan-convert face to generate pixel coordinates
        pixels = scan_convert_face(face)
        
        for pixel in pixels:
            # Generate ray-segment starting at pixel, through face
            ray_segment = generate_ray_segment(pixel, face, cell)
            
            # Trace through local subvolume using Koyamada's algorithm
            # The traversal respects connectivity across cluster boundaries
            exit_face, exit_z = trace_ray_through_local_subvolume(ray_segment, local_clusters)
            
            # Step 5: Insert into RayBuffer
            if exit_face is not None:
                composited_color = ray_segment.composited_color
                composited_opacity = ray_segment.composited_opacity
                
                # Insert into linked-list sorted by exit_z
                ray_buffer.insert_sorted(pixel, exit_z, composited_color, composited_opacity)

def trace_ray_through_local_subvolume(ray, local_clusters):
    """
    Traces a ray through the union of local_clusters.
    Returns (exit_face, exit_z) or (None, None) if ray stays within.
    """
    current_cell = ray.start_cell
    current_face = ray.start_face
    
    while True:
        # Move to next cell along ray direction
        next_cell = current_cell.get_neighbor_in_direction(ray.direction)
        
        # If no neighbor, ray exited the local subvolume
        if next_cell is None:
            return (current_face, ray.z_at_exit)
        
        # If neighbor exists, check if it is part of the local subvolume
        if next_cell.cluster_id in local_clusters:
            # Continue traversal within local subvolume
            current_cell = next_cell
            # Update ray segment composition
            ray.compose(next_cell)
            current_face = next_cell.get_exit_face(ray.direction)
        else:
            # Neighbor is non-local, so we exit the local subvolume
            return (current_face, ray.z_at_exit)
```

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Virtual Non-Convexities**: Object-Space (OS) decomposition may create local subvolumes with virtual non-convexities, even if the global dataset is convex [S1, unofficial]. This increases the number of ray-segments generated per pixel compared to the sequential algorithm [S1, unofficial].
*   **RayBuffer Necessity**: Due to virtual non-convexities, multiple ray-segments may exit the local subvolume for a single pixel. The `RayBuffer` structure is crucial for handling this multiplicity and maintaining correct depth sorting [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Depends on the `RayBuffer` size, which scales with the number of ray-segments per pixel. Virtual non-convexities can increase this number [S1, unofficial].
*   **Strengths**: Eliminates redundant computation on pseudo-boundary faces by treating local clusters as a single subvolume [S1, unofficial].
*   **Limitations**: Virtual non-convexities from decomposition can lead to more ray-segments per pixel than in the sequential case, requiring robust `RayBuffer` management [S1, unofficial].
*   **When to use**: When implementing OS parallel direct volume rendering using Koyamada's Algorithm [S1, unofficial].

## Background
The Local Rendering Phase was introduced in the context of parallel direct volume rendering of unstructured grids based on object-space decomposition [S1, unofficial]. It addresses the inefficiencies of rendering local clusters independently by recognizing that clusters on the same processor are highly interacting and should be rendered as a unified local subvolume [S1, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
