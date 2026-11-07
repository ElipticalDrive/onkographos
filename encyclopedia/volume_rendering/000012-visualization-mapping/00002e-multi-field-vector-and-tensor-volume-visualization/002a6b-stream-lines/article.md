# Stream Lines

Also called: CUDA Streamline Visualization, Euler method streamline, Optical Flow Glyphs and Particle Tracing, Particle Tracing, Particle path integration, Vector Field Trajectory Visualization

## Overview
Stream lines visualize 3D vector fields by computing the trajectories of mass-less particles seeded at specific locations, integrating the field to determine the path [S2, S3, unofficial]. Unlike [Line Glyphs for Vector Field Visualization](2569-line-glyphs-for-vector-field-visualization.md) or [Arrow Glyphs for Vector Field Visualization](2663-arrow-glyphs-for-vector-field-visualization.md), which display local direction and magnitude at discrete points, stream lines reveal the global flow structure by connecting these points into continuous curves [S1, S3, unofficial]. The method solves the problem of understanding flow topology, such as vortices or turbulence, which static glyphs may obscure [S3, unofficial].

## How it works
**Inputs**: A 3D vector field $\mathbf{V}(x,y,z)$, a set of seed points $\mathbf{S}$, an integration step size $\Delta t$, a maximum number of integration steps $N_{max}$, and a geometry sampling rate $G$ [S3, S4, unofficial].
**Outputs**: A set of vertices forming line strips (or tubes) and corresponding colors mapped from vector magnitude [S3, S4, unofficial].

**Data Structures**:
1.  **Vector Field**: Stored as a 3D texture array (e.g., `cudaArray` with `float4` channels) to enable hardware-accelerated trilinear interpolation [S4, unofficial]. The field values $\mathbf{v} = (v_x, v_y, v_z)$ are retrieved at arbitrary positions [S2, unofficial].
2.  **Color Transfer Function**: A 1D texture mapping vector magnitude to color (e.g., RGB) [S4, unofficial].
3.  **Output Buffers**:
    *   `outVertices`: Array of `float3` positions. Each stream line occupies $\lfloor N_{max} / G \rfloor + 1$ vertices [S3, unofficial].
    *   `outVertexColors`: Array of `float3` colors corresponding to the vertices [S3, unofficial].
    *   `outComputedSteps`: Array of `uint` storing the number of valid steps per seed [S3, unofficial].

**Algorithm**:
The integration is performed in double precision for accuracy, but results are stored in single precision to save memory [S3, unofficial].

1.  **Initialization**: For each seed $i$, initialize position $\mathbf{p}_0 = \mathbf{S}_i$ in double precision [S3, unofficial]. Write $\mathbf{p}_0$ to `outVertices` and compute its color from magnitude $|\mathbf{V}(\mathbf{p}_0)|$ [S4, unofficial].
2.  **Integration Loop**: For step $k$ from 1 to $N_{max}$:
    a.  **Boundary Check**: If $\mathbf{p}_{k-1}$ is outside the volume bounds (e.g., $x < 0$ or $x > \text{width}$), terminate the streamline [S3, S4, unofficial].
    b.  **Vector Lookup**: Sample $\mathbf{v} = \mathbf{V}(\mathbf{p}_{k-1})$ using trilinear interpolation [S2, S4, unofficial].
    c.  **Termination Check**: If $|\mathbf{v}| < v_{min}$ (threshold) or the streamline intersects itself (enters a previously visited cell), terminate [S2, unofficial].
    d.  **Integration**: Compute the next position $\mathbf{p}_k$ using an integrator (Euler or RK4) [S3, S4, unofficial].
    e.  **Geometry Sampling**: If $k \pmod G == 0$, write $\mathbf{p}_k$ to `outVertices` and its color to `outVertexColors` [S3, S4, unofficial].
3.  **Finalization**: Record the number of computed steps for seed $i$ [S4, unofficial].

**Integrators**:
*   **Euler Method**:
    $$ \mathbf{p}_k = \mathbf{p}_{k-1} + \Delta t \cdot \mathbf{v}(\mathbf{p}_{k-1}) $$
    Simple but less accurate [S3, S4, unofficial].
*   **Runge-Kutta 4 (RK4)**:
    Uses four texture lookups per step ($k_1$ to $k_4$) and a weighted average:
    $$ \mathbf{p}_k = \mathbf{p}_{k-1} + \frac{\Delta t}{6} (k_1 + 2k_2 + 2k_3 + k_4) $$
    where $k_1 = \mathbf{v}(\mathbf{p}_{k-1})$, $k_2 = \mathbf{v}(\mathbf{p}_{k-1} + \frac{\Delta t}{2}k_1)$, etc. [S4, unofficial].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_streamlines(seeds, vector_field_tex, color_tex, dt, max_steps, geometry_sampling, volume_size):
    """
    seeds: list of (x, y, z) float3 tuples
    vector_field_tex: 3D texture for vector lookup
    color_tex: 1D texture for magnitude-to-color mapping
    dt: integration step size
    max_steps: maximum integration steps per seed
    geometry_sampling: emit vertex every N integration steps
    volume_size: (width, height, depth)
    """
    out_vertices = []
    out_colors = []
    out_steps = []

    for seed in seeds:
        # Double precision position
        pos_x, pos_y, pos_z = float(seed.x), float(seed.y), float(seed.z)
        
        # Store initial vertex
        out_vertices.append((pos_x, pos_y, pos_z))
        mag = get_magnitude(vector_field_tex, pos_x, pos_y, pos_z)
        out_colors.append(map_magnitude_to_color(color_tex, mag))
        
        geometry_step = 0
        step_count = 0
        
        for step in range(1, max_steps + 1):
            # Boundary check
            if (pos_x < 0 or pos_y < 0 or pos_z < 0 or 
                pos_x >= volume_size.width or pos_y >= volume_size.height or pos_z >= volume_size.depth):
                break
            
            # Get vector at current position
            vx, vy, vz = sample_vector(vector_field_tex, pos_x, pos_y, pos_z)
            mag = (vx**2 + vy**2 + vz**2)**0.5
            
            # Termination conditions
            if mag < 1e-6: # Threshold for zero vector
                break
            
            # Integration (Euler example)
            # For RK4, replace this block with 4 sample/weight steps
            dx = dt * vx
            dy = dt * vy
            dz = dt * vz
            
            pos_x += dx
            pos_y += dy
            pos_z += dz
            
            step_count += 1
            
            # Geometry sampling
            geometry_step += 1
            if geometry_step >= geometry_sampling:
                geometry_step = 0
                out_vertices.append((pos_x, pos_y, pos_z))
                out_colors.append(map_magnitude_to_color(color_tex, mag))
        
        out_steps.append(step_count // geometry_sampling)

    return out_vertices, out_colors, out_steps

def sample_vector(tex, x, y, z):
    # Trilinear interpolation implementation
    # Returns (vx, vy, vz)
    pass

def map_magnitude_to_color(tex, mag):
    # Lookup in 1D color texture
    pass
```

**Parameters**:
*   `dt`: Integration step size. Smaller values increase accuracy but require more steps [S3, unofficial].
*   `geometry_sampling`: Integer specifying integration steps per output vertex. Allows high-precision integration (small `dt`) while reducing memory usage for the output geometry [S3, unofficial].
*   `volumeCoordSpaceMult`: Scaling factor to convert world coordinates to texture coordinates for 3D texture lookup [S4, unofficial].
*   `magnitudeCtfNormalizeMult`: Scalar to normalize vector magnitude for color lookup [S4, unofficial].

**Edge Cases and Pitfalls**:
*   **Distorted Volumes**: If the volume is distorted (e.g., horseshoe shape), the vector field given in computation space (C-space) must be transformed into physical space (P-space) using the Jacobian of the coordinate transformation to obtain correct traces [S1, unofficial].
*   **Precision**: Integration should be performed in double precision to prevent drift, but output can be single precision [S3, unofficial].
*   **Self-Intersection**: Streamlines may terminate if they re-enter a previously visited cell [S2, unofficial].

## Subtopics

### [Adaptive Stream Line Seeding](5604-adaptive-stream-line-seeding.md)
Adaptive Stream Line Seeding dynamically refines the density of stream lines in regions of high flow divergence (e.g., turbulence or vortices) while maintaining sparse sampling in laminar areas. Unlike static seeding, it iteratively subdivides neighboring stream lines when the distance between them exceeds a threshold, creating denser representations where flow complexity is highest. It differs from [Stream Surfaces](7696-stream-surfaces.md) by focusing on individual trajectories rather than continuous surfaces.

### [Stream Surfaces](7696-stream-surfaces.md)
Stream surfaces visualize steady-state vector fields by stitching adjacent streamlines to form a continuous geometric surface, helping to ascertain overall flow direction in areas where streamlines curl. Unlike discrete [Stream Lines](4796-stream-lines.md), they provide a unified entity renderable with standard hardware, though they may lose tangential flow direction information.

### Directional Textured Tracelines
A variant using 1D primitives (lines) enhanced with an animated 1D texture to indicate flow direction, which standard lines fail to show [S1, unofficial]. Each vertex is assigned a texture coordinate in [0, 1]; a global parameter $\Delta t$ shifts these coordinates to animate the texture along the line in the flow direction [S1, unofficial]. The texture (e.g., zebra-like pattern) is static and computed once on the CPU [S1, unofficial].

### Flow Geometry Bricks Rendering
A rendering approach for flow visualization using "geometry bricks" that hold range information of the visible region, constructed only when needed for the visible volume (e.g., a horseshoe region) [S1, unofficial]. Streamlines are stored in a single vertex buffer as a line strip, and arrows in an indexed buffer [S1, unofficial]. All geometry is rendered as opaque before the semi-transparent volume with back-to-front blending, using depth testing to accurately mix geometry and volume [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of seeds and $M$ is the number of integration steps per seed [S3, unofficial]. RK4 is roughly 4x more expensive per step than Euler due to multiple texture lookups [S4, unofficial].
*   **Memory Complexity**: $O(N \cdot M / G)$ for output vertices, where $G$ is the geometry sampling rate [S3, unofficial].
*   **Performance**: Real-time visualization is achievable on GPUs using CUDA, leveraging hardware-accelerated texture lookups for interpolation [S4, unofficial].
*   **Strengths**: Reveals global flow topology; interactive seeding allows exploration [S3, unofficial].
*   **Limitations**: Sensitive to seed placement; may clutter in complex flows; requires coordinate transformation for distorted volumes [S1, unofficial].

## Background
Stream lines were introduced as a method to visualize 3D vector fields by integrating trajectories of mass-less particles [S2, S3, unofficial]. Early implementations used Euler integration with piecewise trilinear interpolation [S2, unofficial]. The method has been extended to GPU architectures (CUDA) for real-time performance, supporting both Euler and RK4 integrators [S3, S4, unofficial]. It is widely used in fluid dynamics, optical flow visualization, and diffusion tensor imaging [S1, S3, unofficial].

## Sources
[S1] GPU-assisted Multi-field Video Volume Visualization, https://www.semanticscholar.org/paper/b3eb5a9069e8d00fa52149b0ff85554801f0f917, fetched 2026-11-04 (unofficial)
[S2] Visualization of Surfaces and 3D Vector Fields, https://digital.library.unt.edu/ark:/67531/metadc3235/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
[S3] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S4] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
