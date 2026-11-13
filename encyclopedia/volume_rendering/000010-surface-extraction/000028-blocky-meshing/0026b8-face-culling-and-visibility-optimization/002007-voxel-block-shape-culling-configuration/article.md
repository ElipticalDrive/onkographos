# Voxel Block Shape & Culling Configuration

Also called: Block trait interface, Shape enum system

## Overview
Voxel Block Shape & Culling Configuration is a trait-based system that defines the geometry, transparency culling behavior, and lighting properties for individual block types in a voxel mesh generator [S1, unofficial]. It solves the problem of hard-coded geometry and culling logic by decoupling block identity from mesh generation, allowing different block types (e.g., opaque stone, transparent glass, sloped ramps) to specify their own shape and visibility rules [S1, unofficial]. Unlike generic meshing algorithms that treat all voxels as cubes, this configuration allows per-block-type customization of face generation and merging, which is essential for supporting complex block variants like fluids, slopes, and non-colliding structures [S1, unofficial][S3, unofficial].

## How it works
The system operates by associating a configuration object (often a trait or interface implementation) with each block type. This configuration dictates how the block contributes to the final mesh.

### Inputs and Outputs
*   **Input:** A block type identifier (e.g., `MyBlock::Glass`) and the current voxel state (occupied/empty, neighbor occupancy).
*   **Output:** A set of vertices and indices representing the visible faces of the block, along with metadata such as lighting values and atlas offsets.

### Data Structures
The core data structure is a trait or interface that exposes methods for shape, culling, and lighting.

**Trait Definition (Conceptual):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
trait BlockConfig {
    fn shape(&self) -> Shape;
    fn cull_mode(&self) -> CullMode;
    fn light(&self, face: Face) -> f32;
}
```

**Shape Enum:**
Represents the geometric primitive of the block.
*   `WholeBlock`: Standard 6-face cube.
*   `Cross(CrossInfo)`: A cross-shaped geometry (e.g., for chains or fences) defined by an aligned face and stretch parameter.
*   `Ramp`: A sloped block composed of a slope quad and side triangles.
*   `Fluid`: A block that only generates top faces and stitches with neighbors.

**CullMode Enum:**
Determines how faces are culled based on neighbor occupancy and transparency.
*   `Empty`: No faces generated (e.g., Air).
*   `Opaque`: Standard neighbor culling; faces are generated if the neighbor is empty.
*   `TransparentUnmerged`: Faces are generated if the neighbor is empty or a different block type, even if both are transparent. Shared faces between identical transparent blocks are kept.
*   `TransparentMerged(T)`: Faces are generated if the neighbor is empty or a different block type. If the neighbor is a transparent block with the same `TransparentGroup` value `T`, the shared face is culled.

**Vertex Layout:**
Vertices include position, UV, normal, ambient occlusion (AO), smooth light, block ID, atlas offset, and a two-sided flag.
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct Vertex {
    pos: Vec4<f32>,
    uv: Vec2<f32>,
    normal: Vec3<f32>,
    ao: f32,       // 0.0 = fully occluded, 1.0 = fully lit
    smooth_light: f32,
    block: MyBlock,
    atlas_offset: f32,
    two_sided: bool,
}
```

### Steps
1.  **Configuration Lookup:** For each non-empty voxel at position $P$, retrieve its block type $B$ and its configuration $C = \text{Config}(B)$.
2.  **Shape Determination:** Call $C.\text{shape}()$ to determine the geometric primitive.
    *   If `WholeBlock`, generate up to 6 faces.
    *   If `Cross`, generate the specific cross geometry.
    *   If `Ramp`, generate the slope quad and side triangles.
    *   If `Fluid`, generate only the top face and check for stitching with neighbors.
3.  **Culling Decision:** For each potential face with normal $N$:
    *   Identify the neighbor voxel at $P_{neighbor} = P + N$.
    *   If $P_{neighbor}$ is empty or out of bounds, the face is potentially visible.
    *   If $P_{neighbor}$ is occupied by block $B_{neighbor}$:
        *   Retrieve $C_{neighbor} = \text{Config}(B_{neighbor})$.
        *   If $C.\text{cull_mode}()$ is `Opaque`, cull the face.
        *   If $C.\text{cull_mode}()$ is `TransparentUnmerged`, keep the face.
        *   If $C.\text{cull_mode}()$ is `TransparentMerged(T)`:
            *   If $C_{neighbor}.\text{cull_mode}()$ is `TransparentMerged(T)` and $T$ matches, cull the face.
            *   Otherwise, keep the face.
4.  **Geometry Generation:**
    *   For `WholeBlock`: Generate 4 vertices per face.
    *   For `Ramp`: Generate 6 vertices for the slope, 3 for the left triangle, and 3 for the right triangle.
    *   For `Fluid`: Generate 4 vertices for the top face. If adjacent fluid blocks have matching IDs, stitch the surface by sharing vertices.
5.  **Lighting Calculation:** Call $C.\text{light}(face)$ to determine the lighting value for the face. This value is interpolated across the vertices.
6.  **Vertex Assembly:** Create vertex structures with position, UV, normal, AO, smooth light, block ID, and atlas offset.
7.  **Index Generation:** Create index buffers for the generated triangles.
8.  **Mesh Finalization:** Transfer vertex and index data to the main thread (if using multithreading) and create the final mesh object.

### Formulas
*   **Neighbor Position:** $P_{neighbor} = P + N$, where $P$ is the voxel coordinate and $N$ is the face normal vector.
*   **Plane Distance for Frustum Culling:** For a view-projection matrix $VP$, the plane coefficients $(A, B, C, D)$ are derived from the matrix columns. For example, the left plane is derived from $Col4 + Col1$ [S2, unofficial].
*   **Lighting Interpolation:** Lighting values are interpolated across the vertices of a face based on the `light()` return value for each face.

### Parameters
*   **TransparentGroup (T):** A value used in `TransparentMerged` to group transparent blocks that should merge their shared faces.
*   **Fluid ID:** A unique identifier for fluid blocks. Adjacent fluid blocks with the same ID stitch their surfaces [S1, unofficial].
*   **FLUID_ENABLED:** A boolean flag. If set to true without implementing fluid shape, it causes a debug assertion failure [S1, unofficial].

### Edge Cases and Pitfalls
*   **Fluid Shape Implementation:** Setting `FLUID_ENABLED` to true without implementing the fluid shape logic causes a debug assertion failure rather than silently drawing fluids as cubes [S1, unofficial].
*   **Threading Constraints:** In environments like Unity3D, mesh creation functions must be called on the main thread, even if vertex/index calculation is done on a background thread [S3, unofficial].
*   **Tint Fallback:** For ramp shapes, if a specific tint is `Vector3.One`, it should fallback to another face's tint to avoid visual artifacts [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh_for_chunk(chunk):
    vertices = []
    indices = []
    
    for x, y, z in chunk.voxels:
        block_type = chunk.get_block(x, y, z)
        if block_type == AIR:
            continue
            
        config = get_block_config(block_type)
        shape = config.shape()
        cull_mode = config.cull_mode()
        
        # Determine faces to generate
        faces_to_generate = []
        for face_normal in [UP, DOWN, NORTH, SOUTH, EAST, WEST]:
            neighbor_pos = (x + face_normal.x, y + face_normal.y, z + face_normal.z)
            neighbor_block = chunk.get_block(*neighbor_pos)
            
            if neighbor_block == AIR:
                faces_to_generate.append(face_normal)
                continue
                
            neighbor_config = get_block_config(neighbor_block)
            neighbor_cull_mode = neighbor_config.cull_mode()
            
            if cull_mode == OPAQUE:
                continue
            elif cull_mode == TRANSPARENT_UNMERGED:
                faces_to_generate.append(face_normal)
            elif cull_mode == TRANSPARENT_MERGED:
                if neighbor_cull_mode == TRANSPARENT_MERGED and neighbor_config.transparent_group == config.transparent_group:
                    continue
                else:
                    faces_to_generate.append(face_normal)
        
        # Generate geometry for each face
        for face in faces_to_generate:
            if shape == WHOLE_BLOCK:
                verts, idxs = generate_cube_face(face, config)
            elif shape == RAMP:
                verts, idxs = generate_ramp_face(face, config)
            elif shape == FLUID:
                verts, idxs = generate_fluid_face(face, config, chunk)
            else:
                continue
                
            # Apply lighting and attributes
            light_val = config.light(face)
            for v in verts:
                v.ao = interpolate_ao(v, chunk)
                v.smooth_light = light_val
                v.block = block_type
                v.atlas_offset = get_atlas_offset(face)
                
            vertices.extend(verts)
            indices.extend(idxs)
            
    # Transfer to main thread and create mesh
    create_mesh_on_main_thread(vertices, indices)
```

## Subtopics

### [Three-Valued Opacity Classification for Meshing](4414-three-valued-opacity-classification-for-meshing.md)
A classification system for block IDs into three mutually exclusive layers ('opaque', 'water', 'transparentSolid') to determine rendering material and buffer assignment. It solves the insufficiency of a single boolean 'transparent' flag by distinguishing between fluids (water) requiring bespoke shaders and see-through solids (glass/leaves) requiring alpha blending. Unlike a boolean transparent flag which collapses water and glass/leaves into one category, this three-valued model allows water to use a bespoke shader (ripple, refraction) while glass/leaves use the ordinary block atlas with alpha blending.

### [VoxelBlockyTypeLibrary with Attributes](12820-voxelblockytypelibrary-with-attributes.md)
`VoxelBlockyTypeLibrary` is a model management system for blocky voxel terrains that organizes meshes by logical "block types" with configurable state attributes, rather than storing individual model references directly. It solves the problem of managing complex block behaviors (e.g., orientation, powered state, connections) by automatically generating unique numerical IDs for every combination of attribute values, similar to Minecraft's block state system. It differs from the standard `VoxelBlockyLibrary` by decoupling model identity from manual ID assignment, using a combinatorial attribute system instead.

### Ramp Shape Geometry
A shape representing a sloped block face, composed of a slope quad (2 triangles) and two side triangles (left and right) [S2, unofficial]. Instance creation involves iterating visible faces. If the face is Left, Right, Top, or Front, `drawSlope` is set to true and tint is assigned to `tints[0]` (Top/Front), `tints[1]` (Left), or `tints[2]` (Right). If a specific tint is `Vector3.One`, it falls back to another face's tint. Instances are added for slope, left triangle, and right triangle using respective shape IDs and tints. If the face is Back or Bottom, an instance is added using `cubeBackFaceShapeID` or `cubeBottomFaceShapeID` [S2, unofficial]. The slope vertices are: (0,1,0) with UV[0], (0,0,1) with UV[1], (1,1,0) with UV[2], (1,1,0) with UV[3], (0,0,1) with UV[4], (1,0,1) with UV[5]. The left triangle vertices are: (0,0,1) with UV[0], (0,1,0) with UV[1], (0,0,0) with UV[2]. The right triangle vertices are: (1,0,0) with UV[0], (1,1,0) with UV[1], (1,0,1) with UV[2] [S2, unofficial].

### Fluid Surface Stitching
A mesh generation technique where fluid columns share a surface height field with neighbors, stitching surfaces when fluid IDs match [S1, unofficial]. Two adjacent fluid blocks join their surfaces only when their `FluidInfo::id` (u8) values match [S1, unofficial].

### Geometry Builder Pattern
A design pattern using an interface to control the type of geometry created for each non-empty voxel, allowing different geometry types (cubes, ramps, X-shapes) to be generated via polymorphism without large switch statements [S3, unofficial]. During mesh creation, `BuildGeometry()` is called once for each non-empty block, passing a reference to the chunk object. Inside `BuildGeometry()`, vertices and indices are added to the chunk's linked list. After `BuildGeometry()` has been called on every block for the chunk, the vertex and index data is placed into a Unity3D Mesh object on the main thread [S3, unofficial]. The interface `IGeometryBuilder` contains a single function: `BuildGeometry()` [S3, unofficial]. A pitfall is that Unity3D functions must be called on the main thread, so while vertex/index calculation is done on a background thread, the final Mesh object creation must be moved to the main thread [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is processed once. The complexity of face generation depends on the shape, but is generally constant per voxel.
*   **Memory Complexity:** $O(F)$ where $F$ is the number of visible faces. The memory usage is proportional to the surface area of the generated mesh.
*   **Performance:** The trait-based approach adds a small overhead for configuration lookup but allows for significant performance gains through efficient culling and geometry reduction.
*   **Strengths:** Flexibility in supporting diverse block types, efficient culling of internal and hidden faces, and separation of concerns between block logic and mesh generation.
*   **Limitations:** Requires careful implementation of culling modes to avoid visual artifacts. Threading constraints in some environments (e.g., Unity3D) can complicate implementation.
*   **When to Use:** When generating voxel meshes with diverse block types that require different geometries and culling behaviors.
*   **When Not to Use:** For simple voxel worlds where all blocks are identical cubes and standard neighbor culling is sufficient.

## Background
The concept of trait-based configuration for voxel mesh generation originated in the need to support complex block types in voxel games like Minecraft [S3, unofficial]. The Geometry Builder pattern was introduced to manage the variety of block geometries without resorting to large switch statements [S3, unofficial]. The Three-Valued Opacity Classification was developed to handle the distinct rendering requirements of fluids and transparent solids [S1, unofficial]. The Ramp Shape Geometry and Fluid Surface Stitching techniques were implemented to enhance visual fidelity and realism in voxel terrains [S2, unofficial][S1, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
