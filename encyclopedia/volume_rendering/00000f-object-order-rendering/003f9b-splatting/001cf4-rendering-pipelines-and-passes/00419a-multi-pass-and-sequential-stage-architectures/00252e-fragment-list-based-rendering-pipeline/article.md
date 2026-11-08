# Fragment List-Based Rendering Pipeline

Also called: 4-Stage Rendering Pipeline, Smoker Rendering Pipeline

## Overview
The Fragment List-Based Rendering Pipeline is a multi-pass object-order technique for real-time smoke volume rendering that decouples geometric traversal from shading to maintain accuracy and avoid raytracing self-intersection artifacts [S1, unofficial]. It differs from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by explicitly managing per-pixel fragment data in intermediate storage structures rather than accumulating color directly, and differs from the [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md) by focusing on volumetric primitive assembly and insertion rather than standard surface geometry [S1, unofficial].

## How it works
The pipeline processes smoke volumes through four distinct stages executed by three shader programs: Clear, Render (combining Construction and Insertion), and Display (Evaluation) [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Smoke volume data (density, flow tangents), geometric primitives (triangles representing volume slices or particles), and OpenGL buffer objects for intermediate storage [S1, unofficial].
*   **Output:** Final rendered image with per-pixel visualization of the smoke volume [S1, unofficial].

**Data Structures:**
*   **OpenGL Buffer Objects:** Used for resetting state and storing intermediate per-pixel information. Specific layouts depend on the visualization method (e.g., pixel counters or complex storage structures) [S1, unofficial].
*   **Fragment List:** An implicit or explicit data structure storing fragments generated during the Insertion stage, compatible with the Evaluation step [S1, unofficial].

**Numbered Steps:**

1.  **Clear:** Reset OpenGL buffer objects required for the subsequent frame. This stage is initiated by rendering a single screen-filling quad [S1, unofficial].
2.  **Construction:**
    *   **Vertex Shader:** Performs perspective transformation on vertices and assigns additional vertex data, such as density or flow tangents [S1, unofficial].
    *   **Geometry Shader:** Assembles primitives, processes triangle-related data (orientation, topology, volume information), and forwards results to the Insertion stage [S1, unofficial].
3.  **Insertion:**
    *   Primitives constructed in the previous stage are rasterized and forwarded to the fragment shader [S1, unofficial].
    *   The fragment shader processes incoming fragments. This ranges from a simple pixel counter to insertion into a complicated storage data structure [S1, unofficial].
    *   **Constraint:** The type of fragment processing must be compatible with the Evaluation step [S1, unofficial].
    *   **Pitfall:** Concurrent modification of shared memory during this step causes bottlenecks and inconsistent states if unmanaged, as shader threads cannot check whether a shared address is currently being modified [S1, unofficial].
4.  **Evaluation:** Render a single screen-filling quad to initiate fragment evaluation. This stage interprets and visualizes the per-pixel information prepared by the Insertion stage [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Stage 1: Clear
def clear_stage():
    # Reset OpenGL buffer objects for the subsequent frame
    # Initiated by rendering a single screen-filling quad
    render_screen_filling_quad(clear_shader)

# Stage 2 & 3: Construction and Insertion (Render Stage)
def render_stage(geometry):
    # Vertex Shader: Perspective transformation and data assignment
    transformed_vertices = []
    for vertex in geometry.vertices:
        pos = perspective_transform(vertex.position)
        density = vertex.density
        flow_tangent = vertex.flow_tangent
        transformed_vertices.append({
            'position': pos,
            'density': density,
            'flow_tangent': flow_tangent
        })
    
    # Geometry Shader: Primitive assembly and processing
    primitives = []
    for triangle in geometry.triangles:
        # Process triangle-related data
        orientation = compute_orientation(triangle)
        topology = compute_topology(triangle)
        volume_info = compute_volume_info(triangle)
        
        # Assemble and forward
        primitives.append({
            'vertices': [transformed_vertices[v] for v in triangle.vertices],
            'orientation': orientation,
            'topology': topology,
            'volume_info': volume_info
        })
    
    # Rasterization and Fragment Shader: Insertion
    for primitive in primitives:
        fragments = rasterize(primitive)
        for fragment in fragments:
            # Process fragment based on visualization method
            # Must be compatible with Evaluation step
            insert_fragment(fragment, storage_structure)

# Stage 4: Evaluation (Display Stage)
def evaluation_stage():
    # Render a single screen-filling quad
    # Interpret and visualize per-pixel information from Insertion stage
    render_screen_filling_quad(display_shader, storage_structure)
```

**Parameters:**
*   **Pipeline Stages:** Clear, Construction, Insertion, Evaluation [S1, unofficial].
*   **Shader Programs:** Clear, Render (Construction + Insertion), Display (Evaluation) [S1, unofficial].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Concurrent Access:** The Insertion step is the greatest bottleneck due to simultaneous access to shared indexing mechanisms. Unmanaged access may leave shared memory in an inconsistent state for both write and read operations [S1, unofficial].
*   **Compatibility:** Fragment processing in the Insertion stage must be strictly compatible with the Evaluation step [S1, unofficial].

## Subtopics

### Atomics
Techniques to manage concurrent modification of shared memory during the Insertion step, addressing the bottleneck of simultaneous access to shared indexing mechanisms [S1, unofficial].

### Runtime Switching
The pipeline allows switching between rendering methods during runtime without re-initializing the smoke volume, enabling comparison of different approaches using identical configurations [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Implemented in the Smoker application for real-time smoke volume rendering [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Maintains best possible accuracy and circumvents problems arising from raytracing and self or global intersections [S1, unofficial].
*   **Limitations:** The Insertion step is a significant bottleneck due to concurrent access to shared memory, requiring careful management to avoid inconsistent states [S1, unofficial].
*   **When to use:** Real-time smoke volume rendering where accuracy is prioritized over the potential speed of raytracing, and when avoiding self-intersection artifacts is critical [S1, unofficial].
*   **When not to use:** Scenes where raytracing self-intersection is not a concern, or where hardware lacks support for the required OpenGL buffer objects and shader functionality [S1, unofficial].

## Background
The Fragment List-Based Rendering Pipeline was developed as part of the Smoker application for real-time smoke volume rendering [S1, unofficial]. It was chosen over raytracing to circumvent problems arising from raytracing and self or global intersections [S1, unofficial]. The pipeline generalizes the rendering process into four stages executed by three shader programs, making heavy use of shader application, memory transactions, and GPU-related functionality [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
