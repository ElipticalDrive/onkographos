# Fragment List-Based Rendering Pipeline

Also called: 4-Stage Rendering Pipeline, Smoker Rendering Pipeline

## Overview
The Fragment List-Based Rendering Pipeline is a multi-pass architecture designed for real-time smoke volume rendering that processes volumetric primitives through four distinct stages: Clear, Construction, Insertion, and Evaluation [S1, unofficial]. It solves the problem of maintaining high accuracy in transparent volume visualization while circumventing the computational complexity and self-intersection artifacts inherent in raytracing methods [S1, unofficial]. Unlike standard ray-casting which accumulates contributions along rays, this method rasterizes primitives into per-pixel fragment lists, deferring the final compositing and visualization to a dedicated evaluation pass [S1, unofficial].

## How it works
The pipeline executes four sequential stages using three distinct shader programs: Clear, Render (combining Construction and Insertion), and Display (Evaluation) [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Smoke volume data (density, flow tangents), geometry primitives (triangles representing volume slices or particles), and screen resolution.
*   **Output:** Final rendered image buffer.

**Data Structures:**
*   **Fragment Lists:** Per-pixel storage structures (e.g., linked lists, atomic counters, or fixed-size arrays) that accumulate fragment data during the Insertion stage. The specific layout depends on the chosen visualization method but must be compatible with the Evaluation stage [S1, unofficial].
*   **OpenGL Buffer Objects:** Used to store intermediate states, reset per frame, and pass data between stages [S1, unofficial].

**Numbered Steps:**

1.  **Clear Stage:**
    *   Reset all OpenGL buffer objects and fragment list storage structures required for the subsequent frame [S1, unofficial].
    *   Initiated by rendering a single screen-filling quad using the **Clear** shader program [S1, unofficial].

2.  **Construction Stage:**
    *   **Vertex Shader:** Performs perspective transformation of vertices from volume space to screen space. Assigns additional vertex data such as density ($\rho$) and flow tangents [S1, unofficial].
    *   **Geometry Shader:** Assembles primitives (e.g., triangles) from the transformed vertices. Processes triangle-related data including orientation, topology, and volume information. Forwards the constructed primitives to the Insertion stage [S1, unofficial].
    *   This stage is part of the **Render** shader program [S1, unofficial].

3.  **Insertion Stage:**
    *   **Rasterization:** The primitives constructed in the previous stage are rasterized [S1, unofficial].
    *   **Fragment Shader:** Processes incoming fragments. The processing logic varies by visualization method, ranging from a simple pixel counter to inserting fragment data into a complicated storage data structure (fragment list) [S1, unofficial].
    *   **Constraint:** The type of fragment processing must be strictly compatible with the subsequent Evaluation step [S1, unofficial].
    *   This stage is part of the **Render** shader program [S1, unofficial].

4.  **Evaluation Stage:**
    *   Render a single screen-filling quad using the **Display** shader program [S1, unofficial].
    *   Interpret and visualize the per-pixel information (e.g., accumulated density, color, depth) that was prepared and stored by the Insertion stage [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode representing the logical flow of the 4-stage pipeline

def fragment_list_rendering_pipeline(volume_data, screen_resolution):
    # 1. Clear Stage
    # Reset buffers and fragment lists for the new frame
    clear_buffers()
    reset_fragment_lists(screen_resolution)
    
    # Launch Clear Stage: Render a single screen-filling quad
    render_screen_filling_quad(shader="Clear")

    # 2. Construction & 3. Insertion Stage (Combined in Render Program)
    # Process geometry primitives
    for primitive in volume_data.primitives:
        # Vertex Shader: Transform and assign data
        transformed_vertices = []
        for vertex in primitive.vertices:
            # Perspective transformation
            screen_pos = perspective_transform(vertex.position)
            # Assign additional data
            density = vertex.density
            flow_tangent = vertex.flow_tangent
            transformed_vertices.append({
                'screen_pos': screen_pos,
                'density': density,
                'flow_tangent': flow_tangent
            })
        
        # Geometry Shader: Assemble primitives and process topology
        constructed_primitives = assemble_primitives(transformed_vertices)
        
        # Rasterization and Fragment Shader: Insertion
        for fragment in rasterize(constructed_primitives):
            # Process fragment based on visualization method
            # Example: Insert into fragment list
            insert_fragment_into_list(fragment.screen_pos, fragment.density, fragment.flow_tangent)

    # 4. Evaluation Stage
    # Launch Evaluation Stage: Render a single screen-filling quad
    render_screen_filling_quad(shader="Display")
    
    # Display shader interprets per-pixel info from fragment lists
    # and writes final image to framebuffer

def render_screen_filling_quad(shader):
    # Generic function to launch a full-screen quad pass
    bind_shader(shader)
    draw_quad()
    unbind_shader()

def perspective_transform(position):
    # Standard MVP matrix multiplication
    return model_view_projection_matrix * position

def assemble_primitives(vertices):
    # Logic to form triangles or other primitives from vertices
    # Handles orientation and topology
    return [create_triangle(v0, v1, v2) for v0, v1, v2 in vertex_triplets(vertices)]

def rasterize(primitives):
    # Standard GPU rasterization process
    # Returns list of fragments with screen coordinates and interpolated attributes
    return gpu_rasterize(primitives)

def insert_fragment_into_list(screen_pos, density, flow_tangent):
    # Critical section: Concurrent modification of shared memory
    # Must handle atomic operations or synchronization to avoid inconsistent states
    # Implementation depends on specific fragment list structure (e.g., atomic counter + array)
    atomic_add_fragment_count(screen_pos)
    store_fragment_data(screen_pos, density, flow_tangent)

def clear_buffers():
    # Reset OpenGL buffer objects
    gl_clear_buffers()

def reset_fragment_lists(resolution):
    # Initialize per-pixel storage structures
    for x in range(resolution.width):
        for y in range(resolution.height):
            fragment_lists[x][y].clear()
```

**Parameters:**
*   **Pipeline Stages:** Clear, Construction, Insertion, Evaluation [S1, unofficial].
*   **Shader Programs:** Clear, Render (Construction + Insertion), Display (Evaluation) [S1, unofficial].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Concurrent Modification:** The Insertion step is the greatest bottleneck due to simultaneous access to shared indexing mechanisms. Unmanaged access to shared memory by shader threads can leave the memory in an inconsistent state because threads cannot check if a shared address is currently being modified [S1, unofficial].
*   **Compatibility:** The fragment processing method in the Insertion stage must be strictly compatible with the Evaluation stage; mismatched data structures will result in incorrect visualization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Performance is heavily influenced by the complexity of the fragment list insertion and the number of primitives per pixel.
*   **Memory Complexity:** Depends on the fragment list storage structure. Requires additional memory for per-pixel lists and intermediate buffers [S1, unofficial].
*   **Measured Performance:** Implemented in the Smoker application for real-time smoke volume rendering [S1, unofficial].
*   **Strengths:** Maintains high accuracy; circumvents problems arising from raytracing, self-intersections, and global intersections [S1, unofficial].
*   **Limitations:** The Insertion stage is a significant bottleneck due to concurrent memory access issues; requires careful management of shared memory states [S1, unofficial].
*   **When to use:** Real-time smoke volume rendering where accuracy is prioritized over the simplicity of ray-casting and where self-intersection artifacts must be avoided [S1, unofficial].
*   **When not to use:** Scenes with extremely high primitive density per pixel where fragment list management overhead becomes prohibitive, or when hardware does not support efficient atomic operations for fragment list insertion [S1, unofficial].

## Background
The Fragment List-Based Rendering Pipeline was developed as part of the "Smoker" application for real-time smoke volume rendering [S1, unofficial]. It was chosen over raytracing techniques to maintain the best possible accuracy and to circumvent problems arising from raytracing and self or global intersections [S1, unofficial]. The method is detailed in a master's thesis by Sowinski (2012) [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
