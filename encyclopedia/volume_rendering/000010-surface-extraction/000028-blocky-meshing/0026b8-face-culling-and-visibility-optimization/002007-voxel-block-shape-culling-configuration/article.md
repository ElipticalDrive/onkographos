# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
Voxel Block Shape & Culling Configuration is a trait-based system that defines the geometry, transparency culling behavior, and lighting properties for individual block types in a voxel mesh generator. It solves the problem of heterogeneous block rendering by allowing each block type to declare its own shape (e.g., whole cube, ramp, fluid) and how it interacts with neighbors during face culling. It differs from generic meshing algorithms by encapsulating block-specific logic (such as fluid surface stitching or transparent merging) directly into the block's configuration, rather than handling these cases via global flags or complex switch statements in the meshing loop.

## How it works
The system operates by associating a configuration object (often a trait or interface implementation) with each block type. This configuration dictates the output geometry and the culling mode applied during mesh generation.

### Inputs and Outputs
*   **Input:** A block type identifier (e.g., `MyBlock::Glass`, `MyBlock::Water`) and the current face direction being evaluated.
*   **Output:** A `Shape` enum value defining the vertex layout, a `CullMode` enum defining neighbor interaction rules, and lighting parameters.

### Data Structures

#### Shape Enum
Defines the geometric primitive for a block face.
*   `WholeBlock`: A standard 1x1x1 cube face.
*   `Cross(CrossInfo)`: A cross-shaped face, used for thin blocks like chains or rails.
    *   `face`: The aligned face direction (`PosX`, `NegY`, etc.).
    *   `stretch`: The stretch factor (e.g., 0 for full cross, 1 for half).
*   `Ramp`: A sloped face composed of a slope quad and side triangles.
*   `Fluid`: A fluid surface face.

#### CullMode Enum
Defines how faces are culled based on neighbor occupancy and transparency.
*   `Empty`: No geometry is generated.
*   `Opaque`: Standard topological culling; faces are visible if the neighbor is empty.
*   `TransparentMerged(T)`: Culls shared faces between two transparent blocks if their `TransparentGroup` values are equal. Used for blocks that should merge visually (e.g., glass panes or specific fluid types).
*   `TransparentUnmerged`: Faces are never merged with neighbors; always rendered if visible. Used for blocks like leaves or ladders that require distinct edges.

#### Vertex Layout
Vertices contain the following fields [S1, unofficial]:
*   `pos`: `Vec4<f32>` (Position).
*   `uv`: `Vec2<f32>` (Texture coordinates).
*   `normal`: `Vec3<f32>` (Surface normal).
*   `ao`: `f32` (Ambient Occlusion, 0.0 = occluded, 1.0 = lit).
*   `smooth_light`: `f32` (Smooth lighting value).
*   `block`: Block identifier (for render pass filtering).
*   `atlas_offset`: `f32` (Pre-computed atlas tile offset).
*   `two_sided`: `bool` (Whether the quad is two-sided).

### Steps

1.  **Block Configuration Lookup:** For each block type, retrieve its associated configuration object.
2.  **Shape Determination:** Call the `shape()` method on the configuration.
    *   If the block is a standard cube, return `Shape::WholeBlock`.
    *   If the block is a thin structure (e.g., chain), return `Shape::Cross` with appropriate face and stretch parameters.
    *   If the block is a fluid, return `Shape::Fluid`.
3.  **Culling Mode Determination:** Call the `cull_mode()` method.
    *   Return `CullMode::Empty` for air.
    *   Return `CullMode::Opaque` for solid blocks (cobblestone, clay).
    *   Return `CullMode::TransparentMerged(())` for blocks that merge with identical transparent neighbors (glass).
    *   Return `CullMode::TransparentUnmerged` for blocks that do not merge (leaves, ladders, chains).
4.  **Geometry Generation:**
    *   For `Shape::WholeBlock`: Generate 6 faces (if not culled).
    *   For `Shape::Cross`: Generate the cross-shaped geometry based on the `CrossInfo`.
    *   For `Shape::Ramp`: Generate the slope quad and side triangles.
    *   For `Shape::Fluid`: Generate the top surface, potentially stitching with neighbors if `FluidInfo::id` matches.
5.  **Lighting Assignment:** Call the `light()` method to assign per-face lighting values.
    *   Return a base lighting value for the block.
    *   Override for specific faces (e.g., cactus top vs. side).

### Formulas
*   **Culling Condition for `TransparentMerged`:**
    A face between block $A$ and neighbor $B$ is culled if:
    $$ \text{CullMode}(A) = \text{TransparentMerged}(T) \land \text{CullMode}(B) = \text{TransparentMerged}(T) \land T_A = T_B $$
    Where $T$ is the transparent group identifier.

*   **Fluid Stitching Condition:**
    Two adjacent fluid blocks join their surfaces if:
    $$ \text{FluidInfo::id}(A) == \text{FluidInfo::id}(B) $$

### Parameters
*   `FLUID_ENABLED`: Boolean flag. If `true` but fluid shape is not implemented, causes debug assertion failure [S1, unofficial].
*   `TransparentGroup`: Identifier for merging transparent blocks.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_block_config(block_type: BlockType) -> BlockConfig:
    """Retrieve the configuration for a specific block type."""
    if block_type == BlockType.Air:
        return BlockConfig(cull_mode=CullMode.Empty, shape=Shape.Empty)
    elif block_type in [BlockType.Cobblestone, BlockType.Clay, BlockType.Lamp]:
        return BlockConfig(cull_mode=CullMode.Opaque, shape=Shape.WholeBlock)
    elif block_type == BlockType.Glass:
        return BlockConfig(cull_mode=CullMode.TransparentMerged(group=1), shape=Shape.WholeBlock)
    elif block_type in [BlockType.Leaves, BlockType.Ladder, BlockType.ChainY]:
        return BlockConfig(cull_mode=CullMode.TransparentUnmerged, shape=Shape.WholeBlock)
    elif block_type == BlockType.Cactus:
        return BlockConfig(cull_mode=CullMode.Opaque, shape=Shape.WholeBlock)
    elif block_type == BlockType.Water:
        return BlockConfig(cull_mode=CullMode.TransparentMerged(group=2), shape=Shape.Fluid)
    else:
        return BlockConfig(cull_mode=CullMode.Opaque, shape=Shape.WholeBlock)

def cull_face(block_type: BlockType, neighbor_type: BlockType, face_direction: FaceDirection) -> bool:
    """Determine if a face should be culled based on block configurations."""
    block_config = get_block_config(block_type)
    neighbor_config = get_block_config(neighbor_type)
    
    # Empty blocks are always culled
    if block_config.cull_mode == CullMode.Empty:
        return True
        
    # Opaque blocks: cull if neighbor is also opaque
    if block_config.cull_mode == CullMode.Opaque:
        return neighbor_config.cull_mode == CullMode.Opaque
        
    # Transparent Merged: cull if neighbor is same type and same group
    if block_config.cull_mode == CullMode.TransparentMerged:
        if neighbor_config.cull_mode == CullMode.TransparentMerged:
            if block_config.transparent_group == neighbor_config.transparent_group:
                return True
        return False
        
    # Transparent Unmerged: never cull based on neighbor transparency
    if block_config.cull_mode == CullMode.TransparentUnmerged:
        return False
        
    return False

def generate_geometry(block_type: BlockType, face_direction: FaceDirection) -> MeshData:
    """Generate mesh data for a specific block face."""
    block_config = get_block_config(block_type)
    shape = block_config.shape
    
    if shape == Shape.WholeBlock:
        return generate_cube_face(block_type, face_direction)
    elif shape == Shape.Cross:
        return generate_cross_face(block_type, face_direction)
    elif shape == Shape.Fluid:
        return generate_fluid_face(block_type, face_direction)
    elif shape == Shape.Ramp:
        return generate_ramp_face(block_type, face_direction)
    else:
        return MeshData.empty()
```

### Edge Cases and Pitfalls
*   **Fluid Shape Implementation:** Setting `FLUID_ENABLED` to `true` without implementing the fluid shape logic causes a debug assertion failure rather than silently rendering fluids as cubes [S1, unofficial].
*   **Threading Constraints:** In Unity3D environments, vertex/index calculation can be done on a background thread, but the final `Mesh` object creation must be moved to the main thread [S3, unofficial].
*   **Transparent Group Mismatch:** If two transparent blocks have different `TransparentGroup` values, they will not merge, potentially causing visual artifacts or increased draw calls.

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers: 'opaque', 'water', and 'transparentSolid'. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### [VoxelBlockyTypeLibrary with Attributes](12820-voxelblockytypelibrary-with-attributes.md)
`VoxelBlockyTypeLibrary` is a model management system for blocky voxel terrains that organizes meshes by logical "block types" with configurable state attributes, rather than storing individual model references directly. It solves the problem of managing complex block behaviors (e.g., orientation, powered state, connections) by automatically generating unique numerical IDs for every combination of attribute values, similar to Minecraft's block state system. It differs from the standard `VoxelBlockyLibrary` by decoupling model identity from manual ID assignment, using a combinatorial attribute system instead.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right). Instance creation involves iterating visible faces. If the face is Left, Right, Top, or Front, `drawSlope` is set to `true` and tint is assigned to `tints[0]` (Top/Front), `tints[1]` (Left), or `tints[2]` (Right). If a specific tint is `Vector3.One`, it falls back to another face's tint. Instances are added for slope, left triangle, and right triangle using respective shape IDs and tints. If the face is Back or Bottom, an instance is added using `cubeBackFaceShapeID` or `cubeBottomFaceShapeID` [S2, unofficial].
*   **Slope vertices (6):** (0,1,0) UV[0], (0,0,1) UV[1], (1,1,0) UV[2], (1,1,0) UV[3], (0,0,1) UV[4], (1,0,1) UV[5].
*   **Left triangle vertices (3):** (0,0,1) UV[0], (0,1,0) UV[1], (0,0,0) UV[2].
*   **Right triangle vertices (3):** (1,0,0) UV[0], (1,1,0) UV[1], (1,0,1) UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match. Two adjacent fluid blocks join their surfaces only when their `FluidInfo::id` (u8) values are equal [S1, unofficial].

### Geometry Builder Pattern
A design pattern using an interface (`IGeometryBuilder`) to control the type of geometry created for each non-empty voxel, allowing different geometry types (cubes, ramps, X-shapes) to be generated via polymorphism without large switch statements. During mesh creation, `BuildGeometry()` is called once for each non-empty block, passing a reference to the chunk object. The builder adds vertices and indices to the chunk's linked list. After all blocks are processed, the data is placed into a Unity3D Mesh object on the main thread. Calculations are done on a background thread, but Unity3D functions must be called on the main thread [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of blocks, as each block is processed once to determine its shape and culling status.
*   **Memory Complexity:** $O(N)$ for storing the mesh data, plus $O(1)$ for the block configuration lookup table.
*   **Strengths:** Modular design allows easy addition of new block types without modifying the core meshing algorithm. Efficient culling reduces draw calls significantly in dense scenes.
*   **Limitations:** Requires careful management of transparent groups to avoid visual artifacts. Threading constraints in some engines (like Unity3D) require careful synchronization for mesh creation.
*   **When to use:** When rendering voxel worlds with heterogeneous block types requiring different geometries and rendering behaviors (e.g., Minecraft-like games).
*   **When not to use:** For simple, uniform voxel terrains where all blocks are identical cubes.

## Background
The concept of block-specific configuration in voxel rendering evolved from the need to support diverse block types in games like Minecraft. Early implementations used simple boolean flags for transparency, which proved insufficient for complex interactions like fluid rendering and leaf occlusion. The trait-based interface and shape enum system emerged as a more flexible solution, allowing for polymorphic geometry generation and fine-grained control over culling and lighting. This approach is widely used in modern voxel engines to balance performance and visual fidelity.

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
