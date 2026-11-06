# WebGL 2 Hardware Instancing

Also called: Cube Particles, Hardware Instancing, Instanced Rendering, Instanced cube particles, gl.drawElementsInstanced

## Overview
WebGL 2 Hardware Instancing is a rendering technique that sends a single base primitive (typically a 2x2x2 cube) to the GPU and executes `gl.drawElementsInstanced` to render hundreds of thousands of copies simultaneously [S1, S2, unofficial]. It solves the CPU bottleneck associated with issuing individual draw calls for each voxel by packing translation, scaling, and color data into flat typed arrays [S1, unofficial]. Unlike surface-based methods, it renders the full volume geometry directly, relying on depth buffering for visibility [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A base cube geometry (vertices, normals, indices), instance data arrays (positions, scales, colors), and transformation matrices (Model-View-Projection).
*   **Output:** Rasterized pixels on the screen.

**Data Structures:**
The implementation uses two distinct layout strategies found in sources:
1.  **Attribute-based (S1):** Separate `Float32Array` buffers for instance positions (3 floats), scales (1 float), and colors (3 floats).
2.  **Matrix-based (S2):** A single buffer per instance containing 21 floats: 16 for a transformation matrix, 4 for color (RGBA), and 1 for health [S2, unofficial].

**Numbered Steps:**
1.  **Base Geometry Creation:** Create vertex buffers for the base cube. A standard cube uses 8 vertices and 36 indices (6 faces, 2 triangles each) [S1, unofficial].
    *   Positions: 8 vertices at $(\pm 1, \pm 1, \pm 1)$ [S1, unofficial].
    *   Normals: Flat shading normals per face [S1, unofficial].
    *   Indices: `0,1,2, 0,2,3...` [S1, unofficial].
2.  **Instance Data Preparation:**
    *   Calculate world positions for each voxel. For a grid of size $N$ with spacing $S$, the offset is $O = (N \times S) / 2$. Position $P_{x,z} = (i \times S) - O$ [S1, unofficial].
    *   Compute scale $S_y$ based on heightmap data or procedural noise (e.g., $S_y = 1 + \text{height} \times \text{factor}$) [S1, unofficial].
    *   Pack data into `Float32Array` buffers [S1, unofficial].
3.  **Buffer Upload:**
    *   Bind each instance attribute buffer using `gl.ARRAY_BUFFER`.
    *   Upload data using `gl.bufferData` with `gl.DYNAMIC_DRAW` if updated per frame, or `gl.STATIC_DRAW` if static [S1, unofficial].
4.  **Attribute Configuration:**
    *   Enable vertex attributes using `gl.enableVertexAttribArray`.
    *   Set pointer using `gl.vertexAttribPointer` to define stride and offset.
    *   **Crucial Step:** Set the divisor to 1 using `gl.vertexAttribDivisor(location, 1)` to indicate that the attribute changes once per instance, not once per vertex [S1, unofficial].
5.  **Shader Execution:**
    *   Vertex shader receives base vertex position `aVertexPosition` and instance attributes (`aInstancePosition`, `aInstanceScaleY`, `aInstanceColor`) [S1, unofficial].
    *   Apply transformation: `pos.y *= aInstanceScaleY; pos.xyz += aInstancePosition;` [S1, unofficial].
    *   Transform to clip space: `gl_Position = uProjectionMatrix * uViewMatrix * pos;` [S1, unofficial].
6.  **Draw Call:**
    *   Execute `gl.drawElementsInstanced(gl.TRIANGLES, indexCount, gl.UNSIGNED_SHORT, 0, instanceCount)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Setup Base Geometry
base_positions = [-1,-1,1, 1,-1,1, 1,1,1, -1,1,1, ...] # 8 vertices
base_normals   = [0,0,1, 0,0,1, ...]                   # Flat normals
base_indices   = [0,1,2, 0,2,3, ...]                   # 36 indices

pos_buf = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, pos_buf)
gl.bufferData(gl.ARRAY_BUFFER, Float32Array(base_positions), gl.STATIC_DRAW)

norm_buf = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, norm_buf)
gl.bufferData(gl.ARRAY_BUFFER, Float32Array(base_normals), gl.STATIC_DRAW)

idx_buf = gl.createBuffer()
gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, idx_buf)
gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, Uint16Array(base_indices), gl.STATIC_DRAW)

# 2. Setup Instance Data (Example: 500x500 grid)
grid_size = 500
spacing = 2.0
offset = (grid_size * spacing) / 2
num_instances = grid_size * grid_size

instance_positions = []
instance_scales = []
instance_colors = []

for x in range(grid_size):
    for z in range(grid_size):
        px = (x * spacing) - offset
        pz = (z * spacing) - offset
        # Example scale from heightmap
        scale_y = 1.0 + (heightmap[z * grid_size + x] / 255.0) * 100.0
        
        instance_positions.extend([px, 0.0, pz])
        instance_scales.append(scale_y)
        instance_colors.extend([1.0, 1.0, 1.0]) # Default white

pos_data = Float32Array(instance_positions)
scale_data = Float32Array(instance_scales)
color_data = Float32Array(instance_colors)

# 3. Create and Upload Instance Buffers
inst_pos_buf = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, inst_pos_buf)
gl.bufferData(gl.ARRAY_BUFFER, pos_data, gl.DYNAMIC_DRAW)

inst_scale_buf = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, inst_scale_buf)
gl.bufferData(gl.ARRAY_BUFFER, scale_data, gl.DYNAMIC_DRAW)

inst_color_buf = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, inst_color_buf)
gl.bufferData(gl.ARRAY_BUFFER, color_data, gl.DYNAMIC_DRAW)

# 4. Configure Attributes
# Base Position (Location 0)
gl.enableVertexAttribArray(0)
gl.bindBuffer(gl.ARRAY_BUFFER, pos_buf)
gl.vertexAttribPointer(0, 3, gl.FLOAT, False, 0, 0)

# Base Normal (Location 1)
gl.enableVertexAttribArray(1)
gl.bindBuffer(gl.ARRAY_BUFFER, norm_buf)
gl.vertexAttribPointer(1, 3, gl.FLOAT, False, 0, 0)

# Instance Position (Location 2)
gl.enableVertexAttribArray(2)
gl.bindBuffer(gl.ARRAY_BUFFER, inst_pos_buf)
gl.vertexAttribPointer(2, 3, gl.FLOAT, False, 0, 0)
gl.vertexAttribDivisor(2, 1) # Change once per instance

# Instance Scale (Location 3)
gl.enableVertexAttribArray(3)
gl.bindBuffer(gl.ARRAY_BUFFER, inst_scale_buf)
gl.vertexAttribPointer(3, 1, gl.FLOAT, False, 0, 0)
gl.vertexAttribDivisor(3, 1)

# Instance Color (Location 4)
gl.enableVertexAttribArray(4)
gl.bindBuffer(gl.ARRAY_BUFFER, inst_color_buf)
gl.vertexAttribPointer(4, 3, gl.FLOAT, False, 0, 0)
gl.vertexAttribDivisor(4, 1)

# 5. Draw
gl.drawElementsInstanced(gl.TRIANGLES, 36, gl.UNSIGNED_SHORT, 0, num_instances)
```

**Parameters:**
*   `gridSize`: Dimension of the square world grid (e.g., 500) [S1, unofficial].
*   `spacing`: Distance between voxel centers (e.g., 2.0) [S1, unofficial].
*   `instanceCount`: Total number of cubes ($gridSize^2$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Using `Float32Array` for large grids may cause z-fighting if the view matrix is not optimized for the scene bounds [S1, unofficial].
*   **Update Frequency:** If instance data changes every frame (e.g., particles), use `gl.DYNAMIC_DRAW` and `gl.bufferSubData` or re-upload with `gl.bufferData` [S1, S2, unofficial].
*   **Culling:** Frustum culling must be implemented on the CPU side before uploading instance data to avoid rendering off-screen voxels [S2, unofficial].

## Subtopics

### Surface-based Instancing with Noise Modulation
Renders only visible surface voxels, reducing draw calls compared to full volume instancing. Choose when memory is constrained and surface detail is sufficient. [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md)

### 3D Gaussian Splatting via Instancing
Uses Gaussian primitives instead of cubes for smoother, anti-aliased volumes. Choose for high-quality soft volumes. [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md)

### GPU Instancing with Indirect Draw Calls
Offloads instance count and draw parameter calculation to the GPU using compute shaders. Choose for dynamic, massive datasets where CPU upload is the bottleneck. [GPU Instancing with Indirect Draw Calls](2205-gpu-instancing-with-indirect-draw-calls.md)

### Procedural Particle Rendering
Renders points or billboards rather than cubes. Choose for performance when geometric fidelity of cubes is not required. [Procedural Particle Rendering](2926-procedural-particle-rendering.md)

### Geometry Shader Advanced Instancing for Cubes
Uses geometry shaders to generate cube faces from points. Choose when vertex buffer management is complex. [Geometry Shader Advanced Instancing for Cubes](4576-geometry-shader-advanced-instancing-for-cubes.md)

### Procedural Point Rendering with Billboard Transformation
Renders 2D quads that always face the camera. Choose for 2D overlays or simple particle effects. [Procedural Point Rendering with Billboard Transformation](6422-procedural-point-rendering-with-billboard-transformation.md)

### Buffer-Based Instancing (SSBO/Instance Buffers)
Uses Shader Storage Buffer Objects for larger instance data. Choose when instance data exceeds standard vertex attribute limits. [Buffer-Based Instancing (SSBO/Instance Buffers)](6501-buffer-based-instancing-ssbo-instance-buffers.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ for data upload and $O(1)$ for draw call, where $N$ is the number of instances. The GPU processes all instances in parallel.
*   **Memory Complexity:** $O(N)$ for instance buffers. Each instance requires 12-21 floats depending on the layout [S1, S2, unofficial].
*   **Measured Performance:** Renders 250,000 cubes (500x500 grid) in a single draw call [S1, unofficial].
*   **Strengths:** Bypasses CPU draw call overhead; efficient for static or semi-static volumes; simple implementation in WebGL 2.
*   **Limitations:** High memory usage for dense grids; no built-in occlusion culling (requires manual frustum culling); flat shading may appear jagged without normal interpolation [S1, S2, unofficial].
*   **When to Use:** Real-time voxel visualization where performance is critical and geometric simplicity is acceptable.
*   **When Not to Use:** When transparency requires strict back-to-front sorting (instancing does not sort instances) or when memory bandwidth is severely constrained.

## Background
Originating from WebGL 2 extensions and core features, hardware instancing allows efficient rendering of repeated geometry. The specific implementation of "Cube Particles" using `gl.drawElementsInstanced` was popularized in open-source voxel engines like CubeWorld-Canyon and CubeEngine [S1, S2, unofficial]. It builds on the principle of converting scalar field data into geometric primitives [Parent].

## Sources
[S1] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S2] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
