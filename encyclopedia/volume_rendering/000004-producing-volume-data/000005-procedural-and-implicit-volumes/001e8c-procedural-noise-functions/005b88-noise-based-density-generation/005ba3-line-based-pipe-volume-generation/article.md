# Line-Based Pipe Volume Generation


## Overview
Generates a 3D volume density field along predefined line paths using OpenSimplex noise to create pipe-like cave structures, which are later meshed via marching cubes [S1, unofficial]. It differs from general noise-based density generation by restricting the noise evaluation domain to the vicinity of parametric line segments, effectively carving volumetric tubes through space rather than filling a bounding box with uniform noise patterns [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A set of line segments defined by nodes and properties; voxel grid dimensions (`_voxel.numVertex`); noise parameters (`noiseIntensity`, `noiseScale`, `smoothingFactor`, `frequency`, `lacunarity`); boolean `isWorldSpaceNoise` [S1, unofficial].
*   **Output**: A 3D density field stored in `_pointsBuffer` (ComputeBuffer), which is subsequently processed by a marching cubes shader to produce a triangle mesh [S1, unofficial].

**Data Structures**
The method relies on three primary ComputeBuffers:
1.  `_pointsBuffer`: Size `_voxel.totalVertex`. Stride 16 bytes. Layout: `Vector4` (x, y, z, density) [S1, unofficial].
2.  `_linePropBuffer`: Size `_lineGenerator.line.Count`. Stride 52 bytes. Contains properties for each line segment [S1, unofficial].
3.  `_nodePositionBuffer`: Size `_lineGenerator.node.Count`. Stride 20 bytes. Contains node positions [S1, unofficial].

**Algorithm Steps**
1.  **Thread Group Calculation**: Calculate the number of thread groups per axis based on the voxel vertex count. The dimension is `ceil(numVertex / 8) + 1`, where 8 is the number of threads per group [S1, unofficial].
2.  **Buffer Initialization**: Populate `_linePropBuffer` and `_nodePositionBuffer` with current line generator data [S1, unofficial].
3.  **Shader Configuration**: Set the following shader inputs:
    *   Buffers: `nodeProps` (from `_nodePositionBuffer`), `lineProps` (from `_linePropBuffer`) [S1, unofficial].
    *   Scalars: `isWorldSpaceNoise`, `noiseIntensity`, `noiseScale`, `smoothingFactor`, `frequency`, `lacunarity` [S1, unofficial].
    *   Vector: `offset` (used for chunked generation) [S1, unofficial].
4.  **Density Computation**: Dispatch the density shader. For each voxel vertex, the shader evaluates OpenSimplex noise. If `isWorldSpaceNoise` is false, the query point is converted to the local coordinate system of the nearest line segment [S1, unofficial].
5.  **Smoothing**: Apply the `smoothingFactor` to the resulting density values to ensure smooth transitions [S1, unofficial].
6.  **Meshing**: The resulting density field is passed to a marching cubes shader (`_marchingCubeShader`) with an `isoLevel` to generate triangles [S1, unofficial].

**Pseudocode**
The following pseudocode represents the CPU-side setup and dispatch logic derived from the source [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
NUM_THREADS_PER_GROUP = 8

class Vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z

class Mesh:
    def __init__(self): self.vertices = []; self.indices = []
    def recalculate_normals(self): pass

def create_mesh(vertices, indices):
    mesh = Mesh()
    mesh.vertices = vertices
    mesh.indices = indices
    return mesh

def generate_pipe_volume(voxel, line_generator, iso_level):
    # 1. Calculate thread groups per axis
    num_groups_x = math.ceil(voxel.num_vertex.x / NUM_THREADS_PER_GROUP) + 1
    num_groups_y = math.ceil(voxel.num_vertex.y / NUM_THREADS_PER_GROUP) + 1
    num_groups_z = math.ceil(voxel.num_vertex.z / NUM_THREADS_PER_GROUP) + 1

    # 2. Prepare buffers
    line_props = line_generator.get_line_props()
    node_positions = line_generator.get_nodes()

    # 3. Initialize density field
    density_field = [0.0] * voxel.total_vertex

    # 4. Evaluate noise and compute density (simulating GPU dispatch)
    for k in range(num_groups_z):
        for j in range(num_groups_y):
            for i in range(num_groups_x):
                offset = Vector3(i * voxel.boundary_world_pos.x,
                                 j * voxel.boundary_world_pos.y,
                                 k * voxel.boundary_world_pos.z)
                # Evaluate OpenSimplex noise for each vertex in the chunk
                for idx in range(voxel.total_vertex):
                    pos = Vector3(*voxel.vertex_positions[idx])
                    # Apply offset and coordinate conversion if isWorldSpaceNoise is False
                    query_pos = pos + offset
                    noise_val = open_simplex_noise(query_pos, line_props, node_positions) # [G]
                    density_field[idx] = noise_val * voxel.noise_intensity

    # 5. Marching Cubes
    triangles = marching_cubes(density_field, voxel.num_voxel, iso_level) # [G]

    # 6. Retrieve and Construct Mesh
    num_tris = len(triangles)
    mesh_vertices = []
    mesh_indices = []
    for i in range(num_tris):
        for j in range(3):
            mesh_indices.append(i * 3 + j)
            mesh_vertices.append(triangles[i][j])

    mesh = create_mesh(mesh_vertices, mesh_indices)
    mesh.recalculate_normals()
    return mesh
```

**Edge Cases and Pitfalls**
*   **0th Line Index**: A boundary/edge case issue occurs specifically at the 0th line index during pipe volume generation [S1, unofficial].
*   **Coordinate Systems**: If `isWorldSpaceNoise` is disabled, coordinates are automatically converted to the line's local coordinate system. Incorrect implementation of this conversion may lead to artifacts [S1, unofficial].

## Operating characteristics
*   **Complexity**: Thread group dimensions are calculated as `ceil(numVertex / 8) + 1` per axis [S1, unofficial].
*   **Performance**: The source logs indicate performance measurement in seconds, but specific hardware metrics are not provided [S1, unofficial].
*   **Strengths**: Efficiently generates tubular structures (caves/pipes) without needing to evaluate noise across the entire volume uniformly [S1, unofficial].
*   **Limitations**: Dependent on the quality of the underlying OpenSimplex noise implementation; potential artifacts at line boundaries or index 0 [S1, unofficial].

## Background
Originated from an unofficial GitHub repository by `sachPico` in April 2020. The project, `lineBasedCave_Simplex`, was developed as a variation of procedural cave generation using line data for paths and OpenSimplex noise for volume data [S1, unofficial]. It was intended for use in Unity, leveraging Compute Shaders for density generation and Marching Cubes for mesh extraction [S1, unofficial].

## Sources
[S1] sachPico/lineBasedCave_Simplex, https://github.com/sachPico/lineBasedCave_Simplex, fetched 2026-10-27 (unofficial)
