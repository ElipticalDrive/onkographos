# Particle Traces on 2D Surfaces

Also called: Particle traces on surfaces, Surface particle traces, Trace lines on surfaces

## Overview
Particle traces on 2D surfaces is a visualization technique that represents the magnitude and direction of a 3D vector field on arbitrary 2D surfaces (not necessarily contour surfaces) by drawing long, evenly spaced particle trace lines [S1][S2, unofficial]. Unlike random seed placement which does not guarantee coverage near every point, this method uses a hierarchical integer lattice to ensure some seed point is near every point on the surface, avoiding density issues where the visualization becomes too dense or too sparse [S1][S2, unofficial]. It sets itself apart from its parent [Integral Surface](11268-integral-surface.md) by explicitly breaking traces into segments of contrasting color to resolve directional ambiguity in still images and by projecting the vector field onto the surface geometry rather than integrating within the volume [S1][S2, unofficial].

## How it works
**Inputs:**
*   A set of arbitrary 2D surfaces (represented as piecewise linear meshes).
*   A 3D vector field defined over the domain containing the surfaces.
*   A length threshold parameter $L_{thresh}$ (initially large, reduced gradually).
*   A constant time interval $\Delta t$ for segment length calculation.

**Outputs:**
*   A set of particle trace lines rendered on the surfaces, composed of colored segments.

**Data Structures:**
*   **Octree:** The surfaces are scan-converted into an octree. Each cell in the octree holds a plane equation (four numbers: $Ax + By + Cz + D = 0$). Only cells intersecting the surfaces are present. Surfaces are subdivided down to a constant size cell [S1].
*   **Seed Lattice:** An integer lattice used for seed point selection, organized spatially hierarchically [S1][S2, unofficial].
*   **Trace Storage:** A list of segments for each trace. Each segment stores its start and end points, color, and length.
*   **Cell History:** For each octree cell, store the length of the longest erased trace that passed through it, to prevent extensive recalculation during backtracking [S1][S2, unofficial].

**Steps:**

1.  **Surface Representation:** Scan convert the input surfaces into an octree. Subdivide surfaces until they fit into constant-size cells. Each cell stores a plane equation [S1].

2.  **Seed Selection:** Choose seed points on an integer lattice in a spatially hierarchical manner. This ensures the first particle traces start well away from each other [S1][S2, unofficial].

3.  **Advection Loop:** For each seed point:
    a.  Initialize particle position $P$ at the seed point.
    b.  Advect the particle along the surface in both forward and backward directions using variable step size Euler's method [S1][S2, unofficial].
    c.  At each step, project the 3D vector field $\vec{V}$ onto the surface to get the tangent direction $\vec{T}$ [S1][S2, unofficial].
    d.  Calculate segment length $L_{seg} = \Delta t \cdot |\vec{V}|$, where $|\vec{V}|$ is the magnitude of the vector field [S1].
    e.  Create a segment from the previous position to the new position. Color the segment using a sawtooth-shaped color map to resolve directional ambiguity in still images [S1][S2, unofficial].
    f.  Check termination conditions:
        i.  Particle reaches a stagnation zone (velocity approaches zero).
        ii. Particle reaches an edge of the surface.
        iii. Particle becomes too close to its own trace or another particle's trace [S1][S2, unofficial].
    g.  If the total trace length $L_{trace}$ is greater than the current length threshold $L_{thresh}$, keep the trace.
    h.  If $L_{trace} < L_{thresh}$, erase the trace. Preserve the length of the erased trace in the octree cells it passed through to prevent extensive recalculation while backtracking [S1][S2, unofficial].

4.  **Threshold Reduction:** After all seed points at the current resolution are tried, reduce the length threshold $L_{thresh}$ and repeat the process. Gradually reducing the threshold from maximum to minimum produces an aesthetically pleasing distribution but takes longer than using a single threshold value [S1][S2, unofficial].

5.  **Rendering:** Render the trace segments. Color table animation can be used to add motion, making the lines "flow" in the direction of the vector field with velocity proportional to the magnitude [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def particle_traces_on_2d_surfaces(surfaces, vector_field, delta_t, initial_threshold, min_threshold):
    # Step 1: Surface Representation
    octree = scan_convert_to_octree(surfaces)
    
    # Step 2: Seed Selection
    seeds = generate_hierarchical_integer_lattice(surfaces)
    
    traces = []
    cell_history = {} # Map cell_id -> max_erased_trace_length
    
    current_threshold = initial_threshold
    
    while current_threshold >= min_threshold:
        new_traces = []
        for seed in seeds:
            # Step 3: Advection Loop
            trace_segments = []
            current_pos = seed
            
            # Advect forward
            forward_segments = advect_along_surface(current_pos, vector_field, octree, delta_t, cell_history)
            trace_segments.extend(forward_segments)
            
            # Advect backward
            backward_segments = advect_along_surface(current_pos, vector_field, octree, delta_t, cell_history, direction=-1)
            trace_segments.extend(backward_segments)
            
            total_length = sum(seg.length for seg in trace_segments)
            
            # Step 3h: Check threshold
            if total_length >= current_threshold:
                new_traces.append(trace_segments)
            else:
                # Preserve length in cells
                for seg in trace_segments:
                    update_cell_history(cell_history, seg, total_length)
        
        traces.extend(new_traces)
        
        # Step 4: Threshold Reduction
        current_threshold *= 0.5 # Example reduction factor
        
    return traces

def advect_along_surface(start_pos, vector_field, octree, delta_t, cell_history, direction=1):
    segments = []
    current_pos = start_pos
    prev_pos = start_pos
    
    while True:
        # Get vector at current position
        v = vector_field.get_vector(current_pos)
        
        # Check for stagnation
        if magnitude(v) < epsilon:
            break
        
        # Project vector onto surface
        surface_normal = get_surface_normal(current_pos, octree)
        t = project_vector_onto_surface(v, surface_normal)
        
        # Normalize tangent
        t = normalize(t)
        
        # Calculate step size based on magnitude and delta_t
        step_size = delta_t * magnitude(v)
        
        # Update position
        next_pos = current_pos + direction * step_size * t
        
        # Create segment
        segment = Segment(prev_pos, next_pos, color=calculate_sawtooth_color(magnitude(v)))
        segments.append(segment)
        
        # Update history
        update_cell_history(cell_history, segment, segment.length)
        
        prev_pos = next_pos
        current_pos = next_pos
        
        # Check termination conditions
        if is_edge(current_pos, octree):
            break
        if is_too_close_to_other_trace(current_pos, segments):
            break
            
    return segments

def project_vector_onto_surface(v, normal):
    # Project v onto the plane defined by normal
    # T = V - (V . N) * N
    return v - dot(v, normal) * normal
```

**Parameters:**
*   $\Delta t$: Constant time interval for segment length calculation. Not given in the sources.
*   $L_{thresh}$: Length threshold. Initially set to a large value, then gradually reduced to minimum [S1].
*   $L_{min}$: Minimum length threshold. Not given in the sources.

**Pitfalls:**
*   Directional ambiguity in still images is resolved using a sawtooth-shaped color map across each segment [S1][S2, unofficial].
*   Random seed placement does not guarantee coverage near every point on the surfaces [S1][S2, unofficial].

## Subtopics

### [Vector Field Projection onto Surface](13034-vector-field-projection-onto-surface.md)
This family defines methods for projecting a 3D vector field onto a 2D surface to determine the direction of particle trace segments, as required by [Particle Traces on 2D Surfaces](12772-particle-traces-on-2d-surfaces.md). The primary distinction among siblings is viewpoint dependency: most techniques are viewpoint independent, while the vieweye projection requires recalculation when the viewpoint changes.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, the method involves hierarchical seed selection and advection, which may be computationally expensive for large surfaces and complex vector fields. Gradually reducing the threshold increases computation time compared to using a single threshold [S1][S2, unofficial].
*   **Memory Complexity:** The octree representation is more memory efficient than a full 3D grid, storing only cells intersecting the surfaces [S1].
*   **Performance:** Real-time rotation and particle motion can be achieved on small data sets, and interactive performance on large data sets, using hardware texture mapping on an Onyx2 workstation [S2, unofficial].
*   **Strengths:** Guarantees coverage near every point on the surface, avoids density issues, allows for long flowlines to be easily understood, and supports color table animation for motion [S1][S2, unofficial].
*   **Limitations:** Viewpoint dependent projection techniques require recalculation of particle traces when the viewpoint changes [S1]. Gradually reducing the threshold takes longer to calculate [S1][S2, unofficial].

## Background
The technique was introduced in the paper "Visualizing 3D velocity fields near contour surfaces" by Van Gelder and Wilhelms [S1][S2, unofficial]. The authors experimented with several different projections of the vector field to the surfaces, including normal, xy normal, cylinder, and eye projections [S1][S2, unofficial]. Color table animation, an old technique [Shoup79], was applied to flow visualization by Van Gelder and Wilhelms [VanGelder92] and was a primary goal of this technique [S1].

## Sources
[S1] Visualizing 3D velocity fields near contour surfaces, https://doi.org/10.5555/951087.951133, fetched 2026-10-26
[S2] Visualizing 3D velocity fields near contour surfaces. Revision 1, https://digital.library.unt.edu/ark:/67531/metadc1395286/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
