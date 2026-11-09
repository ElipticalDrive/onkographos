# RealityKit Voxel Mesh Generation

Also called: MeshDescriptor voxel mesh, Per-voxel opacity mesh

## Overview
Generates a single `MeshResource` from an array of voxel positions and colors to render ~1M voxels in real time, avoiding the per-entity overhead of instantiating individual `ModelEntity` objects [S1, unofficial]. It builds on [Voxels Grid to Mesh Conversion (Uncolored/Expanded)](8733-voxels-grid-to-mesh-conversion-uncolored-expanded.md) by packing the expanded cube geometry into a single draw call via `MeshDescriptor`.

## How it works
**Inputs:**
- `voxels`: Array of `Voxel` structs.
- `voxelSize`: Physical size of each voxel cube along each axis (default `1.0`) [S1, unofficial].

**Outputs:**
- `MeshResource`: A single mesh resource containing all voxel geometry.

**Data Structures:**
- `Voxel`:
  - `position: SIMD3<Float>`: World position of the voxel center.
  - `color: SIMD4<Float>`: RGBA color/opacity.
- Mesh Buffers (contiguous Swift arrays converted to `MeshBuffer`):
  - `positions: [SIMD3<Float>]`: Vertex positions.
  - `normals: [SIMD3<Float>]`: Vertex normals.
  - `textureCoordinates: [SIMD2<Float>]`: UV coordinates.
  - `primitives: .triangles([UInt32])`: Index buffer.

**Steps:**
1. Initialize empty arrays for `positions`, `normals`, `uvs`, `colors`, and `indices`.
2. Iterate through each `voxel` in `voxels`:
   a. Generate local cube vertices relative to `(0,0,0)` with size `voxelSize`.
      - `halfSize = voxelSize / 2`
      - Vertices (8 total):
        - Front: `[-halfSize, -halfSize, halfSize]`, `[halfSize, -halfSize, halfSize]`, `[halfSize, halfSize, halfSize]`, `[-halfSize, halfSize, halfSize]`
        - Back: `[-halfSize, -halfSize, -halfSize]`, `[halfSize, -halfSize, -halfSize]`, `[halfSize, halfSize, -halfSize]`, `[-halfSize, halfSize, -halfSize]`
   b. Translate vertices by `voxel.position`.
   c. Append translated vertices to `positions`.
   d. Append normals. For a unit cube aligned with axes, normals are constant per face:
      - Front (Z+): `[0, 0, 1]`
      - Back (Z-): `[0, 0, -1]`
      - Right (X+): `[1, 0, 0]`
      - Left (X-): `[-1, 0, 0]`
      - Top (Y+): `[0, 1, 0]`
      - Bottom (Y-): `[0, -1, 0]`
      - Each vertex gets the normal of its face.
   e. Append UVs. Standard mapping:
      - Front/Back faces use `[0,0], [1,0], [1,1], [0,1]` for their respective 4 vertices.
      - Side faces also use `[0,0], [1,0], [1,1], [0,1]` mapped to their 4 vertices.
   f. Append `voxel.color` repeated 8 times to `colors`.
   g. Append indices `[0,1,2, 2,3,0, 1,5,6, 6,2,1, 5,4,7, 7,6,5, 4,0,3, 3,7,4, 3,2,6, 6,7,3, 0,4,5, 5,1,0]` offset by `vertexOffset` to `indices`.
   h. Increment `vertexOffset` by 8.
3. Create `MeshDescriptor(name: "voxels")`.
4. Assign buffers:
   - `descriptor.positions = MeshBuffer(positions)`
   - `descriptor.normals = MeshBuffer(normals)`
   - `descriptor.textureCoordinates = MeshBuffer(uvs)`
   - `descriptor.primitives = .triangles(indices)`
   - `descriptor.materials = .allFaces(0)`
5. Create `MeshResource` from descriptor.
6. Apply `UnlitMaterial` with `.transparent` blending for opacity.

**Pitfalls:**
- Updating per-voxel opacity requires regenerating the entire `MeshResource` and reassigning it to `entity.model?.mesh` [S1, unofficial].
- The example code comments out `descriptor.colors = MeshBuffer(colors)`, relying instead on `UnlitMaterial` blending with a global opacity, which prevents per-voxel alpha updates from taking effect if colors are not used [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import simd

def make_cube(at position: simd_float3, size: float) -> tuple[list[simd_float3], list[simd_float3], list[int]]:
    half_size = size / 2.0
    # Local vertices relative to center
    local_verts = [
        # Front face (Z+)
        simd_float3(-half_size, -half_size, half_size),
        simd_float3(half_size, -half_size, half_size),
        simd_float3(half_size, half_size, half_size),
        simd_float3(-half_size, half_size, half_size),
        # Back face (Z-)
        simd_float3(-half_size, -half_size, -half_size),
        simd_float3(half_size, -half_size, -half_size),
        simd_float3(half_size, half_size, -half_size),
        simd_float3(-half_size, half_size, -half_size),
    ]
    # Translate to world position
    vertices = [v + position for v in local_verts]
    
    # Normals per face
    normals = [
        # Front
        simd_float3(0, 0, 1), simd_float3(0, 0, 1), simd_float3(0, 0, 1), simd_float3(0, 0, 1),
        # Back
        simd_float3(0, 0, -1), simd_float3(0, 0, -1), simd_float3(0, 0, -1), simd_float3(0, 0, -1),
    ]
    
    # Indices for 6 faces, 2 triangles each
    indices = [
        0, 1, 2, 2, 3, 0,  # Front
        1, 5, 6, 6, 2, 1,  # Right
        5, 4, 7, 7, 6, 5,  # Back
        4, 0, 3, 3, 7, 4,  # Left
        3, 2, 6, 6, 7, 3,  # Top
        0, 4, 5, 5, 1, 0   # Bottom
    ]
    return vertices, normals, indices

def generate_voxel_mesh(voxels: list[dict], voxel_size: float = 1.0) -> dict:
    positions = []
    normals = []
    uvs = []
    colors = []
    indices = []
    vertex_offset = 0
    
    # UVs for a single cube (8 vertices)
    cube_uvs = [
        simd_float2(0, 0), simd_float2(1, 0), simd_float2(1, 1), simd_float2(0, 1), # Front
        simd_float2(0, 0), simd_float2(1, 0), simd_float2(1, 1), simd_float2(0, 1), # Back
        # Note: Side faces also use 0-1 UVs, but order depends on winding. 
        # The source example repeats the same 8 UVs for all 8 vertices, assuming 
        # a standard mapping where each face gets a unit square.
        # For simplicity, we repeat the 8 UVs for all 8 vertices as per source.
    ]
    # Actually, the source says:
    # let cubeUVs: [SIMD2<Float>] = [
    #     [0, 0], [1, 0], [1, 1], [0, 1], // front face
    #     [0, 0], [1, 0], [1, 1], [0, 1]  // back face
    # ]
    # This implies only 8 UVs are defined for the 8 vertices, likely assuming 
    # a specific vertex ordering where front/back are first/last 4.
    # We will follow the source's explicit UV array.
    
    for voxel in voxels:
        verts, norms, inds = make_cube(at=voxel['position'], size=voxel_size)
        positions.extend(verts)
        normals.extend(norms)
        uvs.extend(cube_uvs)
        colors.extend([voxel['color']] * 8)
        indices.extend([i + vertex_offset for i in inds])
        vertex_offset += 8
        
    # Create descriptor
    descriptor = {
        'name': 'voxels',
        'positions': positions,
        'normals': normals,
        'textureCoordinates': uvs,
        'primitives': indices,
        'materials': 'allFaces(0)'
    }
    return descriptor
```

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of voxels, as each voxel generates a constant amount of geometry (8 vertices, 36 indices).
- **Memory Complexity:** O(N) for storing vertex and index data.
- **Performance:** Capable of rendering ~1M voxels in real time on visionOS 2.5 [S1, unofficial].
- **Strengths:** Avoids per-entity overhead of `ModelEntity` instantiation. Single draw call.
- **Limitations:** Updating per-voxel opacity requires full mesh regeneration and reassignment [S1, unofficial]. Per-voxel alpha is not supported if using global material opacity [S1, unofficial].

## Background
Originated from a developer's attempt to render ~1M voxels in RealityKit for visionOS 2.5, moving from individual `ModelEntity` instances to a single `MeshResource` via `MeshDescriptor` [S1, unofficial].

## Sources
[S1] RealityKit voxel mesh using MeshResource from MeshDescriptor and per-voxel opacity, https://stackoverflow.com/questions/79656898/realitykit-voxel-mesh-using-meshresource-from-meshdescriptor-and-per-voxel-opaci, fetched 2026-10-28 (unofficial)
