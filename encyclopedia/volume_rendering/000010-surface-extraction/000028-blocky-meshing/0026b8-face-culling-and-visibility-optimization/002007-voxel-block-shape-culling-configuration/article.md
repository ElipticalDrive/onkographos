# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
Voxel Block Shape & Culling Configuration is a trait-based system that defines the geometry, transparency culling behavior, and lighting properties for individual block types in a voxel mesh generator [S1, unofficial]. It solves the problem of hard-coded geometry and culling logic by decoupling block identity from mesh generation, allowing different block types (e.g., opaque stone, transparent glass, sloped ramps) to specify their own rendering characteristics [S1, unofficial][S3, unofficial]. Unlike generic mesh generation that treats all voxels as cubes, this configuration allows for complex shapes and fine-grained culling modes (such as merging transparent faces) to be defined per block type [S1, unofficial].

## How it works
The system operates by associating a configuration object (often a trait or interface implementation) with each block type. This configuration dictates how the block contributes to the mesh.

### Inputs and Outputs
*   **Input:** The block type identifier (e.g., `MyBlock::Glass`) and the face orientation being evaluated (e.g., `AlignedFace::PosX`).
*   **Output:** A culling mode enum, a shape definition (geometry), and lighting values.

### Data Structures
The core data structure is a trait or interface that exposes methods for shape, culling, and lighting.

**CullMode Enum (Example from [S1, unofficial]):**
*   `Empty`: The block is air; no geometry is generated.
*   `Opaque`: Standard face culling; shared faces with solid neighbors are culled.
*   `TransparentMerged(T)`: Shared faces between two transparent blocks are culled if their `TransparentGroup` values match. This reduces overdraw for similar transparent materials.
*   `TransparentUnmerged`: Faces are never culled against other transparent blocks, even if they share the same material, to preserve visual separation (e.g., for alpha blending artifacts).

**Shape Enum (Example from [S1, unofficial]):**
*   `WholeBlock`: A standard 1x1x1 cube.
*   `Cross(CrossInfo)`: A cross-shaped geometry (e.g., for fences/chains) defined by face and stretch.
*   *Note: Other shapes like ramps or fluids are handled via specific builders or specialized shape IDs [S2, unofficial][S1, unofficial].*

**Vertex Layout (Example from [S1, unofficial]):**
Vertices carry per-vertex attributes to support lighting and texturing:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct Vertex {
    pos: Vec4<f32>,      // Position
    uv: Vec2<f32>,       // Texture coordinates
    normal: Vec3<f32>,   // Surface normal
    ao: f32,             // Ambient occlusion (0.0 = occluded, 1.0 = lit)
    smooth_light: f32,   // Smooth lighting value
    block: MyBlock,      // Block ID for render pass filtering
    atlas_offset: f32,   // Atlas tile offset
    two_sided: bool,     // Flag for two-sided rendering
}
```

### Numbered Steps
1.  **Block Identification:** The mesh generator iterates through non-empty voxels in a chunk.
2.  **Trait Lookup:** For each voxel, the system retrieves its associated configuration trait.
3.  **Culling Decision:** The `cull_mode()` method is called. If the mode is `Empty`, the block is skipped. If `Opaque`, standard neighbor checks apply. If `TransparentMerged`, the system checks if the neighbor has the same `TransparentGroup` ID; if so, the shared face is culled.
4.  **Geometry Generation:** The `shape()` method determines the geometry. For standard blocks, this returns `WholeBlock`. For complex blocks (e.g., chains), it returns specific shape definitions.
5.  **Lighting Calculation:** The `light()` method returns lighting values based on the block type and face orientation. For example, cactus blocks may have different lighting for top vs. side faces [S1, unofficial].
6.  **Mesh Assembly:** Vertices and indices are generated based on the shape and added to the chunk's vertex buffer.

### Formulas
*   **Transparent Merging Condition:** A face between block $A$ and block $B$ is culled if:
    $$ \text{CullMode}(A) = \text{TransparentMerged}(T_A) \land \text{CullMode}(B) = \text{TransparentMerged}(T_B) \land T_A = T_B $$
    where $T$ is the transparent group identifier [S1, unofficial].

### Parameters
*   **TransparentGroup:** An identifier (e.g., `()`, `u8`) used to group transparent materials. In the example, `Glass` uses `TransparentMerged(())` [S1, unofficial].
*   **Fluid ID:** For fluid stitching, adjacent fluid blocks join surfaces if their `FluidInfo::id` (u8) matches [S1, unofficial].

### Edge Cases and Pitfalls
*   **Fluid Shape Assertion:** Setting `FLUID_ENABLED` to true without implementing the fluid shape logic causes a debug assertion failure rather than silently rendering fluids as cubes [S1, unofficial].
*   **Threading Constraints:** In Unity3D implementations, while vertex/index calculation can be done on a background thread, the final `Mesh` object creation must be performed on the main thread [S3, unofficial].
*   **Tint Fallback:** For ramp shapes, if a specific face's tint is `Vector3.One`, the system falls back to another face's tint [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk_mesh(chunk):
    vertices = []
    indices = []
    
    for block_pos in chunk.non_empty_blocks():
        block_type = chunk.get_block(block_pos)
        config = get_block_config(block_type)
        
        # Step 1: Check Culling Mode
        cull_mode = config.cull_mode()
        if cull_mode == CullMode.Empty:
            continue
            
        # Step 2: Determine Shape
        shape = config.shape(block_pos, chunk)
        
        # Step 3: Generate Geometry based on Shape
        if shape == Shape.WholeBlock:
            generate_cube_geometry(block_pos, cull_mode, config, vertices, indices, chunk)
        elif shape == Shape.Ramp:
            generate_ramp_geometry(block_pos, config, vertices, indices)
        elif shape == Shape.Fluid:
            generate_fluid_geometry(block_pos, config, vertices, indices, chunk)
        else:
            # Handle other shapes (Cross, etc.)
            pass
            
    # Step 4: Finalize Mesh (Main Thread in Unity)
    return create_mesh_object(vertices, indices)

def generate_cube_geometry(pos, cull_mode, config, vertices, indices, chunk):
    for face in [PosX, NegX, PosY, NegY, PosZ, NegZ]:
        neighbor = chunk.get_neighbor(pos, face)
        
        # Apply Culling Logic
        if should_cull_face(cull_mode, config, neighbor, face):
            continue
            
        # Get Lighting
        light_val = config.light(face)
        
        # Generate Vertices
        v = create_face_vertices(pos, face, config)
        v.ao = calculate_ao(pos, face, chunk)
        v.smooth_light = light_val
        v.block = config.block_type
        
        vertices.extend(v)
        indices.extend(get_face_indices())

def should_cull_face(cull_mode, config, neighbor, face):
    if cull_mode == CullMode.Opaque:
        return neighbor.is_solid()
    elif cull_mode == CullMode.TransparentMerged:
        if not neighbor.is_transparent():
            return False
        # Check TransparentGroup match
        return config.transparent_group == neighbor.config.transparent_group
    elif cull_mode == CullMode.TransparentUnmerged:
        return False
    return False
```

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers: 'opaque', 'water', and 'transparentSolid'. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### [VoxelBlockyTypeLibrary with Attributes](12820-voxelblockytypelibrary-with-attributes.md)
`VoxelBlockyTypeLibrary` is a model management system for blocky voxel terrains that organizes meshes by logical "block types" with configurable state attributes, rather than storing individual model references directly. It solves the problem of managing complex block behaviors (e.g., orientation, powered state, connections) by automatically generating unique numerical IDs for every combination of attribute values, similar to Minecraft's block state system. It differs from the standard `VoxelBlockyLibrary` by decoupling model identity from manual ID assignment, using a combinatorial attribute system instead.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right) [S2, unofficial]. Instance creation involves iterating visible faces; if the face is Left, Right, Top, or Front, `drawSlope` is set to true and tints are assigned based on face type (Top/Front to `tints[0]`, Left to `tints[1]`, Right to `tints[2]`) [S2, unofficial]. If a specific tint is `Vector3.One`, it falls back to another face's tint [S2, unofficial]. Back and Bottom faces use `cubeBackFaceShapeID` or `cubeBottomFaceShapeID` [S2, unofficial]. Slope vertices are defined as: (0,1,0) UV[0], (0,0,1) UV[1], (1,1,0) UV[2], (1,1,0) UV[3], (0,0,1) UV[4], (1,0,1) UV[5] [S2, unofficial]. Left triangle vertices: (0,0,1) UV[0], (0,1,0) UV[1], (0,0,0) UV[2] [S2, unofficial]. Right triangle vertices: (1,0,0) UV[0], (1,1,0) UV[1], (1,0,1) UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match [S1, unofficial]. Two adjacent fluid blocks join their surfaces only when their `FluidInfo::id` (u8) values are equal [S1, unofficial].

### Geometry Builder Pattern
A design pattern using an interface to control the type of geometry created for each non-empty voxel, allowing different geometry types (cubes, ramps, X-shapes) to be generated via polymorphism without large switch statements [S3, unofficial]. The interface `IGeometryBuilder` contains a single function `BuildGeometry()` [S3, unofficial]. During mesh creation, `BuildGeometry()` is called once for each non-empty block, passing a reference to the chunk object [S3, unofficial]. Inside `BuildGeometry()`, vertices and indices are added to the chunk's linked list [S3, unofficial]. After all blocks are processed, the vertex and index data is placed into a Unity3D Mesh object on the main thread [S3, unofficial]. A pitfall is that Unity3D functions must be called on the main thread, so while calculation is done on a background thread, final mesh creation must be moved to the main thread [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is processed once to determine its geometry and culling status [S3, unofficial].
*   **Memory Complexity:** $O(V + I)$ where $V$ is the number of vertices and $I$ is the number of indices generated. The configuration data itself is minimal, stored per block type [S1, unofficial].
*   **Strengths:** High flexibility for custom block shapes and culling behaviors; reduces overdraw for transparent materials via merging; decouples block logic from mesh generation code [S1, unofficial][S3, unofficial].
*   **Limitations:** Requires careful implementation of threading constraints in certain engines (e.g., Unity) [S3, unofficial]; complex shapes (ramps, fluids) require additional geometry handling logic [S2, unofficial][S1, unofficial].
*   **When to Use:** When rendering voxel worlds with diverse block types (e.g., Minecraft-like) that require specific visual treatments (transparency, slopes, fluids) and optimized rendering [S1, unofficial][S3, unofficial].
*   **When Not to Use:** For simple, uniform voxel grids where all blocks are identical cubes and no transparency or complex geometry is needed [S3, unofficial].

## Background
The concept of configuring block geometry and culling per type originated in voxel game development, particularly in games like Minecraft, where different blocks have distinct shapes and rendering properties [S3, unofficial]. The trait-based interface approach was formalized in projects like `voxmesh` to provide a clean, extensible way to manage these configurations [S1, unofficial]. The Geometry Builder pattern was introduced in academic work on procedural voxel generation to handle complex shapes without cluttering the main mesh generation loop [S3, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
