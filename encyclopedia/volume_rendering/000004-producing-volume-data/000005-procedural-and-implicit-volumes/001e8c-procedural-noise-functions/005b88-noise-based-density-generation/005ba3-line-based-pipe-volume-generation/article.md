# Line-Based Pipe Volume Generation

Also called: Line-Based Cave Generation, Procedural Pipe Volume with OpenSimplex Noise

## Overview
Generates a 3D volume density field along predefined line paths using OpenSimplex noise to create pipe-like cave structures, which are later meshed via marching cubes [S1, unofficial]. It differs from general noise-based density generation by restricting the noise evaluation domain to the vicinity of parametric line segments, effectively carving volumetric tubes through space rather than filling a bounding box with uniform noise patterns [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A set of line segments defined by nodes and properties; voxel grid dimensions (`_voxel.numVertex`); noise parameters (`noiseIntensity`, `noiseScale`, `smoothingFactor`, `frequency`, `lacunarity`); boolean `isWorldSpaceNoise`.
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
# Constants
NUM_THREADS_PER_GROUP = 8

def generate_pipe_volume(voxel, line_generator, density_shader, marching_cube_shader, iso_level):
    # 1. Calculate thread groups
    # Note: Source uses numVertex for calculation, likely referring to voxel grid dimensions
    num_threads_x = math.ceil(voxel.numVertex.x / NUM_THREADS_PER_GROUP) + 1
    num_threads_y = math.ceil(voxel.numVertex.y / NUM_THREADS_PER_GROUP) + 1
    num_threads_z = math.ceil(voxel.numVertex.z / NUM_THREADS_PER_GROUP) + 1
    
    # 2. Prepare buffers
    line_props = line_generator.get_line_props()
    node_positions = line_generator.get_nodes()
    
    # Set buffer data
    density_shader.set_buffer("lineProps", line_props)
    density_shader.set_buffer("nodeProps", node_positions)
    
    # 3. Set shader parameters
    density_shader.set_bool("isWorldSpaceNoise", True) # Example default
    density_shader.set_float("noiseIntensity", 1.0)
    density_shader.set_float("noiseScale", 1.0)
    density_shader.set_float("smoothingFactor", 0.5)
    density_shader.set_float("frequency", 1.0)
    density_shader.set_float("lacunarity", 2.0)
    
    # 4. Dispatch Density Shader
    # The source loops over chunks (i, j, k) with offsets. 
    # For a single volume generation step:
    offset = Vector3(0, 0, 0)
    density_shader.set_vector("offset", offset)
    
    density_shader.dispatch(0, num_threads_x, num_threads_y, num_threads_z)
    
    # 5. Marching Cubes
    marching_cube_shader.set_buffer("triangles", _mesh_buffer)
    marching_cube_shader.set_float("isoLevel", iso_level)
    marching_cube_shader.dispatch(0, voxel.numVoxel.x, voxel.numVoxel.y, voxel.numVoxel.z)
    
    # 6. Retrieve and Construct Mesh
    tri_count = _tri_count_buffer.get_data()
    triangles = _mesh_buffer.get_data(tri_count)
    
    mesh = create_mesh(triangles)
    mesh.recalculate_normals()
    return mesh
```

**Edge Cases and Pitfalls**
*   **0th Line Index**: A boundary/edge case issue occurs specifically at the 0th line index during pipe volume generation [S1, unofficial].
*   **Coordinate Systems**: If `isWorldSpaceNoise` is disabled, coordinates are automatically converted to the line's local coordinate system. Incorrect implementation of this conversion may lead to artifacts [S1, unofficial].

## Operating characteristics
*   **Complexity**: Thread group dimensions are calculated as `ceil(numVertex / 8) + 1` per axis [S1, unofficial].
*   **Performance**: The source logs indicate performance measurement in seconds, but specific hardware metrics are not provided [S1, unofficial].
*   **Strengths**: Efficiently generates tubular structures (caves/pipes) without needing to evaluate noise across the entire volume uniformly.
*   **Limitations**: Dependent on the quality of the underlying OpenSimplex noise implementation; potential artifacts at line boundaries or index 0 [S1, unofficial].

## Background
Originated from an unofficial GitHub repository by `sachPico` in April 2020. The project, `lineBasedCave_Simplex`, was developed as a variation of procedural cave generation using line data for paths and OpenSimplex noise for volume data [S1, unofficial]. It was intended for use in Unity, leveraging Compute Shaders for density generation and Marching Cubes for mesh extraction [S1, unofficial].

## Sources
[S1] sachPico/lineBasedCave_Simplex, https://github.com/sachPico/lineBasedCave_Simplex, fetched 2026-10-27 (unofficial)
