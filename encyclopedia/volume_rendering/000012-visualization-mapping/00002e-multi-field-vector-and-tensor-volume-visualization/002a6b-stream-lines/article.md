# Stream Lines

Also called: CUDA Streamline Visualization, Euler method streamline, Optical Flow Glyphs and Particle Tracing, Particle Tracing, Particle path integration, Vector Field Trajectory Visualization

## Overview
Stream lines visualize 3D vector fields by computing the trajectories of massless particles seeded at specific locations, integrating the field to produce continuous curves [S2, S3, unofficial]. Unlike [Arrow Glyphs for Vector Field Visualization](2663-arrow-glyphs-for-vector-field-visualization.md) which show local direction at discrete points, stream lines reveal global flow patterns and connectivity [S3, unofficial]. They differ from [Stream Surfaces](7696-stream-surfaces.md) by representing flow as discrete trajectories rather than stitched geometric surfaces, preserving individual path information [S1, unofficial].

## How it works
**Inputs:**
*   Vector field data $\mathbf{V}(x,y,z)$, typically stored in a 3D texture or grid [S3, S4, unofficial].
*   Seed points $\mathbf{p}_0$, often arranged on a line or plane [S3, unofficial].
*   Integration parameters: step size $\Delta t$, maximum steps $N_{max}$, and geometry sampling rate $S_{geo}$ [S3, S4, unofficial].
*   Termination thresholds: minimum magnitude $v_{min}$, boundary bounds, and self-intersection checks [S2, unofficial].

**Outputs:**
*   Vertex buffer containing positions $\mathbf{p}_i$ of the stream line [S3, S4, unofficial].
*   Color buffer mapping vector magnitude to color [S3, S4, unofficial].
*   Step count buffer indicating the length of each stream line [S3, S4, unofficial].

**Data Structures:**
*   **Vector Field:** Stored as a 3D texture (`cudaArray` with `float4` channels) bound to a texture object with linear filtering and clamp addressing [S4, unofficial].
*   **Color Transfer Function:** Stored as a 1D texture (`cudaArray` with `float4` channels) mapping normalized magnitude to RGBA color [S4, unofficial].
*   **Output Buffers:**
    *   `outVertices`: Array of `float3` positions. Each stream line occupies $N_{max}/S_{geo} + 1$ vertices [S3, unofficial].
    *   `outVertexColors`: Array of `float3` colors corresponding to vertices [S3, unofficial].
    *   `outComputedSteps`: Array of `uint` storing the number of valid steps per seed [S3, unofficial].

**Algorithm Steps:**

1.  **Initialization:** For each seed point $i$, initialize position $\mathbf{p}$ in double precision to maintain accuracy during integration [S3, unofficial]. Write $\mathbf{p}$ as the first vertex in `outVertices` [S4, unofficial].
2.  **Integration Loop:** Iterate from step $k=1$ to $N_{max}$:
    a.  **Boundary Check:** If $\mathbf{p}$ is outside the volume bounds ($x<0, y<0, z<0$ or $x>W, y>H, z>D$), terminate the stream line [S3, S4, unofficial].
    b.  **Vector Lookup:** Sample the vector field $\mathbf{v} = \mathbf{V}(\mathbf{p})$ using trilinear interpolation [S1, S2, unofficial].
    c.  **Termination Check:** If $|\mathbf{v}| < v_{min}$, terminate [S2, unofficial].
    d.  **Integration:** Compute the next position $\mathbf{p}_{next}$ using an integrator (Euler or RK4) [S3, S4, unofficial].
    e.  **Update:** Set $\mathbf{p} = \mathbf{p}_{next}$ [S4, unofficial].
    f.  **Geometry Sampling:** Decrement a counter. If the counter reaches zero, reset it to $S_{geo}$ and write $\mathbf{p}$ to `outVertices` and its color to `outVertexColors` [S3, S4, unofficial].
3.  **Finalization:** Record the total number of geometry samples written for seed $i$ in `outComputedSteps` [S4, unofficial].

**Formulas:**

*   **Euler Integration:**
    $$ \mathbf{p}_{i+1} = \mathbf{p}_i + \Delta t \cdot \mathbf{V}(\mathbf{p}_i) $$
    where $\mathbf{V}$ is the interpolated vector field [S1, S3, unofficial].

*   **RK4 Integration:**
    $$ \mathbf{p}_{i+1} = \mathbf{p}_i + \frac{\Delta t}{6} (\mathbf{k}_1 + 2\mathbf{k}_2 + 2\mathbf{k}_3 + \mathbf{k}_4) $$
    where:
    $$ \mathbf{k}_1 = \mathbf{V}(\mathbf{p}_i) $$
    $$ \mathbf{k}_2 = \mathbf{V}\left(\mathbf{p}_i + \frac{\Delta t}{2} \mathbf{k}_1\right) $$
    $$ \mathbf{k}_3 = \mathbf{V}\left(\mathbf{p}_i + \frac{\Delta t}{2} \mathbf{k}_2\right) $$
    $$ \mathbf{k}_4 = \mathbf{V}(\mathbf{p}_i + \Delta t \mathbf{k}_3) $$
    [S4, unofficial].

*   **Trilinear Interpolation Weights:**
    For a point $(x,y,z)$ in a cell defined by corners $(x_i, y_i, z_i)$ to $(x_i+dx, y_i+dy, z_i+dz)$, the weights $b_1 \dots b_8$ are:
    $$ b_1 = \frac{(x_i+dx-x)(y_i+dy-y)(z_i+dz-z)}{dx \cdot dy \cdot dz} $$
    $$ b_2 = \frac{(x-x_i)(y_i+dy-y)(z_i+dz-z)}{dx \cdot dy \cdot dz} $$
    $$ b_3 = \frac{(x-x_i)(y-y_i)(z_i+dz-z)}{dx \cdot dy \cdot dz} $$
    $$ b_4 = \frac{(x_i+dx-x)(y-y_i)(z_i+dz-z)}{dx \cdot dy \cdot dz} $$
    $$ b_5 = \frac{(x_i+dx-x)(y_i+dy-y)(z-z_i)}{dx \cdot dy \cdot dz} $$
    $$ b_6 = \frac{(x-x_i)(y_i+dy-y)(z-z_i)}{dx \cdot dy \cdot dz} $$
    $$ b_7 = \frac{(x-x_i)(y-y_i)(z-z_i)}{dx \cdot dy \cdot dz} $$
    $$ b_8 = \frac{(x_i+dx-x)(y-y_i)(z-z_i)}{dx \cdot dy \cdot dz} $$
    [S2, unofficial].

*   **Coordinate Transformation (Distorted Volumes):**
    If the volume is distorted (e.g., horseshoe shape), vectors in computation space $\mathbf{V}_C$ must be transformed to physical space $\mathbf{V}_P$ using basis vectors $\mathbf{e}_i$ derived from the Jacobian:
    $$ \mathbf{V}_P = \sum_{i=x,y,t} \mathbf{e}_i v_i $$
    [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_streamline_cuda_kernel(seeds, seeds_count, dt, max_steps, volume_size, geometry_sampling, vector_field_tex, magnitude_ctf_tex):
    """
    CUDA-style kernel for computing stream lines.
    Each thread handles one seed.
    """
    # Calculate global thread ID
    thread_id = blockIdx.x * blockDim.x + threadIdx.x
    
    if thread_id >= seeds_count:
        return

    # Output buffer indices
    # Each streamline outputs max_steps / geometry_sampling + 1 vertices
    vertices_per_line = max_steps // geometry_sampling + 1
    out_vertex_idx = thread_id * vertices_per_line
    
    # Initialize position in double precision for accuracy
    pos_x = float(seeds[thread_id].x)
    pos_y = float(seeds[thread_id].y)
    pos_z = float(seeds[thread_id].z)
    
    # Write initial position
    out_vertices[out_vertex_idx].x = pos_x
    out_vertices[out_vertex_idx].y = pos_y
    out_vertices[out_vertex_idx].z = pos_z
    out_vertex_idx += 1
    
    geometry_step_counter = geometry_sampling
    step_count = 0
    
    for step in range(1, max_steps):
        # Boundary check
        if (pos_x < 0 or pos_y < 0 or pos_z < 0 or 
            pos_x > volume_size.width or pos_y > volume_size.height or pos_z > volume_size.depth):
            break
            
        # Sample vector field using trilinear interpolation
        # Note: In CUDA, this is done via tex3D<vectorFieldTex, float4>(pos)
        # Here we assume a helper function for clarity
        vec = sample_vector_field(pos_x, pos_y, pos_z, vector_field_tex)
        
        # Termination check: magnitude threshold
        mag = length(vec)
        if mag < 1e-6: # vmin threshold
            break
            
        # Integration step (Euler for simplicity, can be swapped for RK4)
        # p_{i+1} = p_i + dt * V(p_i)
        next_x = pos_x + dt * vec.x
        next_y = pos_y + dt * vec.y
        next_z = pos_z + dt * vec.z
        
        # Update position
        pos_x = next_x
        pos_y = next_y
        pos_z = next_z
        
        # Geometry sampling: only emit vertex every geometry_sampling steps
        geometry_step_counter -= 1
        if geometry_step_counter == 0:
            geometry_step_counter = geometry_sampling
            
            # Write position
            out_vertices[out_vertex_idx].x = pos_x
            out_vertices[out_vertex_idx].y = pos_y
            out_vertices[out_vertex_idx].z = pos_z
            
            # Map magnitude to color
            # Normalize magnitude for texture lookup
            norm_mag = mag * magnitude_ctf_normalize_mult
            color = sample_color_transfer_function(norm_mag, magnitude_ctf_tex)
            
            out_vertex_colors[out_vertex_idx].x = color.x
            out_vertex_colors[out_vertex_idx].y = color.y
            out_vertex_colors[out_vertex_idx].z = color.z
            
            out_vertex_idx += 1
            step_count += 1
            
    # Record the number of steps/vertices generated for this seed
    out_computed_steps[thread_id] = step_count
```

## Subtopics

### [Adaptive Stream Line Seeding](5604-adaptive-stream-line-seeding.md)
Adaptive Stream Line Seeding dynamically refines the density of stream lines in regions of high flow divergence (such as turbulence or vortices) while maintaining sparse sampling in laminar areas. Unlike static seeding strategies that distribute points uniformly, this method iteratively subdivides pairs of neighboring stream lines when the distance between them exceeds a defined threshold, effectively creating denser surface representations where flow complexity is highest. It sets itself apart from [Stream Surfaces](7696-stream-surfaces.md) by focusing on the adaptive generation of individual trajectories rather than constructing continuous surfaces, and from static [Stream Lines](4796-stream-lines.md) by reacting to the geometric spread of the field during rendering.

### Directional Textured Tracelines
A flow visualization technique using 1D primitives (lines) to convey orientation and extent, enhanced with an animated 1D texture to indicate flow direction, which standard lines fail to show [S1, unofficial]. Each vertex of a line is assigned a texture coordinate in the range $[0, 1]$. The local texture coordinate of each vertex is shifted with a global parameter $\Delta t$ to move the texture along the line in the direction of the underlying flow field [S1, unofficial]. The 1D texture (e.g., a zebra-like pattern) is computed on the CPU once and downloaded to the GPU, requiring no updates for animation [S1, unofficial].

### Flow Geometry Bricks Rendering
A rendering approach for flow visualization (arrows and streamlines) using geometry bricks that hold range information of the visible region, constructed for the whole visible horseshoe region only when needed [S1, unofficial]. Streamlines are stored in a single vertex buffer and rendered as a line strip, while arrows are stored in an indexed vertex buffer to avoid redundant vertices [S1, unofficial]. All geometry buffers are rendered as opaque geometry before displaying the semi-transparent volume with back-to-front blending, using depth testing to accurately mix geometry and volume information [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of seeds and $M$ is the number of integration steps per seed [S3, unofficial].
*   **Memory Complexity:** $O(N \cdot M / S_{geo})$ for the output vertex buffer, where $S_{geo}$ is the geometry sampling factor [S3, unofficial].
*   **Performance:** GPU acceleration (CUDA) allows real-time visualization of large datasets by evaluating stream lines in parallel across threads [S3, S4, unofficial].
*   **Strengths:** Reveals global flow structures, vortices, and connectivity; interactive seeding allows exploration [S3, unofficial].
*   **Limitations:** Requires careful selection of seed points to avoid clutter or missing features; integration errors can accumulate; distorted volumes require coordinate transformation [S1, S2, unofficial].
*   **When to use:** When understanding the global trajectory and connectivity of flow is more important than local vector direction at every point [S3, unofficial].
*   **When not to use:** When the field is unsteady (stream lines are for steady fields; path lines or streak lines are needed for unsteady fields, though not covered here) or when the field is too noisy for stable integration [S2, unofficial].

## Background
Stream lines were introduced as a fundamental technique for visualizing vector fields, computing trajectories by integrating ordinary differential equations (ODEs) [S2, unofficial]. The method has been widely implemented in GPU-based systems for real-time visualization, leveraging CUDA for parallel integration and texture-based interpolation [S3, S4, unofficial]. It is commonly used in fluid dynamics, aerodynamics, and video volume visualization to analyze optical flow and complex vector fields [S1, S3, unofficial].

## Sources
[S1] GPU-assisted Multi-field Video Volume Visualization, https://www.semanticscholar.org/paper/b3eb5a9069e8d00fa52149b0ff85554801f0f917, fetched 2026-11-04 (unofficial)
[S2] Visualization of Surfaces and 3D Vector Fields, https://digital.library.unt.edu/ark:/67531/metadc3235/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
[S3] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S4] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
