# Instancing with Procedural/Shader Generation

Also called: Instanced Voxel Rendering with Palette Textures, Palette-based Voxel Rendering

## Overview
Instancing with Procedural/Shader Generation renders a voxel model by instantiating a single unit cube mesh for each occupied voxel, positioning them via per-instance 3D offsets and coloring them via per-instance UV coordinates sampled from a 2D palette texture [S1, unofficial]. It solves the problem of efficiently rendering large voxel worlds by decoupling geometry generation from instance placement, allowing the GPU to handle the repetition of identical primitives [S1, unofficial]. It differs from [Binary Volume Rendering (Opacity Thresholding)](1015-binary-volume-rendering-opacity-thresholding.md) by explicitly rasterizing geometric proxies rather than ray-marching a scalar field, and from [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md) by pushing the coordinate transformation and visibility logic to the vertex/geometry stage rather than evaluating voxel occupancy entirely within a fragment shader [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of occupied voxel coordinates $(x, y, z)$ and a 2D palette texture containing base colors [S1, unofficial].
*   **Output:** A rasterized image of the voxel scene with Phong lighting [S1, unofficial].

**Data Structures:**
1.  **Instance Buffer (VBO):** A contiguous array of floats bound to `GL_ARRAY_BUFFER` [S1, unofficial]. Each instance contains exactly 5 floats in the following order:
    *   `offsetX` (float): X-coordinate of the voxel center.
    *   `offsetY` (float): Y-coordinate of the voxel center.
    *   `offsetZ` (float): Z-coordinate of the voxel center.
    *   `u` (float): S-coordinate for palette texture sampling.
    *   `v` (float): T-coordinate for palette texture sampling.
    *   Total size: $N_{voxels} \times 5 \times 4$ bytes [S1, unofficial].
2.  **Palette Texture:** A 2D texture storing the color map [S1, unofficial]. Wrap modes must be set to `GL_CLAMP_TO_EDGE` for both S and T axes to prevent sampling artifacts at boundaries [S1, unofficial].
3.  **Cube Geometry:** A static mesh representing a unit cube (typically 6 faces, 24 vertices for flat shading or 6 vertices for shared shading, though the specific vertex count depends on the base mesh implementation) [S1, G, unofficial].

**Numbered Steps:**
1.  **Data Preparation:** Iterate through the voxel grid [S1, unofficial]. For each occupied voxel at grid index $(i, j, k)$, calculate its world position [S1, unofficial]. If the voxel format uses Z-up and OpenGL uses Y-up, swap the Y and Z components during offset calculation: `localY = gridZ`, `localZ = gridY` [S1, unofficial].
2.  **Buffer Upload:** Pack the offsets and palette UVs into the contiguous float array [S1, unofficial]. Upload this array to the GPU using `glBufferData` with `GL_STATIC_DRAW` hint [S1, unofficial].
3.  **Shader Setup:** Bind the vertex shader and fragment shader program [S1, unofficial]. Retrieve uniform locations for the Model-View-Projection (MVP) matrix, light position, and palette texture [S1, unofficial].
4.  **Vertex Processing:**
    *   The vertex shader receives the base cube vertex position `aLocalPosition` and the instance offset `aInstanceOffset` (from the VBO) [S1, unofficial].
    *   Compute the combined world position: `localCombined = aLocalPosition + aInstanceOffset` [S1, unofficial].
    *   Transform to clip space: `gl_Position = uMVPMatrix * vec4(localCombined, 1.0)` [S1, unofficial].
    *   Pass the instance UVs (`vUV`) to the fragment shader [S1, unofficial].
5.  **Fragment Processing:**
    *   Sample the base color from the palette texture: `baseColor = texture(uPaletteTexture, vUV).rgb` [S1, unofficial].
    *   Compute lighting using a Phong model:
        *   Ambient: `ambient = baseColor * uAmbientIntensity`
        *   Diffuse: `diffuse = baseColor * max(dot(N, L), 0.0)`
        *   Specular: `specular = vec3(1.0) * spec * uSpecularIntensity`
        *   Final color: `baseColor + ambient + diffuse + specular` [S1, unofficial].
6.  **Rendering:** Issue a single draw call (e.g., `glDrawElementsInstanced` or `glDrawArraysInstanced`) to render all instances [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_instanced_voxel_renderer(voxel_data, palette_texture):
    # 1. Create Instance Buffer
    instance_data = []
    for voxel in voxel_data:
        x, y, z = voxel.position
        # Handle coordinate system swap if necessary (Z-up to Y-up)
        if voxel.format_is_z_up:
            y, z = z, y
        # Pack 5 floats: x, y, z, u, v
        instance_data.extend([x, y, z, voxel.u, voxel.v])
    
    # Upload to GPU as a contiguous float array bound to GL_ARRAY_BUFFER
    instance_vbo = allocate_buffer(instance_data, usage="static")
    
    # 2. Setup Shaders
    program = compile_program(vertex_shader_code, fragment_shader_code)
    u_mvp = get_uniform_location(program, "uMVPMatrix")
    u_palette = get_uniform_location(program, "uPaletteTexture")
    
    return {
        "instance_vbo": instance_vbo,
        "program": program,
        "u_mvp": u_mvp,
        "u_palette": u_palette,
        "count": len(voxel_data)
    }

def render_instanced_voxels(renderer, mvp_matrix):
    # Activate shader program
    use_program(renderer["program"])
    
    # Set Uniforms
    set_uniform_matrix4fv(renderer["u_mvp"], mvp_matrix)
    bind_texture_unit(0, renderer["palette_texture"])
    set_uniform1i(renderer["u_palette"], 0)
    
    # Bind Instance Data
    bind_buffer(renderer["instance_vbo"], target="array_buffer")
    
    # Enable vertex attributes for instance offsets and UVs
    # Assuming attribute locations are set up in shader/linking
    # glVertexAttribPointer for offset (stride 5*4, offset 0)
    # glVertexAttribPointer for uv (stride 5*4, offset 3*4)
    # glEnableVertexAttribArray for both
    # glVertexAttribDivisor for both to 1 (per-instance)
    
    # Draw
    draw_arrays_instanced(primitive="triangles", count=36, instances=renderer["count"])
```

**Parameters:**
*   `AMBIENT_INTENSITY`: 0.35 [S1, unofficial].
*   `SPECULAR_INTENSITY`: 0.5 [S1, unofficial].
*   `SHININESS`: 32 [S1, unofficial].
*   `LIGHT_POS_WORLD`: $(10, 15, 10)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate System Mismatch:** Voxel formats may use Z-up while OpenGL uses Y-up. This must be handled by swapping Y and Z components during offset calculation [S1, unofficial].
*   **Palette Artifacts:** Sampling at the edges of the palette texture can cause bleeding. Mitigate by setting texture wrap modes to `GL_CLAMP_TO_EDGE` [S1, unofficial].

## Subtopics

### [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md)
Surface-based Instancing with Noise Modulation generates a dense cloud of instanced points on a parametric surface, where point density and position are modulated by procedural noise functions. It solves the problem of creating smooth, organic volumetric appearances (such as smoke or fire) from discrete surface meshes by distributing millions of sample points across the geometry. It differs from sibling methods like [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md) by using scalar/vector noise to drive density and displacement rather than learned Gaussian parameters, and from [Procedural Particle Rendering](2926-procedural-particle-rendering.md) by explicitly mapping noise to surface parameter space ($s,t$) rather than world space.

### [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md)
This method renders sorted 2D Gaussians by submitting instanced quads to the standard hardware vertex/fragment pipeline, avoiding the memory overhead of tile-based rasterization. It solves the problem of efficiently projecting these Gaussians onto the screen by dynamically sizing quads based on opacity cutoffs and evaluating the Gaussian decay function in the fragment shader. It differs from [Volumetric Spot Noise Rendering](2360-volumetric-spot-noise-rendering.md) by relying on deterministic instanced geometry and standard blending rather than stochastic noise functions and hybrid OIT pipelines.

### [Indirect Draw Argument Setting](2418-indirect-draw-argument-setting.md)
Indirect Draw Argument Setting is a rendering optimization technique that uses a persistently mapped Vertex Buffer Object (VBO) as a memory pool for interleaved vertex data, divided into fixed-size buckets. Each occupied bucket is associated with an indirect draw call structure, allowing multiple sub-meshes to be rendered simultaneously via `glMultiDrawElementsIndirect`. This method reduces driver overhead from frequent state changes and draw calls while simplifying memory management for procedural voxel meshing. It sets itself apart from naive per-chunk VAO/VBO systems by merging sub-meshes and from merged VAO/VBO systems by avoiding full buffer resizes and re-uploads when updating sub-regions.

### [Procedural Particle Rendering](2926-procedural-particle-rendering.md)
Procedural Particle Rendering generates particle attributes on-the-fly by seeding a random number generator (RNG) with a particle ID, eliminating the need to store particle data in memory buffers. This approach allows particles to be generated directly inside a simple shader, reducing data management overhead.

### [Geometry Shader Advanced Instancing for Cubes](4576-geometry-shader-advanced-instancing-for-cubes.md)
A technique using geometry shaders to generate cube meshes from instanced points, optimizing for visibility by emitting at most 12 vertices (3 quads) per cube. It differs from creating full 24-vertex cubes by leveraging the fact that backface culling always removes one of two opposite faces, allowing the shader to emit only the visible subset.

### [Procedural Point Rendering with Billboard Transformation](6422-procedural-point-rendering-with-billboard-transformation.md)
A rendering technique using Unity procedural instancing to render surface points as billboards. It transforms a grid-based point cloud into world space, orients each point to face the camera, and scales them based on a step size.

### [Buffer-Based Instancing (SSBO/Instance Buffers)](6501-buffer-based-instancing-ssbo-instance-buffers.md)
Buffer-Based Instancing is a rendering technique that supplies per-instance data (such as position, rotation, and material IDs) to the vertex shader via large, flexible Shader Storage Buffer Objects (SSBOs) rather than standard vertex attribute buffers or uniform buffers. It solves the problem of efficiently rendering massive numbers of distinct objects (e.g., voxels) by allowing the CPU or a compute shader to write instance records directly into GPU memory, which the vertex shader then accesses dynamically using `gl_InstanceID`. It differs from standard hardware instancing by decoupling instance data from the vertex stream, enabling complex, non-uniform instance layouts and procedural generation of transforms in the vertex stage, avoiding CPU-side mesh generation.

### [Procedural and Rule-Based Spawning](13402-procedural-and-rule-based-spawning.md)
Procedural and Rule-Based Spawning is a family of techniques for populating voxel terrains with large numbers of objects (e.g., grass, rocks, trees) by evaluating spawn rules against terrain geometry rather than manual placement. It distinguishes itself from static instancing by integrating tightly with the voxel terrain's Level of Detail (LOD) system and Signed Distance Field (SDF) surface snapping, ensuring instances align with uneven terrain geometry automatically.

### [Hardware Instancing via Draw Calls](16221-hardware-instancing-via-draw-calls.md)
Hardware Instancing via Draw Calls is a rendering technique that renders a large number of identical meshes (voxels) efficiently by issuing a single draw call that instructs the GPU to replicate a base primitive multiple times. It solves the CPU bottleneck associated with issuing individual draw calls for large numbers of voxels by decoupling the CPU-side draw call issuance from the GPU-side execution logic. Unlike standard per-object rendering, which requires repeated matrix multiplications, uniform uploads, and state changes per frame, this method uploads transformation data once and lets the GPU handle the replication.

### Sphere Impostors for DEM Pebbles
Sphere Impostors for DEM Pebbles renders Discrete Element Method (DEM) pebble beds using instanced sphere impostors [S2, unofficial]. Each pebble is drawn as a quad with an exact orthographic ray-sphere hit test in the fragment shader, providing sphere shading and depth [S2, unofficial]. It draws DEM centres, not solver geometry, and renders to an offscreen target for interaction while positions stream [S2, unofficial]. It scales to 27,000 pebbles at interactive rates with 1.2-1.3 ms of CPU overhead per frame [S2, unofficial]. If no wgpu device is available, egui discs are used as the fallback rendering method [S2, unofficial].

### Splat quad vertex layout
Splat quad vertex layout sets up a 4-vertex quad with specific coordinates for shader-based scaling of projected 2D Gaussian eigenvectors [S3, unofficial]. Vertex positions are defined as $(-2,-2), (2,-2), (2,2), (-2,2)$ [S3, unofficial]. Indices are $0,1,2,0,2,3$ [S3, unofficial]. The vertex buffer uses `VET_FLOAT2` at `VES_POSITION` [S3, unofficial]. The index buffer uses `IT_16BIT` [S3, unofficial]. The instance buffer contains 1 float per instance (`VET_FLOAT1`, `VES_TEXTURE_COORDINATES[0]`, location 8) [S3, unofficial].

### Uniform Object Distribution via Modulo Instancing
Uniform Object Distribution via Modulo Instancing is a technique where a modulo function is applied to the scene distance function to repeat a single SDF component with a defined interval, generating a uniform grid of object copies [S4]. A pitfall is that cutting off the previous grid hit by a ray may result in 3D objects associated with the cell and potentially intersected by the ray being removed from the scene [S4]. The original single-pass approach causes visible artifacts from many vantage points despite using Existence operators to reduce visibility [S4].

## Operating characteristics
*   **Time Complexity:** Setup time is $O(V)$ to compute offsets and pack data, where $V$ is the number of voxels [S1, unofficial]. Draw time is $O(1)$ draw call with instancing, but fragment shader cost is proportional to visible voxels [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the instance buffer (5 floats per voxel) and $O(P)$ for the palette texture [S1, unofficial].
*   **Performance:** Implemented in the VoxelRenderer Android application for rendering .vly voxel models using OpenGL ES 3.0 [S1, unofficial]. It significantly reduces CPU overhead compared to per-voxel draw calls [S1, unofficial].
*   **Strengths:** Efficient rendering of large voxel counts; simple pipeline; easy color variation via palette textures [S1, unofficial].
*   **Limitations:** Limited to discrete voxel positions; requires palette texture management; potential coordinate system mismatches [S1, unofficial].

## Background
This method is commonly used in mobile and embedded graphics applications where hardware instancing support is available but advanced compute-based voxelization is not feasible [S1, unofficial]. It was notably implemented in the VoxelRenderer Android application for rendering .vly voxel models [S1, unofficial]. The technique leverages standard OpenGL ES 3.0 features, making it widely compatible with mobile GPUs [S1, unofficial].

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] dhoby-ghaut: GPU rendering for the 3D and 2D views (sphere impostors for pebbles; GPU or accelerated CSG ray tracing), https://github.com/theodoreOnzGit/outram-park-backend/issues/587, fetched 2026-10-26 (unofficial)
[S3] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S4] Real-time visualization of Dynamic Unlimited Objects Instancing, https://doi.org/10.24132/csrn.2018.2802.4, fetched 2026-10-26
