# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
Voxel Block Shape & Culling Configuration is a trait-based system that defines block geometry, transparency culling modes, and lighting per block type to optimize mesh generation. It solves the problem of managing diverse voxel behaviors (opaque, transparent, fluid, complex shapes) by decoupling shape definition from the meshing algorithm. Unlike sibling methods that rely on global boolean flags or simple neighbor checks, this approach assigns specific `CullMode` and `Shape` traits to individual block IDs, allowing for nuanced handling such as merging transparent faces or generating non-cubic geometry [S1, S3, unofficial].

## How it works
**Inputs:** Block type identifier (e.g., `MyBlock` enum), neighbor occupancy data, and configuration constants (e.g., `FLUID_ENABLED`).
**Outputs:** Vertex and index data for visible faces, categorized by culling mode and shape type.

**Data Structures:**
1.  **Block Trait Interface:** A trait (or interface) defining three key methods for each block type:
    *   `shape() -> Shape`: Returns the geometric primitive (e.g., `WholeBlock`, `Cross`, `Ramp`).
    *   `cull_mode() -> CullMode`: Returns the culling strategy (e.g., `Opaque`, `TransparentMerged`, `TransparentUnmerged`, `Empty`).
    *   `light() -> u8` (or `f32`): Returns the lighting value for the block/face.
2.  **Shape Enum:** Defines the geometry. Common variants include:
    *   `WholeBlock`: Standard 6-face cube.
    *   `Cross`: A cross-shaped geometry (e.g., for chains), defined by an `AlignedFace` and `stretch` value.
    *   `Ramp`: Sloped geometry composed of a slope quad and side triangles.
3.  **CullMode Enum:**
    *   `Empty`: No faces generated (e.g., Air).
    *   `Opaque`: Standard face culling; shared faces with solid neighbors are culled.
    *   `TransparentMerged(T)`: Culls shared faces between two transparent blocks if their `TransparentGroup` values are equal. Does not replace `PartialEq` for greedy meshing.
    *   `TransparentUnmerged`: Transparent faces are never merged with neighbors, even if identical.
4.  **Vertex Layout:**
    *   `pos`: `Vec4<f32>` (position).
    *   `uv`: `Vec2<f32>` (texture coordinates).
    *   `normal`: `Vec3<f32>` (face normal).
    *   `ao`: `f32` (ambient occlusion, 0.0=occluded, 1.0=lit).
    *   `smooth_light`: `f32` (smooth lighting value).
    *   `block`: Block ID (for render pass filtering).
    *   `atlas_offset`: `f32` (pre-computed atlas tile offset).
    *   `two_sided`: `bool` (whether the vertex belongs to a two-sided quad).

**Steps:**
1.  **Initialize Configuration:** Define the block types and their associated traits. For example:
    *   `Air`: `CullMode::Empty`.
    *   `Cobblestone`: `CullMode::Opaque`, `Shape::WholeBlock`.
    *   `Glass`: `CullMode::TransparentMerged(())`.
    *   `Leaves`: `CullMode::TransparentUnmerged`.
    *   `ChainZ`: `Shape::Cross(CrossInfo { face: AlignedFace::PosZ, stretch: 0 })`.
2.  **Iterate Voxel Chunk:** For each voxel in the chunk:
    a.  Retrieve the block type.
    b.  Call `cull_mode()` to determine if faces should be generated.
    c.  If `Empty`, skip.
    d.  If `Opaque` or `TransparentUnmerged`, perform standard neighbor occupancy checks. If a neighbor is solid (or matches the culling criteria for transparent blocks), cull the face.
    e.  If `TransparentMerged(T)`, check if the neighbor is transparent and has the same `TransparentGroup` value. If so, cull the shared face.
3.  **Generate Geometry:**
    a.  Call `shape()` to determine the primitive.
    b.  For `WholeBlock`, generate standard cube faces.
    c.  For `Cross`, generate the specific cross geometry based on the `AlignedFace`.
    d.  For `Ramp` (if implemented via this system), iterate visible faces. If the face is Left, Right, Top, or Front, set `drawSlope = true`. Assign tints: `tints[0]` for Top/Front, `tints[1]` for Left, `tints[2]` for Right. If a tint is `Vector3.One`, fallback to another face's tint. Add instances for slope, left triangle, and right triangle. If Back or Bottom, use specific back/bottom face shape IDs.
4.  **Apply Lighting:** For each generated face, call `light()` to get the lighting value. For specific blocks like `Cactus`, the lighting value may depend on the face orientation (e.g., `PosY` -> 11.0, `NegY` -> 12.0, else 10.0).
5.  **Build Mesh:** Add vertices and indices to the chunk's data structure.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Trait definition
class BlockTrait:
    def shape(self) -> Shape:
        raise NotImplementedError
    def cull_mode(self) -> CullMode:
        raise NotImplementedError
    def light(self, face: Face) -> float:
        raise NotImplementedError

# CullMode logic
def should_cull_face(block: BlockTrait, neighbor: BlockTrait, face_normal: Vec3) -> bool:
    mode = block.cull_mode()
    if mode == CullMode.Empty:
        return True
    if mode == CullMode.Opaque:
        # Standard culling: if neighbor is solid, cull
        return neighbor.cull_mode() != CullMode.Empty
    if mode == CullMode.TransparentMerged:
        # Cull if neighbor is transparent and has same group
        # Note: T is a type parameter for the group ID
        if neighbor.cull_mode() == CullMode.TransparentMerged:
            return block.get_transparent_group() == neighbor.get_transparent_group()
        return False
    if mode == CullMode.TransparentUnmerged:
        # Never cull transparent faces
        return False
    return False

# Mesh generation step
def generate_block_mesh(block: BlockTrait, position: Vec3, neighbors: Dict[Vec3, BlockTrait]):
    shape = block.shape()
    cull_mode = block.cull_mode()
    
    if cull_mode == CullMode.Empty:
        return []
    
    faces = []
    for face_normal in [UP, DOWN, LEFT, RIGHT, FRONT, BACK]:
        neighbor_pos = position + face_normal
        neighbor = neighbors.get(neighbor_pos)
        
        # Handle boundary as empty/air
        if neighbor is None:
            neighbor_trait = BlockTraitAir() # Assume Air trait
        
        if not should_cull_face(block, neighbor_trait, face_normal):
            # Generate geometry based on shape
            if shape == Shape.WholeBlock:
                faces.extend(generate_cube_face(block, position, face_normal))
            elif shape == Shape.Cross:
                faces.extend(generate_cross_face(block, position, shape.cross_info))
            elif shape == Shape.Ramp:
                faces.extend(generate_ramp_face(block, position, face_normal))
            
            # Apply lighting
            light_val = block.light(face_normal)
            for vertex in faces:
                vertex.smooth_light = light_val
                
    return faces
```

**Parameters:**
*   `FLUID_ENABLED`: Boolean flag. If `true`, fluid shapes are enabled. **Pitfall:** Setting this to `true` without implementing fluid shape causes a debug assertion failure rather than silently drawing fluids as cubes [S1, unofficial].
*   `TransparentGroup`: A value associated with `TransparentMerged` blocks to determine merge eligibility.

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers ('opaque', 'water', 'transparentSolid') to determine rendering material and buffer assignment. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### [VoxelBlockyTypeLibrary with Attributes](12820-voxelblockytypelibrary-with-attributes.md)
`VoxelBlockyTypeLibrary` is a model management system for blocky voxel terrains that organizes meshes by logical "block types" with configurable state attributes, rather than storing individual model references directly. It solves the problem of managing complex block behaviors (e.g., orientation, powered state, connections) by automatically generating unique numerical IDs for every combination of attribute values, similar to Minecraft's block state system. It differs from the standard `VoxelBlockyLibrary` by decoupling model identity from manual ID assignment, using a combinatorial attribute system instead.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right) [S2, unofficial]. Instance creation involves iterating visible faces. If the face is Left, Right, Top, or Front, set `drawSlope = true` and assign tint to `tints[0]` (Top/Front), `tints[1]` (Left), or `tints[2]` (Right). If a specific tint is `Vector3.One`, fallback to another face's tint. Add instances for slope, left triangle, and right triangle using respective shape IDs and tints. If the face is Back or Bottom, add instance using `cubeBackFaceShapeID` or `cubeBottomFaceShapeID` [S2, unofficial]. Slope vertices (6 vertices): (0,1,0) with UV[0], (0,0,1) with UV[1], (1,1,0) with UV[2], (1,1,0) with UV[3], (0,0,1) with UV[4], (1,0,1) with UV[5]. Left triangle vertices (3 vertices): (0,0,1) with UV[0], (0,1,0) with UV[1], (0,0,0) with UV[2]. Right triangle vertices (3 vertices): (1,0,0) with UV[0], (1,1,0) with UV[1], (1,0,1) with UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match [S1, unofficial]. Two adjacent fluid blocks join their surfaces only when their `FluidInfo::id` (u8) values match [S1, unofficial].

### Geometry Builder Pattern
A design pattern using an interface to control the type of geometry created for each non-empty voxel, allowing different geometry types (cubes, ramps, X-shapes) to be generated via polymorphism without large switch statements [S3, unofficial]. During mesh creation, `BuildGeometry()` is called once for each non-empty block. A reference to the chunk object is passed to `BuildGeometry()`, which uses this reference to add vertices and indices to the chunk's linked list. After `BuildGeometry()` has been called on every block for the chunk, the vertex and index data is placed into a Unity3D Mesh object on the main thread [S3, unofficial]. The interface `IGeometryBuilder` contains a single function: `BuildGeometry()` [S3, unofficial]. **Pitfall:** Unity3D functions must be called on the main thread. While vertex/index calculation is done on a background thread, the final Mesh object creation must be moved to the main thread [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is processed once to determine its faces.
*   **Memory Complexity:** $O(F)$ where $F$ is the number of visible faces. The trait system adds minimal overhead per block type definition.
*   **Strengths:** Highly flexible; supports complex shapes (ramps, crosses) and nuanced transparency (merged vs. unmerged) without modifying the core meshing loop. Decouples block behavior from geometry generation.
*   **Limitations:** Requires careful implementation of all trait methods to avoid assertion failures (e.g., missing fluid shapes). The trait dispatch may add slight overhead compared to hardcoded switch statements, though this is negligible in most cases.
*   **When to use:** When the voxel world contains diverse block types with different rendering requirements (e.g., fluids, transparent materials, non-cubic shapes).
*   **When not to use:** For simple, uniform voxel worlds where all blocks are opaque cubes, as the overhead of the trait system is unnecessary.

## Background
The concept of trait-based configuration for voxel mesh generation originated in community-driven voxel engine projects, such as `finnbear/voxmesh` [S1, unofficial]. The Geometry Builder pattern was formalized in academic work on procedural voxel generation, specifically in the thesis "Procedural Generation of Voxel Worlds with Castles" [S3, unofficial], to manage complex geometries like ramps and micro-blocks. The Ramp Shape Geometry and Fluid Surface Stitching techniques are documented in community repositories like `rarefushion/Cubic-Grid_Voxel_Renderer` [S2, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
