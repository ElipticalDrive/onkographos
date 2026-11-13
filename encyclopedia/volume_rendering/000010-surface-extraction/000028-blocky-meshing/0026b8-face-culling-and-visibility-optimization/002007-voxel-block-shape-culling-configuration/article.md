# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
Voxel Block Shape & Culling Configuration is a trait-based system that defines block geometry, transparency culling modes, and lighting properties per block type. It solves the problem of heterogeneous voxel rendering by allowing different block types to specify their own mesh generation rules and visibility constraints. Unlike standard boolean transparency flags, this system distinguishes between opaque, transparent-merged, and transparent-unmerged blocks, and supports complex geometries (ramps, fluids) via polymorphic builders. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by providing the per-block metadata required to execute those culling decisions.

## How it works
The system operates by associating a configuration trait with each block type. This trait dictates the shape of the mesh, the culling mode for its faces, and its lighting behavior.

### Inputs and Outputs
*   **Input**: Block type identifier (e.g., `MyBlock::Glass`), voxel position, and neighbor occupancy data.
*   **Output**: A set of vertices and indices for the mesh, along with metadata (lighting, UV offsets) for rendering.

### Data Structures

#### Block Trait Interface
Each block type implements a trait (or interface) that returns three key values: `shape`, `cull_mode`, and `light`.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
// Conceptual Rust-like trait definition
trait BlockConfig {
    fn shape(&self, face: Face) -> Shape;
    fn cull_mode(&self) -> CullMode;
    fn light(&self, face: Face) -> f32;
}
```

#### Shape Enum
The `Shape` enum defines the geometry to generate. Common variants include:
*   `WholeBlock`: A standard 1x1x1 cube.
*   `Cross(CrossInfo)`: A cross-shaped block (e.g., chains, rails) defined by an aligned face and stretch value.
*   `Ramp`: A sloped block (see Subtopics).
*   `Fluid`: A fluid surface mesh (see Subtopics).

#### CullMode Enum
The `CullMode` enum determines how faces are culled based on neighbor occupancy.

*   `Empty`: The block is air; no faces are generated.
*   `Opaque`: Standard culling. A face is generated if the neighbor is empty.
*   `TransparentMerged(T)`: Shared faces between two transparent blocks are culled if their `TransparentGroup` values are equal. This prevents double-rendering of adjacent transparent surfaces.
*   `TransparentUnmerged`: Faces are never merged with neighbors, even if both are transparent. Used for blocks like leaves or glass where internal faces might be visible or require specific rendering order.

#### Vertex Layout
Vertices contain position, UV, normal, ambient occlusion (AO), smooth light, block ID, atlas offset, and a two-sided flag.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct Vertex {
    pos: Vec4<f32>,       // World position
    uv: Vec2<f32>,        // Texture coordinates
    normal: Vec3<f32>,    // Face normal
    ao: f32,              // Ambient occlusion (0.0 = occluded, 1.0 = lit)
    smooth_light: f32,    // Smooth lighting value
    block: MyBlock,       // Block type ID for render pass filtering
    atlas_offset: f32,    // Pre-computed atlas tile offset
    two_sided: bool,      // Whether the quad is two-sided
}
```

### Numbered Steps

1.  **Iterate Voxels**: For each voxel in the chunk, determine its block type.
2.  **Retrieve Configuration**: Call the block's `cull_mode()` and `shape()` methods.
3.  **Check Culling**:
    *   If `cull_mode` is `Empty`, skip the voxel.
    *   For each of the 6 faces, check the neighbor's occupancy.
    *   If the neighbor is solid and the `cull_mode` is `Opaque`, cull the face.
    *   If the `cull_mode` is `TransparentMerged(T)`, check if the neighbor has the same `TransparentGroup`. If so, cull the face.
    *   If the `cull_mode` is `TransparentUnmerged`, do not cull based on neighbor transparency (only cull if neighbor is solid and opaque, depending on implementation details).
4.  **Generate Geometry**:
    *   Call `BuildGeometry()` on the appropriate `IGeometryBuilder` for the block's shape.
    *   Pass the chunk reference to the builder.
    *   The builder adds vertices and indices to the chunk's linked list.
5.  **Assign Lighting**: For each generated face, call `light(face)` to determine the lighting value.
6.  **Finalize Mesh**: After all voxels are processed, transfer the vertex and index data from the linked list to a GPU mesh object on the main thread.

### Formulas

*   **Neighbor Position**: $P_{neighbor} = P + N$, where $P$ is the voxel position and $N$ is the face normal.
*   **Frustum Culling for Chunks**: A chunk is visible if its bounding box intersects the view frustum. The frustum planes are extracted from the view-projection matrix $VP$:
    *   Near: $Col_3$
    *   Far: $Col_4 - Col_3$
    *   Top: $Col_4 - Col_2$
    *   Bottom: $Col_4 + Col_2$
    *   Left: $Col_4 + Col_1$
    *   Right: $Col_4 - Col_1$
    A box is in the frustum if all its vertices satisfy $N \cdot V + D \ge 0$ for all plane normals $N$ and distances $D$.

### Parameters

*   **Fluid ID**: A `u8` identifier for fluid blocks. Adjacent fluid blocks join their surfaces only if their IDs match.
*   **TransparentGroup**: A value associated with `TransparentMerged` blocks. Blocks with the same group value merge their transparent faces.

### Edge Cases and Pitfalls

*   **Fluid Shape**: Setting `FLUID_ENABLED` to true without implementing fluid shape causes a debug assertion failure rather than silently drawing fluids as cubes [S1, unofficial].
*   **Threading**: Unity3D functions must be called on the main thread. Vertex/index calculation can be done on a background thread, but the final `Mesh` object creation must be moved to the main thread [S3, unofficial].
*   **Tint Fallback**: For ramp shapes, if a specific tint is `Vector3.One`, fallback to another face's tint [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk_mesh(chunk):
    vertices = []
    indices = []
    
    for x, y, z in chunk.voxels:
        block_type = chunk.get_block(x, y, z)
        config = block_type.get_config()
        
        if config.cull_mode == CullMode.Empty:
            continue
            
        shape = config.shape(Face.All)
        builder = get_geometry_builder(shape)
        
        for face in Face.All:
            neighbor_pos = (x + face.normal.x, y + face.normal.y, z + face.normal.z)
            neighbor_block = chunk.get_block(*neighbor_pos)
            
            if is_face_culled(block_type, neighbor_block, face, config.cull_mode):
                continue
                
            # Generate geometry for this face
            builder.build_geometry(chunk, x, y, z, face, vertices, indices)
            
            # Assign lighting
            light_value = config.light(face)
            apply_lighting_to_vertices(vertices, indices, light_value)
            
    # Transfer to GPU on main thread
    gpu_mesh = create_mesh(vertices, indices)
    return gpu_mesh

def is_face_culled(block_type, neighbor_block, face, cull_mode):
    if neighbor_block is None or neighbor_block.is_air():
        return False
        
    if cull_mode == CullMode.Opaque:
        return neighbor_block.is_solid()
        
    if cull_mode == CullMode.TransparentMerged:
        if block_type.transparent_group == neighbor_block.transparent_group:
            return True
        return False
        
    if cull_mode == CullMode.TransparentUnmerged:
        return False # Never cull transparent faces based on neighbor
        
    return False
```

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers ('opaque', 'water', 'transparentSolid') to determine rendering material and buffer assignment. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### [VoxelBlockyTypeLibrary with Attributes](12820-voxelblockytypelibrary-with-attributes.md)
`VoxelBlockyTypeLibrary` is a model management system for blocky voxel terrains that organizes meshes by logical "block types" with configurable state attributes, rather than storing individual model references directly. It solves the problem of managing complex block behaviors (e.g., orientation, powered state, connections) by automatically generating unique numerical IDs for every combination of attribute values, similar to Minecraft's block state system. It differs from the standard `VoxelBlockyLibrary` by decoupling model identity from manual ID assignment, using a combinatorial attribute system instead.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right) [S2, unofficial]. Instance creation involves iterating visible faces. If the face is Left, Right, Top, or Front, `drawSlope` is set to true and tint is assigned to `tints[0]` (Top/Front), `tints[1]` (Left), or `tints[2]` (Right). If a specific tint is `Vector3.One`, fallback to another face's tint. Instances are added for slope, left triangle, and right triangle using respective shape IDs and tints. If the face is Back or Bottom, an instance is added using `cubeBackFaceShapeID` or `cubeBottomFaceShapeID` [S2, unofficial]. Slope vertices are: (0,1,0) UV[0], (0,0,1) UV[1], (1,1,0) UV[2], (1,1,0) UV[3], (0,0,1) UV[4], (1,0,1) UV[5]. Left triangle vertices: (0,0,1) UV[0], (0,1,0) UV[1], (0,0,0) UV[2]. Right triangle vertices: (1,0,0) UV[0], (1,1,0) UV[1], (1,0,1) UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match [S1, unofficial]. Two adjacent fluid blocks join their surfaces only when their `FluidInfo::id` (u8) values match [S1, unofficial].

### Geometry Builder Pattern
A design pattern using an interface to control the type of geometry created for each non-empty voxel, allowing different geometry types (cubes, ramps, X-shapes) to be generated via polymorphism without large switch statements [S3, unofficial]. The interface `IGeometryBuilder` contains a single function: `BuildGeometry()`. During mesh creation, `BuildGeometry()` is called once for each non-empty block, passing a reference to the chunk object. The builder uses this reference to add vertices and indices to the chunk's linked list. After all blocks are processed, the data is placed into a Unity3D Mesh object on the main thread [S3, unofficial]. Pitfall: Unity3D functions must be called on the main thread; vertex/index calculation can be done on a background thread, but final Mesh creation must be on the main thread [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(V)$ where $V$ is the number of voxels in the chunk, as each voxel is processed once.
*   **Memory Complexity**: $O(F)$ where $F$ is the number of visible faces, as only visible faces are stored in the mesh.
*   **Performance**: Efficient for large chunks due to batching and culling. However, complex shapes (ramps, fluids) may increase vertex count and reduce culling efficiency.
*   **Strengths**: Flexible support for diverse block types and geometries. Polymorphic builders simplify code maintenance.
*   **Limitations**: Requires careful management of threading (especially in Unity). Debug assertions can fail if configurations are incomplete (e.g., fluid shapes).
*   **When to Use**: When rendering voxel worlds with heterogeneous block types and complex geometries.
*   **When Not to Use**: For simple, uniform voxel worlds where a single cube shape suffices.

## Background
The concept of block traits and shape enums originated in voxel meshing libraries like `finnbear/voxmesh` [S1, unofficial]. The Geometry Builder pattern was introduced in academic work on procedural voxel generation, such as the "VoxBox" project [S3, unofficial]. Ramp shapes and fluid stitching are common in games like Minecraft and its clones [S2, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
