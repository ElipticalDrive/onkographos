# Fragment List-Based Rendering Pipeline

Also called: 4-Stage Rendering Pipeline, Smoker Rendering Pipeline

## Overview
The Fragment List-Based Rendering Pipeline is a multi-pass architecture designed for real-time smoke volume rendering that processes volumetric primitives through four distinct stages (Clear, Construction, Insertion, Evaluation) using fragment lists to store and evaluate per-pixel data [S1, unofficial]. It is chosen specifically to maintain high accuracy and circumvent the self-intersection and global intersection problems inherent in raytracing approaches [S1, unofficial]. Unlike the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) which may rely on direct ray-casting or simpler splatting, this method explicitly decouples the geometric construction and fragment storage from the final pixel evaluation, allowing for complex per-pixel data structures [S1, unofficial].

## How it works
The pipeline consists of four sequential stages executed by three shader programs: Clear, Render (combining Construction and Insertion), and Display (Evaluation) [S1, unofficial].

### Inputs and Outputs
*   **Input:** Smoke volume data (density, flow tangents), geometry primitives (triangles/cells representing volume elements), and OpenGL buffer objects.
*   **Output:** Final rendered image frame.

### Data Structures
*   **OpenGL Buffer Objects:** Used to store intermediate per-pixel information generated during the Insertion stage. The specific layout depends on the visualization method (e.g., simple counters or complex linked lists) [S1, unofficial].
*   **Fragment List:** A storage structure (potentially complex) where fragments are inserted during the Insertion stage. This structure must be compatible with the Evaluation step [S1, unofficial].

### Steps
1.  **Clear:** Reset OpenGL buffer objects for the subsequent frame. This is initiated by rendering a single screen-filling quad using the Clear shader program [S1, unofficial].
2.  **Construction:**
    *   **Vertex Shader:** Performs perspective transformation of vertices. Assigns additional vertex data such as density or flow tangents [S1, unofficial].
    *   **Geometry Shader:** Assembles primitives. Processes triangle-related data including orientation, topology, and volume information. Forwards results to the Insertion stage [S1, unofficial].
3.  **Insertion:**
    *   **Rasterization:** Primitives constructed in the previous stage are rasterized [S1, unofficial].
    *   **Fragment Shader:** Processes incoming fragments. Depending on the visualization method, this ranges from a simple pixel counter to the insertion of fragments into a complicated storage data structure (the fragment list) [S1, unofficial].
    *   **Constraint:** Fragment processing must be compatible with the Evaluation step [S1, unofficial].
4.  **Evaluation:**
    *   Render a single screen-filling quad using the Display shader program [S1, unofficial].
    *   Interpret and visualize the per-pixel information prepared by the Insertion stage [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Stage 1: Clear
def run_clear_stage():
    # Reset OpenGL buffer objects for the next frame
    reset_buffers()
    # Launch fragment shader threads by rendering a screen-filling quad
    render_screen_filling_quad(clear_shader)

# Stage 2 & 3: Construction and Insertion (Render Stage)
def run_render_stage(geometry):
    # Vertex Shader: Transform and assign data
    transformed_vertices = []
    for vertex in geometry.vertices:
        # Perspective transformation
        pos = perspective_transform(vertex.position)
        # Assign additional data
        density = vertex.density
        flow_tangents = vertex.flow_tangents
        transformed_vertices.append({
            'position': pos,
            'density': density,
            'flow_tangents': flow_tangents
        })
    
    # Geometry Shader: Assemble primitives and process triangle data
    primitives = []
    for triangle in geometry.triangles:
        # Process orientation, topology, volume info
        orientation = calculate_orientation(triangle)
        topology = calculate_topology(triangle)
        volume_info = calculate_volume(triangle)
        
        # Assemble primitive
        primitive = {
            'vertices': [transformed_vertices[v] for v in triangle.vertices],
            'orientation': orientation,
            'topology': topology,
            'volume_info': volume_info
        }
        primitives.append(primitive)
    
    # Rasterization and Fragment Shader (Insertion)
    for primitive in primitives:
        fragments = rasterize(primitive)
        for fragment in fragments:
            # Insertion logic: varies by method
            # Example: Insert into fragment list or increment counter
            insert_fragment(fragment, fragment_list)

# Stage 4: Evaluation
def run_evaluation_stage():
    # Render screen-filling quad to interpret per-pixel info
    render_screen_filling_quad(display_shader, fragment_list)
```

### Parameters
*   **Pipeline Stages:** Clear, Construction, Insertion, Evaluation [S1, unofficial].
*   **Shader Programs:** Clear, Render (Construction + Insertion), Display (Evaluation) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Concurrent Modification:** The Insertion step is the greatest bottleneck due to simultaneous access to shared indexing mechanisms. Concurrent modification of shared memory can cause bottlenecks and inconsistent states if unmanaged, as shader threads cannot check whether a shared address is currently being modified [S1, unofficial]. Critical sections must be treated with caution [S1, unofficial].

## Operating characteristics
*   **Performance:** Designed for real-time interactivity [S1, unofficial].
*   **Bottlenecks:** The Insertion stage is identified as the primary bottleneck due to concurrent access to shared memory structures [S1, unofficial].
*   **Accuracy:** Maintains high accuracy by avoiding raytracing artifacts [S1, unofficial].
*   **Flexibility:** Allows switching between rendering methods during runtime without re-initializing the smoke volume [S1, unofficial].

## Background
The method was implemented in the "Smoker" application for real-time smoke volume rendering [S1, unofficial]. It was chosen over raytracing to circumvent problems arising from self or global intersections [S1, unofficial]. The pipeline generalizes the rendering process into four stages executed by three shader programs [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
