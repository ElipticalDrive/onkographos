# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
This method defines a trait-based configuration system that decouples voxel geometry definition from mesh generation logic. It solves the problem of rigid, switch-case-heavy meshing by allowing each block type to declare its own shape, transparency culling behavior, and lighting properties [S1, unofficial]. Unlike the parent method which assumes uniform cube geometry, this approach supports complex shapes (ramps, fluids) and fine-grained transparency control (merged vs. unmerged) [S1, S2, S3, unofficial].

## How it works
**Inputs:** A voxel grid (or chunk) containing block IDs; a registry of block types implementing the configuration interface.
**Outputs:** A list of vertices and indices for the mesh, annotated with block-specific metadata (AO, light, atlas offset).

**Data Structures:**
1.  **Block Trait Interface:** Each block type implements a trait (or interface) with three key methods:
    *   `shape() -> Shape`: Returns the geometric primitive(s) for the block.
    *   `cull_mode() -> CullMode`: Returns the transparency handling strategy.
    *   `light() -> u8` (or `f32`): Returns the light level or intensity per face.
2.  **Shape Enum:** Defines the geometry. Common variants include:
    *   `WholeBlock`: A full 1x1x1 cube.
    *   `Cross(CrossInfo)`: A cross-shaped block (e.g., chains) defined by an `AlignedFace` and `stretch` value [S1, unofficial].
    *   `Ramp`: A sloped block composed of a slope quad and side triangles [S2, unofficial].
    *   `Fluid`: Implicit surface geometry stitched from neighbors [S1, unofficial].
3.  **CullMode Enum:** Determines face visibility logic.
    *   `Empty`: No geometry generated.
    *   `Opaque`: Standard culling; shared faces between two opaque blocks are culled.
    *   `TransparentMerged(T)`: Shared faces between two transparent blocks are culled if their `TransparentGroup` values match. This reduces overdraw for identical transparent materials [S1, unofficial].
    *   `TransparentUnmerged`: Faces are never culled against other transparent blocks, even if identical. Used for blocks that should appear distinct or have unique lighting (e.g., leaves, chains) [S1, unofficial].
4.  **Vertex Structure:**
    ```rust
    struct Vertex {
        pos: Vec4<f32>,      // Position in world space
        uv: Vec2<f32>,       // Texture coordinates
        normal: Vec3<f32>,   // Surface normal
        ao: f32,             // Ambient occlusion (0.0 = occluded, 1.0 = lit)
        smooth_light: f32,   // Smooth light level (0.0..1.0)
        block: MyBlock,      // Block ID for render pass filtering
        atlas_offset: f32,   // Pre-computed atlas tile offset
        two_sided: bool,     // Whether the quad is two-sided
    }
    ```
    Interpolated vertex-to-fragment data (`VsOut`) includes `uv`, `normal`, `atlas_u_offset`, `ao`, and `smooth_light` [S1, unofficial].

**Steps:**
1.  **Iterate Voxel Grid:** For each non-empty voxel at position $P$ with block type $B$:
2.  **Retrieve Configuration:** Call $B$.`shape()`, $B$.`cull_mode()`, and $B$.`light()`.
3.  **Generate Geometry:**
    *   If `shape()` returns `WholeBlock`, generate 6 faces.
    *   If `shape()` returns `Cross`, generate the specific cross geometry based on `CrossInfo` [S1, unofficial].
    *   If `shape()` returns `Ramp`, generate slope quad and side triangles [S2, unofficial].
    *   If `shape()` returns `Fluid`, calculate surface height and stitch with neighbors if `FluidInfo::id` matches [S1, unofficial].
4.  **Apply Culling:** For each generated face with normal $\mathbf{n}$:
    *   Calculate neighbor position $P_{neighbor} = P + \mathbf{n}$.
    *   Get neighbor block type $B_{neighbor}$.
    *   If $B_{neighbor}$ is empty, the face is visible.
    *   If $B_{neighbor}$ is not empty, check `cull_mode()`:
        *   If `Opaque`, cull if $B$ is opaque.
        *   If `TransparentMerged(T)`, cull if $B$ and $B_{neighbor}$ are transparent and their `TransparentGroup` values are equal.
        *   If `TransparentUnmerged`, do not cull against transparent neighbors.
5.  **Compute Lighting:** For each visible face, retrieve light level from $B$.`light()` (which may vary by face, e.g., for Cactus) [S1, unofficial].
6.  **Add to Mesh:** Add vertices and indices to the chunk's vertex/index lists.
7.  **Finalize:** Once all voxels are processed, upload vertex/index data to the GPU [S3, unofficial].

**Pitfalls:**
*   **Fluid Shape Missing:** Setting `FLUID_ENABLED` to true without implementing fluid shape causes a debug assertion failure rather than silently drawing fluids as cubes [S1, unofficial].
*   **Threading:** In Unity3D, mesh creation (placing data into `Mesh` object) must occur on the main thread, even if geometry calculation is done on a background thread [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk_mesh(chunk: Chunk, block_registry: Dict[BlockID, BlockConfig]):
    vertices = []
    indices = []
    
    for x, y, z in chunk.occupied_voxels():
        block_id = chunk.get_block(x, y, z)
        config = block_registry[block_id]
        
        # Get shape and culling mode
        shape = config.shape()
        cull_mode = config.cull_mode()
        
        # Generate faces based on shape
        faces = shape.generate_faces()
        
        for face in faces:
            # Determine visibility
            neighbor_pos = face.position + face.normal
            neighbor_id = chunk.get_block_at(neighbor_pos)
            
            if neighbor_id is None:
                visible = True
            else:
                neighbor_config = block_registry[neighbor_id]
                visible = not cull_mode.should_cull(config, neighbor_config)
            
            if visible:
                # Compute lighting
                light_level = config.light(face)
                
                # Generate vertex data
                verts = shape.generate_vertices(face, light_level)
                indices.extend(shape.generate_indices(face))
                vertices.extend(verts)
    
    # Finalize mesh (must be on main thread in Unity)
    mesh = create_unity_mesh(vertices, indices)
    return mesh
```

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers: 'opaque', 'water', and 'transparentSolid'. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right) [S2, unofficial]. Instance creation iterates visible faces; if the face is Left, Right, Top, or Front, `drawSlope` is set to true and tint is assigned from `tints[0]` (Top/Front), `tints[1]` (Left), or `tints[2]` (Right) [S2, unofficial]. If a specific tint is `Vector3.One`, it falls back to another face's tint [S2, unofficial]. Slope vertices are defined as: (0,1,0) UV[0], (0,0,1) UV[1], (1,1,0) UV[2], (1,1,0) UV[3], (0,0,1) UV[4], (1,0,1) UV[5] [S2, unofficial]. Left triangle vertices: (0,0,1) UV[0], (0,1,0) UV[1], (0,0,0) UV[2] [S2, unofficial]. Right triangle vertices: (1,0,0) UV[0], (1,1,0) UV[1], (1,0,1) UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match [S1, unofficial]. Two adjacent fluid blocks join their surfaces only when their `FluidInfo::id` (u8) values are equal [S1, unofficial].

### Geometry Builder Pattern
A design pattern using an interface to control the type of geometry created for each non-empty voxel, allowing different geometry types (cubes, ramps, X-shapes) to be generated via polymorphism without large switch statements [S3, unofficial]. The interface `IGeometryBuilder` contains a single function `BuildGeometry()` [S3, unofficial]. During mesh creation, `BuildGeometry()` is called once for each non-empty block, passing a reference to the chunk object to add vertices and indices to the chunk's linked list [S3, unofficial]. After all blocks are processed, the data is placed into a Unity3D Mesh object on the main thread [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, assuming constant-time shape generation per voxel.
*   **Memory Complexity:** $O(V)$ where $V$ is the number of visible faces. The vertex structure is larger than a simple cube vertex due to additional fields (AO, light, block ID, atlas offset) [S1, unofficial].
*   **Strengths:** Highly flexible; supports complex shapes and fine-grained transparency control without modifying the core meshing algorithm [S1, S3, unofficial].
*   **Limitations:** Increased per-vertex memory usage; requires careful management of threading (e.g., Unity main thread constraint) [S3, unofficial].
*   **When to use:** When the voxel world contains diverse block types with different geometries and transparency requirements (e.g., Minecraft-like worlds) [S1, S3, unofficial].
*   **When not to use:** For simple, uniform voxel worlds where performance is critical and shape diversity is low [S1, unofficial].

## Background
Originated in implementations of voxel meshing for games like Minecraft, where block diversity necessitated a more flexible meshing approach than simple cube generation [S1, S3, unofficial]. The trait-based interface and geometry builder pattern were introduced to avoid large switch statements and support complex shapes like ramps and fluids [S3, unofficial]. The three-valued opacity classification addresses the need for distinct rendering pipelines for fluids and transparent solids [S1, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
