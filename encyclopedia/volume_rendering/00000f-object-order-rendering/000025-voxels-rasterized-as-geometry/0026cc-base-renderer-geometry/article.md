# Base Renderer Geometry

Also called: Cube Indices Layout, DrawVoxel, Square Face Indices Layout, Square Line Indices Layout, VoxelPass

## Overview
Base Renderer Geometry is a method for rendering voxel data by converting occupied voxel indices into explicit geometric primitives (cubes, squares, or lines) and submitting them to the graphics pipeline for rasterization. It solves the problem of visualizing scalar field data by treating voxels as discrete 3D objects rather than volume textures. It differs from [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md) by generating geometry in CPU-side or vertex-shader-side buffers rather than computing ray-marching in the fragment shader, and from [Splatting and Point-based Rendering](4975-splatting-and-point-based-rendering.md) by using explicit polygonal faces instead of billboards.

## How it works
The method constructs a rendering pipeline that processes voxel meshes, applies transformations, and issues draw calls for static and skinned batches.

### Inputs and Outputs
**Inputs:**
- Voxel data structure (e.g., `NestedVoxel<T>`).
- Texture atlas handle (`Handle<Atlas>`).
- Transform matrix (`Mat4x4`).
- Optional skinning data (`JointTransforms`).
- Material properties (Albedo, Emission, Metallic/Roughness).
- Rendering parameters: `transparency` (bool), `Orientation` (XY/XZ), `CullMode`.

**Outputs:**
- Rasterized image on the screen.
- Updated depth buffer.

### Data Structures

**Index Layouts**
The geometry is defined by index arrays. The sources define three specific layouts [S1, unofficial]:

1.  **Cube Indices (`CUBE_INDICES`)**: Defines the edges of a cube.
    -   Type: `int` array.
    -   Size: 24 elements.
    -   Content: `[0, 1, 1, 2, 2, 3, 3, 0, 4, 5, 5, 6, 6, 7, 7, 4, 0, 4, 1, 5, 2, 6, 3, 7]`.
    -   Usage: Renders wireframe cubes or line strips.

2.  **Square Face Indices (`SQUARE_FACE_INDICES`)**: Defines a single square face using two triangles.
    -   Type: `int` array.
    -   Size: 6 elements.
    -   Content: `[0, 1, 2, 0, 2, 3]`.
    -   Usage: Renders solid faces.

3.  **Square Line Indices (`SQUARE_LINE_INDICES`)**: Defines the perimeter of a square.
    -   Type: `int` array.
    -   Size: 8 elements.
    -   Content: `[0, 1, 1, 2, 2, 3, 3, 0]`.
    -   Usage: Renders wireframe squares.

**Vertex Attributes**
The base vertex format comprises [S2, unofficial]:
-   `Position`: 3-component float (or 4-component `Vector4` in some implementations [S1, unofficial]).
-   `Normal`: 3-component float.
-   `Tangent`: 3-component float.
-   `Surface`: Packed texture coordinates and ambient occlusion into a single 3-component float array, formatted as RGB 32-bit float.

The skinned format extends this with `JointCombined`.

**Mesh and Batch Structures**
-   `DynamicVoxelMesh<T>`: Holds the voxel data, atlas handle, transform, parent entity info, and a `dirty` flag [S2, unofficial].
-   `VoxelMesh`: Holds the generated `ModelData` (geometry) and atlas handle [S2, unofficial].
-   `DrawVoxel` struct: Manages the rendering state [S2, unofficial].
    -   `pipeline_basic`: Graphics pipeline for opaque/static voxels.
    -   `pipeline_skinned`: Graphics pipeline for skinned voxels.
    -   `static_batches`: `TwoLevelBatch<MaterialId, u32, SmallVec<[VertexArgs; 4]>>`.
    -   `skinned_batches`: `TwoLevelBatch<MaterialId, u32, SmallVec<[SkinnedVertexArgs; 4]>>`.
    -   `vertex_format_base`, `vertex_format_skinned`: Vertex input layouts.
    -   `env`: Environment descriptor set.
    -   `materials`: Material descriptor sets.
    -   `models`: `DynamicVertexBuffer` for static meshes.
    -   `skinned_models`: `DynamicVertexBuffer` for skinned meshes.
    -   `transparency`: Boolean flag controlling the pass.

### Pipeline Construction
1.  **Descriptor Set Layouts**: Create layouts for vertex input, environment, and material data [S2, unofficial].
2.  **Pipeline Creation**: Build a basic graphics pipeline with the following state [S2, unofficial]:
    -   Back-face culling enabled.
    -   Depth test function: `Less`.
    -   Depth write: `!transparent` (disable depth write for transparent passes).
    -   Blending: Optional premultiplied alpha blending if `transparency` is true.
3.  **Skinned Pipeline**: If `skinning` is enabled, build a child pipeline with a different vertex descriptor and skinned vertex shader [S2, unofficial].

### Preparation Phase
1.  **Fetch Components**: Retrieve `VoxelMesh`, `Atlas`, `Handle<VoxelMesh>`, `Transform`, `JointTransforms`, and `Tint` from the world entities [S2, unofficial].
2.  **Filter Meshes**: Exclude meshes whose alpha state does not match the current pass configuration (`transparency`) to prevent blending errors [S2, unofficial].
3.  **Group Batches**: Group meshes by Material ID.
4.  **Assign Batches**: Insert meshes into `static_batches` or `skinned_batches` based on the `skinning` flag.
5.  **Write Buffers**: Write batch data to `models` (static) and `skinned_models` (skinned) dynamic vertex buffers [S2, unofficial].

### Drawing Phase
1.  **Bind Pipeline**: Bind the appropriate graphics pipeline (`pipeline_basic` or `pipeline_skinned`) [S2, unofficial].
2.  **Bind Descriptors**: Bind the environment descriptor set and material descriptor sets [S2, unofficial].
3.  **Iterate Batches**:
    -   Iterate over `static_batches` (or `skinned_batches`).
    -   For each material, bind the material descriptor set.
    -   Iterate over mesh batches.
    -   Call `mesh.bind_and_draw` with instance ranges [S2, unofficial].

### Procedural Generation (Alternative/Supplementary Approach)
Some implementations generate geometry procedurally in the rendering loop using OpenGL immediate mode or similar APIs [S1, unofficial].

**Parameters:**
-   `LocalToWorld`: `Matrix4x4` transformation from local to world space. Default: Identity.
-   `Orientation`: Enum `DRAW_ORIENTATION` with values `XY` or `XZ`. Default: `XY`.
-   `DefaultColor`: `Color`. Default: White.
-   `CullMode`: Rendering cull mode. Default: Off.
-   `Size`: Voxel size. If `ScaleOnZoom` is enabled and camera is orthographic, `size *= camera.orthographicSize / 10.0f`.

**Algorithm:**
1.  Push matrix and load identity.
2.  Set ModelView matrix: `camera.worldToCameraMatrix * localToWorld`.
3.  Set Projection matrix: `camera.projectionMatrix`.
4.  Set material pass.
5.  Begin primitive drawing (`GL.QUADS` or `GL.LINES`).
6.  Iterate through `Vertices` list.
7.  For each vertex `(x, y, z)` with `Color`:
    -   Calculate `half = size * 0.5f`.
    -   Depending on `Orientation`:
        -   **XY**: Draw quad at `z` with corners `(x±half, y±half, z)`.
        -   **XZ**: Draw quad at `y` with corners `(x±half, y, z±half)`.
8.  End primitive drawing.
9.  Pop matrix.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class Vertex:
    position: Vector3
    normal: Vector3
    tangent: Vector3
    surface: Vector3 # Packed UV/AO

class DrawVoxel:
    pipeline_basic: Pipeline
    pipeline_skinned: Pipeline
    static_batches: Dict[MaterialId, List[Batch]]
    skinned_batches: Dict[MaterialId, List[Batch]]
    transparency: bool
    skinning: bool

class Batch:
    material_id: MaterialId
    vertex_buffer: DynamicVertexBuffer
    index_count: int
    instance_count: int

def prepare_voxel_rendering(draw_voxel: DrawVoxel, entities: List[Entity]):
    """
    Prepares batches for rendering.
    """
    draw_voxel.static_batches.clear()
    draw_voxel.skinned_batches.clear()
    
    for entity in entities:
        voxel_mesh = entity.get_component(VoxelMesh)
        transform = entity.get_component(Transform)
        is_skinned = entity.has_component(JointTransforms)
        
        # Filter by transparency
        if voxel_mesh.material.alpha != draw_voxel.transparency:
            continue
            
        material_id = voxel_mesh.material.id
        
        # Create batch data
        batch_data = create_batch_data(voxel_mesh, transform)
        
        if is_skinned and draw_voxel.skinning:
            draw_voxel.skinned_batches[material_id].append(batch_data)
        else:
            draw_voxel.static_batches[material_id].append(batch_data)

def render_voxels(draw_voxel: DrawVoxel, camera: Camera):
    """
    Renders the prepared batches.
    """
    if draw_voxel.transparency:
        pipeline = draw_voxel.pipeline_basic # Assuming basic handles transparency via blend state
    else:
        pipeline = draw_voxel.pipeline_basic
        
    # Bind pipeline
    bind_pipeline(pipeline)
    
    # Bind environment descriptors
    bind_descriptor_set(draw_voxel.env)
    
    # Render Static Batches
    for material_id, batches in draw_voxel.static_batches.items():
        material_desc = draw_voxel.materials[material_id]
        bind_descriptor_set(material_desc)
        
        for batch in batches:
            bind_vertex_buffer(batch.vertex_buffer)
            draw_voxel.draw_call(batch.index_count, batch.instance_count)

    # Render Skinned Batches if enabled
    if draw_voxel.skinning:
        bind_pipeline(draw_voxel.pipeline_skinned)
        for material_id, batches in draw_voxel.skinned_batches.items():
            material_desc = draw_voxel.materials[material_id]
            bind_descriptor_set(material_desc)
            
            for batch in batches:
                bind_vertex_buffer(batch.vertex_buffer)
                draw_voxel.draw_call(batch.index_count, batch.instance_count)

def generate_cube_geometry(size: float, orientation: str, vertices: List[Vector3], colors: List[Color]):
    """
    Procedural generation of voxel geometry (OpenGL-style).
    """
    half = size * 0.5
    for i, vertex in enumerate(vertices):
        x, y, z = vertex
        color = colors[i]
        
        gl_color(color)
        
        if orientation == "XY":
            # Draw quad in XY plane at z
            gl_vertex(x + half, y + half, z)
            gl_vertex(x + half, y - half, z)
            gl_vertex(x - half, y - half, z)
            gl_vertex(x - half, y + half, z)
        elif orientation == "XZ":
            # Draw quad in XZ plane at y
            gl_vertex(x + half, y, z + half)
            gl_vertex(x + half, y, z - half)
            gl_vertex(x - half, y, z - half)
            gl_vertex(x - half, y, z + half)
```

### Edge Cases and Pitfalls
-   **Alpha Mismatch**: Meshes are excluded if their alpha state does not match the pass configuration (`transparency`). This prevents incorrect blending or culling but requires careful setup of material alpha values [S2, unofficial].
-   **Precision**: Floating point precision in vertex positions may cause z-fighting if voxel size is very small or camera is far.
-   **Culling**: Back-face culling is calculated by the dot product of the view-space normal and the view vector. If culling is disabled (`CullMode: Off`), performance may degrade due to overdraw [S1, unofficial].

## Subtopics

### [Binary Volume Rendering (Opacity Thresholding)](1015-binary-volume-rendering-opacity-thresholding.md)
Uses a threshold to determine voxel visibility, often used in conjunction with geometry generation to reduce primitive count.

### [Naive Voxel Rendering](1103-naive-voxel-rendering.md)
Renders every voxel as a full cube, regardless of occlusion, leading to high overdraw.

### [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md)
Uses GPU instancing to render many voxels efficiently, often generating geometry in the vertex shader.

### [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md)
Optimizes rendering by only processing occupied voxels in a sparse data structure.

## Operating characteristics
-   **Time Complexity**: $O(V)$ where $V$ is the number of visible voxels. Preparation is $O(V)$ for batching, and rendering is $O(B)$ where $B$ is the number of batches (draw calls).
-   **Memory Complexity**: $O(V)$ for storing vertex and index buffers.
-   **Performance**: Performance is heavily dependent on the number of draw calls. Batching by material reduces draw calls. Skinned rendering adds overhead for joint transformations.
-   **Strengths**: Simple to implement, supports standard PBR materials, easy to integrate with existing rendering pipelines.
-   **Limitations**: High overdraw if not optimized (e.g., face culling). Not suitable for very large dense volumes without sparse optimization.
-   **When to use**: Small to medium voxel scenes, debugging, or when PBR materials are required.
-   **When not to use**: Very large dense volumes, real-time applications with strict performance constraints on draw calls.

## Background
The method is derived from implementations in voxel rendering engines, specifically those using the Amethyst game engine framework and custom voxel libraries. The `DrawVoxel` struct and batching logic are from [S2, unofficial] (Kurble/amethyst_voxel), while the procedural generation and index layouts are from [S1, unofficial] (Scrawk/Marching-Cubes).

## Sources
[S1] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] Kurble/amethyst_voxel, https://github.com/Kurble/amethyst_voxel, fetched 2026-10-27 (unofficial)
