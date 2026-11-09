# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
This method defines the geometric and rendering properties of individual voxel types within a blocky meshing pipeline. It solves the problem of heterogeneous block geometry (e.g., cubes, ramps, fluids) and varying transparency requirements by associating each block type with a specific shape definition and culling mode. It sets apart from sibling methods like [Face Culling](8735-face-culling.md) by operating at the per-block-type level rather than the per-voxel-instance level, determining *how* a voxel is drawn and *which* of its faces are eligible for culling before the instance-specific visibility checks occur.

## How it works
The system uses a trait-based or interface-based configuration where each block type (e.g., Stone, Glass, Water) implements a set of properties: `Shape`, `CullMode`, and `Light`.

### Inputs and Outputs
- **Input**: Block type identifier (enum or ID).
- **Output**: A configuration object containing:
    - `Shape`: The geometric primitive(s) to generate.
    - `CullMode`: The rule for merging or hiding faces between adjacent blocks.
    - `Light`: Per-face or per-block lighting values.

### Data Structures

#### Block Configuration Trait/Interface
Each block type implements a configuration interface. In Rust-like syntax (from [S1, unofficial]):

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
trait BlockConfig {
    fn shape(&self) -> Shape;
    fn cull_mode(&self) -> CullMode;
    fn light(&self, face: Face) -> f32;
}
```

#### Shape Enum
Defines the geometry. From [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
enum Shape {
    WholeBlock,
    Cross(CrossInfo), // e.g., for chains, rails
    // Other shapes like Ramp, Fluid, etc.
}
```

#### CullMode Enum
Defines transparency handling. From [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
enum CullMode {
    Empty,       // No faces generated (e.g., Air)
    Opaque,      // Standard culling; faces hidden if neighbor is solid
    TransparentMerged(T), // Faces hidden if neighbor is same transparent group
    TransparentUnmerged,  // Faces always generated (no merging)
}
```
- `TransparentMerged(T)`: Culls shared faces between two transparent blocks when their `TransparentGroup` values are equal. It does not replace `Self: PartialEq` for greedy meshing, implying greedy meshing must still handle adjacency logic separately or this mode is used in conjunction with it [S1, unofficial].
- `TransparentUnmerged`: Used for blocks like leaves, cactus, or chains where internal faces should not be merged with identical neighbors to preserve detail or because they are non-solid [S1, unofficial].

#### Vertex Layout
Vertices carry per-vertex data for shading. From [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct Vertex {
    pos: Vec4<f32>,
    uv: Vec2<f32>,
    normal: Vec3<f32>,
    ao: f32,           // Ambient Occlusion (0.0 = occluded, 1.0 = lit)
    smooth_light: f32, // Smooth lighting value
    block: MyBlock,    // Block type for render pass filtering
    atlas_offset: f32, // Pre-computed atlas tile offset
    two_sided: bool,   // Whether this vertex belongs to a two-sided quad
}
```

### Steps for Mesh Generation with Block Configuration

1. **Iterate Voxel Positions**: For each voxel in the chunk.
2. **Retrieve Block Config**: Get the `BlockConfig` for the voxel's block type.
3. **Determine Shape**: Call `config.shape()` to get the geometric primitive.
4. **Determine Cull Mode**: Call `config.cull_mode()` to get the culling rule.
5. **Evaluate Face Visibility**:
    - For each of the 6 faces of the voxel:
        - Check neighbor occupancy.
        - If `CullMode` is `Opaque`: Cull if neighbor is solid (non-empty).
        - If `CullMode` is `TransparentMerged(T)`: Cull if neighbor is transparent AND has the same `TransparentGroup` ID.
        - If `CullMode` is `TransparentUnmerged`: Do not cull based on neighbor transparency; only cull if neighbor is empty (or never cull, depending on implementation).
        - If `CullMode` is `Empty`: Do not generate any faces.
6. **Generate Geometry**:
    - If the face is visible, call the appropriate geometry builder (see [Geometry Builder Pattern](#minor-variants)) to add vertices and indices to the chunk's buffer.
    - Use `config.light(face)` to set lighting values for the generated vertices.
7. **Finalize Mesh**: Transfer vertex/index data to the GPU mesh object.

### Parameters and Typical Values

| Parameter | Type | Example Values | Description |
|-----------|------|----------------|-------------|
| `Shape` | Enum | `WholeBlock`, `Cross`, `Ramp` | Defines the mesh topology. |
| `CullMode` | Enum | `Opaque`, `TransparentMerged`, `TransparentUnmerged` | Defines transparency culling behavior. |
| `Light` | Float | `0.0` to `15.0` | Lighting level per face. |
| `TransparentGroup` | ID | `u8` or `int` | Group ID for merging transparent blocks. |

### Edge Cases and Pitfalls

- **Fluid Shape Assertion**: Setting `FLUID_ENABLED` to true without implementing the fluid shape causes a debug assertion failure rather than silently drawing fluids as cubes [S1, unofficial].
- **Thread Safety**: In Unity3D environments, vertex/index calculation can be done on a background thread, but the final `Mesh` object creation must be moved to the main thread [S3, unofficial].
- **Greedy Meshing Compatibility**: `TransparentMerged` does not automatically handle greedy meshing equality checks; additional logic is required to ensure adjacent identical transparent blocks are merged correctly [S1, unofficial].

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers: 'opaque', 'water', and 'transparentSolid'. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right) [S2, unofficial].
- **Instance Creation**: Iterate visible faces. If face is Left, Right, Top, or Front, set `drawSlope = true` and assign tint to `tints[0]` (Top/Front), `tints[1]` (Left), or `tints[2]` (Right). If a specific tint is `Vector3.One`, fallback to another face's tint. Add instances for slope, left triangle, and right triangle using respective shape IDs and tints. If face is Back or Bottom, add instance using `cubeBackFaceShapeID` or `cubeBottomFaceShapeID` [S2, unofficial].
- **Layout**:
    - Slope vertices (6 vertices): (0,1,0) with UV[0], (0,0,1) with UV[1], (1,1,0) with UV[2], (1,1,0) with UV[3], (0,0,1) with UV[4], (1,0,1) with UV[5].
    - Left triangle vertices (3 vertices): (0,0,1) with UV[0], (0,1,0) with UV[1], (0,0,0) with UV[2].
    - Right triangle vertices (3 vertices): (1,0,0) with UV[0], (1,1,0) with UV[1], (1,0,1) with UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match [S1, unofficial].
- **Parameter**: `FluidInfo::id: u8` - Two adjacent fluid blocks join their surfaces only when their ids match [S1, unofficial].

### Geometry Builder Pattern
A design pattern using an interface to control the type of geometry created for each non-empty voxel, allowing different geometry types (cubes, ramps, X-shapes) to be generated via polymorphism without large switch statements [S3, unofficial].
- **Steps**:
    1. During mesh creation, call `BuildGeometry()` once for each non-empty block [S3, unofficial].
    2. Pass a reference to the chunk object to `BuildGeometry()` [S3, unofficial].
    3. Inside `BuildGeometry()`, use the chunk reference to add vertices and indices to the chunk's linked list [S3, unofficial].
    4. After `BuildGeometry()` has been called on every block for the chunk, place the vertex and index data into a Unity3D Mesh object on the main thread [S3, unofficial].
- **Layout**: Interface `IGeometryBuilder` contains a single function: `BuildGeometry()` [S3, unofficial].
- **Pitfall**: Unity3D functions must be called on the main thread. While vertex/index calculation is done on a background thread, the final Mesh object creation must be moved to the main thread [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(V)$ where $V$ is the number of voxels in the chunk, as each voxel is processed once to determine its shape and culling mode.
- **Memory Complexity**: $O(F)$ where $F$ is the number of visible faces, as geometry is generated only for visible faces.
- **Strengths**: Modular design allows easy addition of new block types and shapes without modifying core meshing logic. Supports complex geometries (ramps, fluids) alongside standard cubes.
- **Limitations**: Requires careful management of transparency groups and culling modes to avoid visual artifacts. Threading constraints in some engines (e.g., Unity3D) require careful handling of mesh finalization.
- **When to Use**: When rendering voxel worlds with diverse block types requiring different geometries and transparency behaviors.
- **When Not to Use**: For simple, uniform voxel worlds where all blocks are identical cubes.

## Background
Originating from implementations in voxel game engines and research projects, this method abstracts the per-voxel rendering configuration. It is commonly found in Rust-based voxel meshers (e.g., `finnbear/voxmesh`) and Unity3D-based voxel generators (e.g., `Cubic-Grid_Voxel_Renderer`, `VoxBox`). The geometry builder pattern is particularly associated with academic projects on procedural voxel world generation [S3, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
