# Fragment List-Based Rendering Pipeline

Also called: 4-Stage Rendering Pipeline, Smoker Rendering Pipeline

## Overview
The Fragment List-Based Rendering Pipeline is a multi-pass architecture designed for accurate real-time smoke volume rendering, specifically chosen to circumvent the self-intersection and global intersection artifacts inherent in raytracing approaches [S1, unofficial]. It decomposes the rendering process into four distinct stages—Clear, Construction, Insertion, and Evaluation—using fragment lists to store and evaluate per-pixel data, thereby maintaining high visual fidelity while leveraging standard rasterization hardware [S1, unofficial].

## How it works
The pipeline executes a sequence of rendering commands where the output of one stage serves as the input for the next, requiring explicit management of intermediate GPU resources (Frame Buffer Objects or buffer objects) [S1, 10584-deferred-rendering-pipeline, unofficial]. The process is orchestrated by three shader programs: `Clear`, `Render` (combining Construction and Insertion), and `Display` (handling Evaluation) [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Smoke volume data (density, flow tangents), geometry primitives (triangles representing volume slices or particles), and screen resolution.
*   **Outputs:** Final composited image buffer.
*   **Intermediate Data:** Per-pixel fragment lists, counters, or accumulation buffers stored in GPU memory between passes [S1, unofficial].

### Data Structures
The core data structure is the **Fragment List**, stored in GPU memory (e.g., via `GL_TEXTURE_BUFFER` or `GL_ATOMIC_COUNTER_BUFFER`).
*   **Layout:** A dynamic array of fragment records. Each record typically contains:
    *   Depth ($z$)
    *   World-space position ($x, y, z$)
    *   Density ($\rho$)
    *   Normal/Flow tangent ($n_x, n_y, n_z$)
    *   [G] A pointer or index to the next fragment in the list to handle collisions, as the number of fragments per pixel is variable and unknown at compile time [G].
*   **Size:** Determined by the maximum number of fragments expected per pixel multiplied by the screen resolution. Not given in the sources.

### Numbered Steps

1.  **Clear Stage**
    *   **Execution:** Render a single screen-filling quad using the `Clear` shader program.
    *   **Action:** Reset all OpenGL buffer objects used for the subsequent frame. This typically involves zeroing out fragment lists, counters, and accumulation buffers to ensure a clean state for the new frame [S1, unofficial].

2.  **Construction Stage**
    *   **Execution:** Part of the `Render` shader program.
    *   **Vertex Shader:** Performs perspective transformation on input vertices. Assigns additional vertex data such as density and flow tangents [S1, unofficial].
    *   **Geometry Shader:** Assembles primitives (e.g., triangles). Processes triangle-related data including orientation, topology, and volume information. Forwards the processed primitives to the Insertion stage [S1, unofficial].

3.  **Insertion Stage**
    *   **Execution:** Part of the `Render` shader program, immediately following Construction.
    *   **Rasterization:** Primitives are rasterized into fragments.
    *   **Fragment Shader:** Processes incoming fragments. Depending on the specific visualization method, this ranges from a simple pixel counter to inserting fragment data into a complicated storage data structure (the fragment list) [S1, unofficial].
    *   **Constraint:** The type of fragment processing must be compatible with the Evaluation step. This stage is the primary bottleneck due to concurrent modification of shared memory [S1, unofficial].

4.  **Evaluation Stage**
    *   **Execution:** Render a single screen-filling quad using the `Display` shader program.
    *   **Action:** Interpret and visualize the per-pixel information prepared by the Insertion stage. This involves reading from the fragment lists and performing the final compositing or shading calculations to produce the output image [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the Fragment List-Based Rendering Pipeline
# Assumes standard OpenGL-like API with shader programs: Clear, Render, Display

def render_frame(smoke_volume, screen_resolution):
    # 1. Clear Stage
    bind_shader(Clear)
    bind_buffer(FragmentListBuffer, target=GL_WRITE_ONLY)
    bind_buffer(CounterBuffer, target=GL_WRITE_ONLY)
    # Reset buffers for the subsequent frame
    gl_clear_buffers() 
    draw_screen_quad()

    # 2. Construction & 3. Insertion Stage (Combined in Render program)
    bind_shader(Render)
    bind_texture(SmokeVolumeTexture, smoke_volume)
    bind_buffer(FragmentListBuffer, target=GL_SHADER_STORAGE_BUFFER) # Read-Write for insertion
    bind_buffer(CounterBuffer, target=GL_SHADER_STORAGE_BUFFER)     # Read-Write for atomic counters
    
    # Rasterize smoke volume geometry
    # Vertex Shader: Transform vertices, assign density/flow
    # Geometry Shader: Assemble primitives, process orientation/topology
    # Fragment Shader: Rasterize, insert fragments into list
    draw_smoke_geometry()

    # 4. Evaluation Stage
    bind_shader(Display)
    bind_texture(FragmentListTexture, FragmentListBuffer) # Read from insertion results
    bind_texture(CounterTexture, CounterBuffer)
    bind_framebuffer(OutputFramebuffer)
    
    # Interpret and visualize per-pixel information
    draw_screen_quad()

# Shader Logic Snippets (Conceptual)

# Clear Shader
void main() {
    // Reset per-pixel counters and list heads
    atomicCounterReset(CounterBuffer[gl_FragCoord.xy]);
    FragmentListBuffer[gl_FragCoord.xy].head = -1;
}

# Render Fragment Shader (Insertion)
void main() {
    // Calculate fragment properties (depth, density, normal)
    vec3 pos = ...;
    float density = ...;
    vec3 normal = ...;
    
    // Atomic operation to get unique index in fragment list
    int index = atomicAdd(CounterBuffer[gl_FragCoord.xy], 1);
    
    // Insert into shared memory structure
    // Pitfall: Concurrent modification requires careful handling of shared addresses
    FragmentListBuffer[gl_FragCoord.xy].entries[index] = Fragment(pos, density, normal);
}

# Display Fragment Shader (Evaluation)
void main() {
    vec4 color = vec4(0.0);
    int head = FragmentListBuffer[gl_FragCoord.xy].head;
    int count = CounterBuffer[gl_FragCoord.xy];
    
    // Iterate through fragment list for this pixel
    for (int i = 0; i < count; i++) {
        Fragment frag = FragmentListBuffer[gl_FragCoord.xy].entries[i];
        // Perform volume rendering integration (e.g., raymarching or splatting)
        color += integrate(frag, color);
    }
    
    gl_FragColor = color;
}
```

### Parameters and Typical Values
*   **Pipeline Stages:** 4 (Clear, Construction, Insertion, Evaluation) [S1, unofficial].
*   **Shader Programs:** 3 (Clear, Render, Display) [S1, unofficial].
*   **Geometry:** Triangles assembled in the Geometry Shader [S1, unofficial].
*   **Typical Values:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Concurrent Memory Access:** The Insertion step is the greatest bottleneck. Concurrent modification of shared memory (fragment lists) by multiple shader threads can cause bottlenecks and inconsistent states if unmanaged. Shader threads cannot check whether a shared address is currently being modified [S1, unofficial].
*   **Self/Global Intersections:** The method is specifically chosen to avoid problems arising from raytracing, such as self-intersection or global intersection artifacts [S1, unofficial].
*   **Compatibility:** Fragment processing in the Insertion stage must be strictly compatible with the Evaluation step. Mismatched data structures will result in incorrect visualization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the Insertion stage due to atomic operations and potential contention on shared fragment lists. The number of draw calls is constant (4 passes), but the computational load per pass scales with the number of fragments and screen resolution. Not given in the sources.
*   **Memory Complexity:** Requires storage for fragment lists proportional to the number of fragments per pixel times the screen resolution. High memory bandwidth usage due to frequent reads/writes to fragment lists during Insertion and Evaluation [S1, unofficial].
*   **Strengths:** Maintains high accuracy for smoke volumes; avoids raytracing artifacts; flexible visualization methods via the fragment list structure; real-time capable on modern GPUs [S1, unofficial].
*   **Limitations:** The Insertion stage is a significant bottleneck due to concurrent memory access; requires careful management of shared memory consistency; complex implementation due to multi-pass nature [S1, unofficial].
*   **When to Use:** Real-time smoke volume rendering where accuracy is prioritized over the simplicity of single-pass methods and where raytracing artifacts are unacceptable [S1, unofficial].
*   **When Not to Use:** Scenes with extremely high fragment density per pixel leading to memory exhaustion or severe contention; hardware with limited atomic operation performance [S1, unofficial].

## Background
The Fragment List-Based Rendering Pipeline was implemented in the **Smoker** application for real-time smoke volume rendering [S1, unofficial]. It was developed as part of a master's thesis by Dirk Sowinski in 2012, presented in "Real-Time Smoke Volume Rendering" [S1, unofficial]. The pipeline was chosen to provide a flexible framework that allows switching between different rendering strategies during runtime without re-initializing the smoke volume, facilitating comparison of different approaches under identical configurations [S1, unofficial]. It builds on the principles of multi-pass and sequential stage architectures, specifically leveraging the 4-stage decomposition (Clear, Construction, Insertion, Evaluation) [S1, 10539-multi-pass-and-sequential-stage-architectures, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
