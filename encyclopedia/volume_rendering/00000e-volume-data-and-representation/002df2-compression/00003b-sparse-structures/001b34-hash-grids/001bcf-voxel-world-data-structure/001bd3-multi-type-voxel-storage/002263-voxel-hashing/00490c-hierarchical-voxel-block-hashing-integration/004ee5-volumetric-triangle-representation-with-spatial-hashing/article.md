# Volumetric Triangle Representation with Spatial Hashing

Also called: Cube-based mesh representation, Spatial-hashed cube field

## Overview
This method provides a compact, incremental data structure for real-time dense 3D reconstruction by embedding mesh vertices directly into a volumetric grid via spatial hashing [S1]. It solves the problem of memory redundancy and inefficient vertex querying in loose mesh storage strategies by establishing a one-to-one correspondence between voxel edges and mesh vertices [S1]. Unlike hierarchical approaches that store TSDF at the voxel level and triangles at the block level, this method maintains both geometric primitives (triangles) and scalar fields (TSDF) at the fine cube scale, enabling $O(1)$ vertex access and eliminating duplicate vertex allocation [S1].

## How it works
**Inputs:** Depth images (from sensors like Kinect), camera poses, and an initial empty spatial hash structure.
**Outputs:** A reformatted triangle mesh suitable for traditional rasterization rendering.

**Data Structures:**
The structure employs a 2-level spatial management system [S1]:
1.  **Block Level:** Space is coarsely divided into blocks allocated only around object surfaces. These blocks are managed by a hash table to avoid collisions and constrain table size [S1].
2.  **Cube Level:** Each block is divided into an array of cubes, typically $8 \times 8 \times 8$ [S1].
    *   **Cube Layout:** To handle edge sharing between adjacent cubes, each cube stores only 1 corner $c$ and 3 outgoing edges $e_x, e_y, e_z$ [S1].
    *   **Corner:** $c = (x_0, y_0, z_0)$, the minimum coordinate of the cube.
    *   **Edges:** Defined relative to the corner:
        $$e_x = (l, 0, 0), \quad e_y = (0, l, 0), \quad e_z = (0, 0, l)$$
        where $l$ is the cube side length (e.g., 8mm in experiments) [S1].
    *   **TSDF:** The Truncated Signed Distance Field value $d(c)$ is stored at the corner $c$ [S1].
    *   **Vertices:** Vertices $v_x, v_y, v_z$ are stored on the respective edges $e_x, e_y, e_z$. Their positions are continuous but bound to the discrete edge coordinates [S1].
    *   **Triangles:** Up to 5 triangles are stored in a pointer array of size 5. These triangles connect vertices on the edges. The triangles may originate from the current cube or adjacent cubes [S1].
    *   **Cube Type:** A record indicating the number and shape of triangles (based on Marching Cubes topology) for both previous and current states [S1].
    *   **Memory Layout:** Edges and triangles are stored in pointer arrays (sizes 3 and 5 respectively). The pointed data (vertex coordinates, triangle indices) are heap-allocated [S1].

**Algorithm Steps:**

1.  **Spatial Hashing Initialization:**
    Initialize a hash table mapping block coordinates to block objects. Define cube resolution $l$ (e.g., 8mm) [S1].

2.  **Data Fusion (TSDF Update):**
    For each incoming depth image:
    a.  Project depth pixels into the volumetric space using known camera poses [S1].
    b.  For each affected cube, update the TSDF value $d(c)$ at its corner $c$ incrementally [S1].
    c.  This step changes the TSDF distributions at the corners of the cubes [S1].

3.  **Mesh Generation/Update:**
    a.  Identify cubes where the TSDF field crosses the truncation threshold (zero-crossing) [S1].
    b.  Apply Marching Cubes (MC) logic locally to determine the cube type and required triangle topology [S1].
    c.  Calculate vertex positions $v_x, v_y, v_z$ on the edges $e_x, e_y, e_z$ based on the TSDF values at the cube corners. The vertices are interpolated along the edges [S1].
    d.  Store vertices on the edges. Because edges are shared, a vertex is stored once per edge, preventing redundancy [S1].
    e.  Construct triangles connecting these vertices. Store pointers to these triangles in the cube's triangle array [S1].
    f.  Update the cube type record to reflect the new topology [S1].

4.  **Mesh Refinement (Optional):**
    a.  Analyze triangle shape consistency using Hamming distance metrics [S1].
    b.  Perform local shape regularization to improve mesh quality if necessary [S1].

5.  **Rendering:**
    a.  Traverse the hash table to retrieve all active cubes [S1].
    b.  Collect all triangles from the cube arrays [S1].
    c.  Reformat the collected data into a standard triangle list/strip format for traditional GPU rendering [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Cube:
    def __init__(self, corner_x, corner_y, corner_z, side_length):
        self.c = Vector3(corner_x, corner_y, corner_z)
        self.l = side_length
        # Edges defined as vectors from corner c
        self.ex = Vector3(side_length, 0, 0)
        self.ey = Vector3(0, side_length, 0)
        self.ez = Vector3(0, 0, side_length)
        
        # TSDF stored at corner
        self.tsdf = 0.0
        
        # Vertices on edges (initially None)
        self.vx = None
        self.vy = None
        self.vz = None
        
        # Triangle pointers (max 5)
        self.triangles = [None] * 5
        self.triangle_count = 0
        
        # Cube type for MC topology
        self.cube_type = 0

class Block:
    def __init__(self, grid_size):
        # Array of cubes, e.g., 8x8x8
        self.cubes = [Cube(0,0,0,0) for _ in range(grid_size**3)]

class VolumetricTriangleRepresentation:
    def __init__(self, cube_resolution):
        self.hash_table = {} # Maps block_key -> Block
        self.cube_resolution = cube_resolution

    def get_cube_key(self, world_pos):
        # Hash world position to block and local cube index
        block_key = self.get_block_key(world_pos)
        local_idx = self.get_local_cube_index(world_pos)
        return block_key, local_idx

    def fuse_depth(self, depth_image, camera_pose):
        # Project depth points to world space
        for point in depth_image.points:
            world_pos = camera_pose.transform(point)
            block_key, local_idx = self.get_cube_key(world_pos)
            
            if block_key not in self.hash_table:
                self.hash_table[block_key] = Block(8) # 8x8x8 blocks
            
            block = self.hash_table[block_key]
            cube = block.cubes[local_idx]
            
            # Update TSDF at corner
            cube.tsdf = update_tsdf(cube.tsdf, distance_from_surface)

    def generate_mesh(self):
        triangles = []
        for block_key, block in self.hash_table.items():
            for cube in block.cubes:
                # Determine MC type based on TSDF at corners of this cube
                # Note: TSDF at adjacent cube corners is needed for full MC
                # This requires accessing neighbors via hash table
                mc_type = compute_marching_cubes_type(cube, self.hash_table)
                
                if mc_type == 0:
                    continue
                
                # Interpolate vertices on edges
                # vx is on ex, vy on ey, vz on ez
                # Position is determined by linear interpolation where TSDF=0
                cube.vx = interpolate_vertex(cube.c, cube.c + cube.ex, cube.tsdf_neighbors_ex)
                cube.vy = interpolate_vertex(cube.c, cube.c + cube.ey, cube.tsdf_neighbors_ey)
                cube.vz = interpolate_vertex(cube.c, cube.c + cube.ez, cube.tsdf_neighbors_ez)
                
                # Generate triangles based on mc_type
                # Store pointers to vertices
                new_triangles = create_triangles_from_type(mc_type, cube.vx, cube.vy, cube.vz)
                
                for i, tri in enumerate(new_triangles):
                    if i < 5:
                        cube.triangles[i] = tri
                        triangles.append(tri)
                        cube.triangle_count += 1
        
        return triangles

    def render(self):
        mesh_data = self.generate_mesh()
        # Reformat for traditional rendering (e.g., vertex buffers)
        return format_for_renderer(mesh_data)
```

**Parameters:**
*   **Cube Resolution ($l$):** The side length of each cube. Typical value: 8mm [S1].
*   **Block Subdivision:** Blocks are typically divided into $8 \times 8 \times 8$ cubes [S1].

**Edge Cases and Pitfalls:**
*   **Memory Redundancy:** Loose mesh storage strategies often allocate duplicate vertices. This method forbids this by using a one-to-one correspondence between voxel edges and mesh vertices [S1].
*   **Hash Collisions:** The 2-level structure (blocks then cubes) helps constrain hash table size and avoid collisions, but collisions must still be handled by the hash table implementation [S1].
*   **Neighbor Access:** To compute TSDF for MC, a cube needs TSDF values from its 8 corners. Since each cube only stores its own corner, accessing adjacent cubes' corners is required. This is done via hash table lookup plus local indexing [S1].

## Operating characteristics
*   **Time Complexity:** Vertex accessing is $O(1)$ via hash table visiting plus local indexing [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of active cubes near the surface. The 2-level structure minimizes hash table size [S1].
*   **Performance:** Designed for real-time dense reconstruction in robotics. Experiments on the 'burghers' scene demonstrated effective online mesh storage and manipulation [S1].
*   **Strengths:** Eliminates vertex redundancy; enables $O(1)$ vertex access; supports parallel execution (lock-based and lock-free operations mentioned) [S1].
*   **Limitations:** Requires careful management of cube types and triangle pointers; dependent on accurate camera poses (though can use SLAM as a black box) [S1].
*   **When to use:** Real-time dense reconstruction for robotics where online mesh storage and manipulation are required [S1].
*   **When not to use:** Scenes with extremely large volumes where sparse structures are insufficient, or when high-fidelity global mesh optimization is prioritized over real-time incremental updates [S1].

## Background
The method was proposed by Wei Dong, Jieqi Shi, Weijie Tang, Xin Wang, and Hongbin Zha from Peking University, China [S1]. It was presented at the IEEE International Conference on Robotics and Automation (ICRA) in 2018 [S1]. The research was supported by the Natural Science Foundation of China (No. 61632003) [S1]. It addresses the gap in efficient incremental frameworks for online mesh storage and manipulation in dense reconstruction pipelines [S1].

## Sources
[S1] An Efficient Volumetric Mesh Representation for Real-Time Scene Reconstruction Using Spatial Hashing, https://doi.org/10.1109/icra.2018.8463157, fetched 2026-10-26
