# Fragment List-Based Rendering Pipeline

Also called: 4-Stage Rendering Pipeline, Smoker Rendering Pipeline

## Overview
A rendering pipeline that processes smoke volumes through four distinct stages (Clear, Construction, Insertion, Evaluation) using fragment lists to store and evaluate per-pixel data, chosen to maintain accuracy and avoid raytracing/self-intersection problems [S1, unofficial]. It differs from raytracing by circumventing self-intersection issues and from other object-order methods by explicitly decoupling geometry assembly from per-pixel evaluation via intermediate fragment storage [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Smoke volume data (density, flow tangents), geometry primitives (vertices, topology), OpenGL buffer objects.
*   **Output:** Final rendered image (visualization of per-pixel information).

**Data Structures:**
*   **OpenGL Buffer Objects:** Used for intermediate storage between stages. Specific layouts depend on the visualization method (e.g., pixel counters or complex storage structures). Not given in the sources.
*   **Fragment List:** An in-memory structure storing per-pixel fragment data generated during the Insertion stage. The exact layout is method-dependent (e.g., simple counter vs. complex structure). Not given in the sources.

**Numbered Steps:**
1.  **Clear:** Reset OpenGL buffer objects for the subsequent frame. This is initiated by rendering a single screen-filling quad using the Clear shader program [S1, unofficial].
2.  **Construction:**
    *   Vertex shader performs perspective transformation and assigns additional vertex data (e.g., density, flow tangents) [S1, unofficial].
    *   Geometry shader assembles primitives, processes triangle-related data (orientation, topology, volume information), and forwards results to the Insertion stage [S1, unofficial].
3.  **Insertion:**
    *   Primitives are rasterized and forwarded to the fragment shader [S1, unofficial].
    *   Process incoming fragments according to the visualization method. This ranges from a simple pixel counter to insertion into a complicated storage data structure [S1, unofficial].
    *   Fragment processing must be compatible with the Evaluation step [S1, unofficial].
    *   *Note:* Construction and Insertion stages belong to the same shader program (the Render stage) [S1, unofficial].
4.  **Evaluation:** Render a single screen-filling quad to initiate fragment evaluation. Interpret and visualize per-pixel information that was prepared by the Insertion stage [S1, unofficial].

**Parameters:**
*   **Pipeline Stages:** Clear, Construction, Insertion, Evaluation.
*   **Shader Programs:** Clear, Render (combines Construction and Insertion), Display (handles Evaluation) [S1, unofficial].

**Pitfalls:**
*   **Concurrent Modification:** Concurrent modification of shared memory during the Insertion step causes bottlenecks and inconsistent states if unmanaged. Shader threads cannot check whether a shared address is currently being modified [S1, unofficial].
*   **Compatibility:** The type of fragment processing in the Insertion stage must be compatible with the Evaluation step [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Stage 1: Clear
def clear_stage():
    """Reset OpenGL buffer objects for the subsequent frame."""
    # Launch Clear shader program by rendering a single screen-filling quad
    render_screen_filling_quad(shader_program="Clear")

# Stage 2 & 3: Construction and Insertion (Render Stage)
def render_stage(geometry):
    """
    Construct primitives and insert fragments into storage.
    Combines Construction and Insertion stages into a single shader program.
    """
    # Vertex Shader: Perspective transformation and data assignment
    transformed_vertices = []
    for vertex in geometry.vertices:
        # Perform perspective transformation
        pos = perspective_transform(vertex.position)
        # Assign additional vertex data (density, flow tangents)
        density = vertex.density
        flow_tangent = vertex.flow_tangent
        transformed_vertices.append({
            'position': pos,
            'density': density,
            'flow_tangent': flow_tangent
        })

    # Geometry Shader: Assemble primitives and process triangle data
    primitives = []
    for triangle in geometry.triangles:
        # Process triangle-related data (orientation, topology, volume info)
        orientation = compute_orientation(triangle)
        topology = compute_topology(triangle)
        volume_info = compute_volume_info(triangle)
        
        # Assemble primitive and forward to Insertion
        primitive = {
            'vertices': [transformed_vertices[v] for v in triangle.vertices],
            'orientation': orientation,
            'topology': topology,
            'volume_info': volume_info
        }
        primitives.append(primitive)

    # Rasterize primitives and forward to Fragment Shader (Insertion)
    for primitive in primitives:
        fragments = rasterize(primitive)
        for fragment in fragments:
            # Fragment Shader: Process incoming fragments
            # Depending on visualization method: simple pixel counter or complex storage
            insert_fragment(fragment)

# Stage 4: Evaluation
def evaluation_stage():
    """Interpret and visualize per-pixel information prepared by Insertion."""
    # Launch Display shader program by rendering a single screen-filling quad
    render_screen_filling_quad(shader_program="Display")

def main():
    # Execute pipeline stages in order
    clear_stage()
    render_stage(geometry)
    evaluation_stage()
```

## Subtopics

### Volume Rendering Pipeline
A general pipeline for volume rendering, often involving raycasting or splatting. The Fragment List-Based Pipeline is a specific implementation within this family, focusing on fragment storage [1318-volume-rendering-pipeline.md].

### Deferred Rendering Pipeline
A pipeline that separates geometry processing from shading, storing intermediate data in buffers. The Fragment List-Based Pipeline shares the concept of intermediate storage but is specifically tailored for smoke volume rendering using a 4-stage approach [10584-deferred-rendering-pipeline.md].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Maintains high accuracy for smoke volume rendering; circumvents problems arising from raytracing and self/global intersections [S1, unofficial].
*   **Limitations:** The Insertion step is a significant bottleneck due to simultaneous access to shared indexing mechanisms; concurrent modification of shared memory can lead to inconsistent states if not managed [S1, unofficial].
*   **When to Use:** Real-time smoke volume rendering where accuracy is prioritized and raytracing is unsuitable due to intersection issues [S1, unofficial].
*   **When Not to Use:** Scenes with complex global intersections where raytracing is preferred; scenarios where shared memory bottlenecks cannot be mitigated [S1, unofficial].

## Background
The Fragment List-Based Rendering Pipeline was implemented in the Smoker application for real-time smoke volume rendering [S1, unofficial]. It was chosen to maintain the best possible accuracy and to circumvent the problems that arise from raytracing and self or global intersections [S1, unofficial]. The pipeline generalizes the rendering process into four stages executed by three shader programs: Clear, Render, and Display [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
