# Eye Projection
Also called: Vieweye projection
## Overview
Eye projection is a viewpoint-dependent technique for visualizing 3D vector fields by projecting 3D vectors onto a surface in a direction parallel to the viewing direction [S1][S2, unofficial]. It produces the most intuitive representation of a constant direction vector field for a single image, making the uniform direction of the field readily apparent [S1][S2, unofficial]. Unlike viewpoint-independent techniques, it introduces visual discontinuities (jumpiness or flashing) when the viewpoint changes in animation [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D vector field $\mathbf{V}$, a surface $S$ (typically a contour surface), and a viewing direction vector $\mathbf{V}_{view}$ [S1][S2, unofficial].
- Output: A 2D projection of the vector field on the surface, visualized as particle traces [S1][S2, unofficial].

**Data Structures:**
Not given in the sources [S1][S2, unofficial].

**Steps:**
1. Determine the viewing direction vector $\mathbf{V}_{view}$ from the camera position to the scene [S2, unofficial].
2. For each point on the surface, project the 3D velocity vector $\mathbf{V}$ onto the surface along the direction of $\mathbf{V}_{view}$ [S2, unofficial].
3. Generate particle traces using the projected vectors, reusing the exact same set of trial seed points as the previous frame [S1][S2, unofficial].
4. Start each particle trace with the same "phase" as a nearby trace in the previous frame to minimize visual discontinuity [S1][S2, unofficial].

**Formulas:**
Not given in the sources. The text describes the geometric concept (projection parallel to viewing direction) but does not provide a specific mathematical formula for the projection operation [S1][S2, unofficial].

**Parameters:**
- Trial seed points: The same set of trial seed points as the previous frame should be used to minimize visual effects of changes [S1][S2, unofficial].
- Phase: Each particle trace should start with the same "phase" as a nearby trace in the previous frame [S1][S2, unofficial].

**Edge cases and pitfalls:**
- Viewpoint changes: When changing the viewpoint for a film loop, the projection of the 3D vectors to the surfaces changes, resulting in a different set of particle traces [S1][S2, unofficial]. This causes jumpiness or flashing in animation [S1][S2, unofficial].
- Surface suitability: This projection is only suitable for simple surfaces which are centered on the rotational axis [S1][S2, unofficial].
- Information loss: The projection of a 3D vector field onto a 2D curved surface loses and distorts much of the information from the 3D field, limiting the usefulness of these techniques for 3D visualization [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from collections import namedtuple
Trace = namedtuple('Trace', ['start_point', 'path', 'phase'])

def eye_projection(vector_field, surface, v_view, previous_traces):
    # Step 1: Determine viewing direction (provided as v_view)
    # Step 2: Project vectors onto surface along v_view
    projected_vectors = []
    for point in surface.points:
        v_3d = vector_field[point]
        # Project along viewing direction: find intersection of ray from point along v_view with surface
        t = intersect_ray_surface(point, v_view, surface)
        if t is not None:
            intersection_point = point + t * v_view
            projected_vectors.append((intersection_point, v_3d))
        else:
            projected_vectors.append((point, v_3d))

    # Step 3 & 4: Generate traces with continuity (same seeds and phase)
    traces = []
    for prev_trace in previous_traces:
        new_trace = advance_trace(prev_trace, projected_vectors, prev_trace.phase)
        traces.append(new_trace)
    return traces

def intersect_ray_surface(origin, direction, surface):
    """Finds the closest positive intersection distance t along direction with surface."""
    closest_t = None
    for face in surface.faces:
        t = ray_triangle_intersection(origin, direction, face)
        if t is not None and (closest_t is None or t < closest_t):
            closest_t = t
    return closest_t

def ray_triangle_intersection(origin, direction, triangle):
    """Möller–Trumbore ray-triangle intersection [G]."""
    v1, v2, v3 = triangle.vertices
    e1 = v2 - v1
    e2 = v3 - v1
    h = cross(direction, e2)
    a = dot(e1, h)
    if abs(a) < 1e-8: return None
    f = 1.0 / a
    s = origin - v1
    u = f * dot(s, h)
    if u < 0.0 or u > 1.0: return None
    q = cross(s, e1)
    v = f * dot(direction, q)
    if v < 0.0 or u + v > 1.0: return None
    t = f * dot(e2, q)
    return t if t > 1e-7 else None

def advance_trace(prev_trace, projected_vectors, phase_offset):
    """Advances a particle trace using projected vectors, maintaining phase continuity [G]."""
    new_path = []
    current_pos = prev_trace.start_point
    step_size = 0.1  # Typical step size for particle integration [G]
    for _ in range(prev_trace.steps):
        nearest = min(projected_vectors, key=lambda p: distance(current_pos, p[0]))
        v = nearest[1]
        current_pos = current_pos + v * step_size
        new_path.append(current_pos)
    return Trace(start_point=prev_trace.start_point, path=new_path, phase=phase_offset)
```

## Subtopics

### [Stereo Projection](5789-stereo-projection.md)
Projects vectors for stereo viewing, providing depth cues [G]. Choose when depth perception is critical [G].

### [MCOP Projection Geometry](6527-mcop-projection-geometry.md)
Uses a specific projection geometry for multi-center of projection [G]. Choose for complex multi-view setups [G].

## Operating characteristics
- Time and memory complexity: Not given in the sources [S1][S2, unofficial].
- Measured performance: Not given in the sources [S1][S2, unofficial].
- Scaling: Not given in the sources [S1][S2, unofficial].
- Strengths: Produces the most intuitive representation of a constant direction vector field for a single image [S1][S2, unofficial]. The uniform direction of the vector field is readily apparent [S1][S2, unofficial].
- Limitations: Viewpoint-dependent, causing jumpiness or flashing in animation when the viewpoint changes [S1][S2, unofficial]. Only suitable for simple surfaces centered on the rotational axis [S1][S2, unofficial]. Loses and distorts much of the information from the 3D field when projected onto a 2D curved surface [S1][S2, unofficial].
- When to use: For single-image visualization of 3D vector fields where intuitive representation of direction is prioritized over animation smoothness or 3D information preservation [S1][S2, unofficial].
- When not to use: For animated sequences where viewpoint changes are frequent, or for complex surfaces not centered on the rotational axis [S1][S2, unofficial].

## Background
Origin: The technique is described in the paper "Visualizing 3D velocity fields near contour surfaces" by Weusing [S1][S2, unofficial].
History and development: The technique is presented as an alternative to viewpoint-independent techniques, highlighting its trade-offs in terms of visual continuity and information preservation [S1][S2, unofficial].
Where it is used: Visualization of 3D velocity fields near contour surfaces [S1][S2, unofficial].

## Sources
[S1] Visualizing 3D velocity fields near contour surfaces, https://doi.org/10.5555/951087.951133, fetched 2026-10-26
[S2] Visualizing 3D velocity fields near contour surfaces. Revision 1, https://digital.library.unt.edu/ark:/67531/metadc1395286/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
