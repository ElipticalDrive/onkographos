# Voxel Shapes
## Overview
Voxel Shapes define the geometric primitive used to represent individual solid voxels, supporting Cubic, Lego, Marching Cubes, Sphere, Cylinder, and Clay shapes [S1].
## Shared principles
The family operates on the output of [Voxel Face and Geometry Generation](11724-voxel-face-and-geometry-generation.md), which determines which voxel faces are exposed [G]. The core mechanism involves replacing the standard cubic quad emission with shape-specific vertex generation logic [G].
1. **Shape Selection:** A global or per-voxel parameter selects the target primitive (Cubic, Lego, Marching Cubes, Sphere, Cylinder, Clay) [S1].
2. **World Scale:** A parameter allows rectangular (non-uniform) scaling of voxel coordinates in world space [S1].
3. **SV Mode:** A specific rendering mode restricts shapes to cubic voxels only [S1].
4. **Assembly:** The generated primitives are assembled into a mesh, maintaining the culling and winding rules defined in the parent [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_face_mesh(chunk_data, chunk_width, chunk_height, chunk_depth, block_size):
    vertices = []
    indices = []
    vertex_count = 0

    FACES = [
        {"normal": (0, 1, 0), "verts": [(0, 1, 0), (1, 1, 0), (1, 1, 1), (0, 1, 1)], "indices": [0, 1, 2, 0, 2, 3]}, # Up
        {"normal": (0, -1, 0), "verts": [(0, 0, 0), (1, 0, 0), (1, 0, 1), (0, 0, 1)], "indices": [0, 1, 2, 0, 2, 3]}, # Down
        {"normal": (0, 0, 1), "verts": [(0, 0, 1), (1, 0, 1), (1, 1, 1), (0, 1, 1)], "indices": [0, 1, 2, 0, 2, 3]}, # North
        {"normal": (0, 0, -1), "verts": [(0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0)], "indices": [0, 1, 2, 0, 2, 3]}, # South
        {"normal": (1, 0, 0), "verts": [(1, 0, 0), (1, 1, 0), (1, 1, 1), (1, 0, 1)], "indices": [0, 1, 2, 0, 2, 3]}, # East
        {"normal": (-1, 0, 0), "verts": [(0, 0, 0), (0, 1, 0), (0, 1, 1), (0, 0, 1)], "indices": [0, 1, 2, 0, 2, 3]}  # West
    ]

    def get_block(x, y, z):
        if not (0 <= x < chunk_width and 0 <= y < chunk_height and 0 <= z < chunk_depth):
            return 0
        return chunk_data[x + chunk_width * (y + chunk_height * z)]

    AIR_BLOCK = 0

    for z in range(chunk_depth):
        for y in range(chunk_height):
            for x in range(chunk_width):
                block_id = get_block(x, y, z)
                if block_id == AIR_BLOCK:
                    continue

                for face in FACES:
                    nx, ny, nz = face["normal"]
                    neighbor_x, neighbor_y, neighbor_z = x + nx, y + ny, z + nz
                    neighbor_block = get_block(neighbor_x, neighbor_y, neighbor_z)

                    if neighbor_block == AIR_BLOCK:
                        base_x, base_y, base_z = x * block_size, y * block_size, z * block_size
                        for vx, vy, vz in face["verts"]:
                            vertices.append((base_x + vx * block_size, base_y + vy * block_size, base_z + vz * block_size))
                        for idx in face["indices"]:
                            indices.append(vertex_count + idx)
                        vertex_count += 4

    return vertices, indices
```

## Subtopics

### Cubic
The default axis-aligned cube primitive [G]. Implementation details link to [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) [G].

### Lego
A primitive mimicking interlocking bricks with studs and tubes [G]. Implementation details link to [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) [G].

### Marching Cubes
A smooth surface approximation generated via the Marching Cubes algorithm on the voxel grid [G]. Implementation details link to [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) [G].

### Sphere
A spherical primitive [G]. Implementation details link to [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) [G].

### Cylinder
A cylindrical primitive [G]. Implementation details link to [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) [G].

### Clay
A rounded, organic shape resembling modeling clay [G]. Implementation details link to [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) [G].

## Background
The concept of varying voxel shapes was introduced in MagicaVoxel to support diverse artistic styles beyond the standard cubic grid [S1]. Early versions supported Lego, Marching Cubes, Sphere, and Cylinder shapes [S1]. The "Clay" shape was added in a later update (version 0.99.1, March 12, 2018) to expand the rendering volume's geometric variety [S1]. The feature set allows for rectangular voxels via the "World Scale" parameter [S1].
## Sources
[S1] MagicaVoxel .vox, https://ephtracy.github.io/, fetched 2026-10-26
