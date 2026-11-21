# Rasterizing Bounding Geometry for Empty Space Skipping

Also called: Bounding Geometry Rasterization, Raster-based Bounding Box Culling, Rasterizing Bounding Geometry

## Overview
Rasterizing Bounding Geometry for Empty Space Skipping is a volume rendering optimization technique that uses hardware-accelerated rasterization to determine the start and end positions for ray marching, thereby skipping empty space [S1, unofficial][S2, unofficial]. It solves the problem of expensive tree traversal on the GPU by replacing hierarchical data structure lookups with fragment shader depth testing [S2, unofficial]. It sets itself apart from [Min-Max Block Culling](8368-min-max-block-culling.md) by using explicit geometric primitives (quads/boxes) and depth buffers rather than scalar min-max values, and from [Early Shader Termination](969-early-shader-termination.md) by performing culling in a dedicated geometry pass rather than within the ray-marching loop [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume texture containing iso values (or occupancy data), a threshold iso value, and a coarse grid definition (e.g., grid cells spanning $N^3$ voxels) [S1, unofficial].
*   **Output:** A rendered image where rays are marched only through active regions, with start and end positions derived from rasterized depth values [S1, unofficial].

**Data Structures:**
1.  **`gridTex`**: A 3D texture where each voxel stores the maximum iso value of the corresponding coarse grid cell [S1, unofficial].
2.  **Bounding Geometry**: A set of quads (or bounding boxes) representing the boundaries between active and inactive regions in `gridTex` [S1, unofficial].
3.  **Offscreen Buffer**: A depth buffer (and optionally a color buffer) used to store minimum and maximum depth values for each fragment during the first pass [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization**:
    *   Construct `gridTex` by sampling the volume texture. Each voxel in `gridTex` contains the maximum iso value within its corresponding coarse grid cell [S1, unofficial].
    *   Define the coarse grid dimensions. For a volume of size $V_x \times V_y \times V_z$ and a cell size of $C^3$ voxels, the grid dimensions are $\lceil V_x/C \rceil \times \lceil V_y/C \rceil \times \lceil V_z/C \rceil$ [S1, unofficial].

2.  **Generate Bounding Geometry**:
    *   For each pair of neighboring voxels in `gridTex`, determine if a boundary exists.
    *   A grid voxel is considered **active** if its maximum iso value is above the threshold iso value [S1, unofficial].
    *   Generate a quad (or bounding box face) if one voxel in the neighbor pair is active and the other is inactive [S1, unofficial].
    *   This process identifies the surface of the active volume data [S1, unofficial].

3.  **First Pass (Depth Computation)**:
    *   Render the generated bounding geometry to an offscreen buffer.
    *   For each fragment, store the **minimum depth** (front face) and **maximum depth** (back face) of the bounding geometry [S1, unofficial].
    *   This pass effectively projects the 3D active regions onto the 2D screen, capturing the depth extent of the volume along each ray [S2, unofficial].

4.  **Second Pass (Ray Marching)**:
    *   Render the front faces of the bounding geometry again to the main buffer.
    *   In the fragment shader, discard the fragment if it is **not** at the minimum depth value recorded in the first pass [S1, unofficial]. This ensures that only the front-most active region is processed, preventing double-counting or processing of occluded empty space [S1, unofficial].
    *   Compute the **start position** of the ray using the minimum depth value and the **end position** using the maximum depth value from the first pass [S1, unofficial].
    *   March the ray between these start and end positions using standard volume rendering techniques (e.g., ray casting) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Rasterizing Bounding Geometry for Empty Space Skipping

def initialize_grid(volume_texture, cell_size):
    """
    Create a coarse grid texture storing max iso values.
    """
    grid_dims = (
        ceil(volume_texture.width / cell_size),
        ceil(volume_texture.height / cell_size),
        ceil(volume_texture.depth / cell_size)
    )
    grid_tex = create_3d_texture(grid_dims)
    
    for z in range(grid_dims[2]):
        for y in range(grid_dims[1]):
            for x in range(grid_dims[0]):
                # Sample the volume texture for this coarse cell
                max_val = -infinity
                for dz in range(cell_size):
                    for dy in range(cell_size):
                        for dx in range(cell_size):
                            vx = x * cell_size + dx
                            vy = y * cell_size + dy
                            vz = z * cell_size + dz
                            val = volume_texture.sample(vx, vy, vz)
                            if val > max_val:
                                max_val = val
                grid_tex.set(x, y, z, max_val)
    return grid_tex

def generate_bounding_geometry(grid_tex, threshold, cell_size, volume_dims):
    """
    Generate quads for boundaries between active and inactive regions.
    """
    quads = []
    grid_dims = grid_tex.dimensions
    
    for z in range(grid_dims[2]):
        for y in range(grid_dims[1]):
            for x in range(grid_dims[0]):
                is_active = grid_tex.get(x, y, z) > threshold
                
                # Check neighbors in +x, +y, +z directions
                neighbors = [
                    (x + 1, y, z),
                    (x, y + 1, z),
                    (x, y, z + 1)
                ]
                
                for nx, ny, nz in neighbors:
                    if nx < grid_dims[0] and ny < grid_dims[1] and nz < grid_dims[2]:
                        neighbor_active = grid_tex.get(nx, ny, nz) > threshold
                        if is_active != neighbor_active:
                            # Generate a quad at the boundary
                            quad = create_quad_at_boundary(x, y, z, nx, ny, nz, cell_size, volume_dims)
                            quads.append(quad)
    return quads

def create_quad_at_boundary(x, y, z, nx, ny, nz, cell_size, volume_dims):
    """
    Create a quad representing the boundary between two grid cells.
    """
    # Calculate world coordinates of the boundary plane
    # This is a simplified representation; actual implementation depends on coordinate system
    boundary_pos = get_boundary_position(x, y, z, nx, ny, nz, cell_size)
    return Quad(position=boundary_pos, normal=get_normal(nx, ny, nz))

def render_first_pass(quads, offscreen_buffer):
    """
    Render bounding geometry to compute min/max depth.
    """
    # Clear depth buffer
    offscreen_buffer.clear_depth()
    
    # Render all quads
    for quad in quads:
        # Render front and back faces to capture depth range
        # Store min and max depth for each fragment
        offscreen_buffer.render(quad, store_min_max_depth=True)
    return offscreen_buffer

def render_second_pass(quads, offscreen_buffer, main_buffer):
    """
    Render front faces and perform ray marching.
    """
    # Render front faces only
    for quad in quads:
        # In fragment shader:
        # 1. Check if current fragment depth == min_depth from first pass
        # 2. If not, discard
        # 3. If yes, compute ray start/end from min/max depth
        # 4. March ray and color pixel
        
        main_buffer.render(quad, fragment_shader=ray_marching_shader, 
                           depth_buffer=offscreen_buffer)

def ray_marching_shader(fragment_depth, min_depth, max_depth):
    """
    Fragment shader for ray marching.
    """
    # Discard if not at minimum depth
    if fragment_depth != min_depth:
        discard()
    
    # Compute ray start and end positions from depth values
    ray_start = depth_to_world(min_depth)
    ray_end = depth_to_world(max_depth)
    
    # March ray
    color = march_ray(ray_start, ray_end)
    return color
```

**Parameters:**
*   **Threshold Iso Value**: The iso value above which a grid cell is considered active. Typical values depend on the data distribution [S1, unofficial].
*   **Cell Size**: The number of voxels per coarse grid cell. A common example is $8^3$ voxels per cell [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Iso Value Changes**: The bounding geometry must be regenerated whenever the threshold iso value changes, as active regions may shift [S1, unofficial].
*   **Precision**: Depth buffer precision must be sufficient to distinguish between front and back faces, especially for thin structures [S1, unofficial].
*   **Occlusion**: The method relies on depth testing to handle occlusion. If the depth buffer is not properly cleared or updated, incorrect start/end positions may be computed [S1, unofficial].

## Subtopics

### [SparseLeap Rasterization](5154-sparseleap-rasterization.md)
SparseLeap is a method for efficient empty space skipping in very large, sparse volume data, such as terabyte-sized microscopy volumes. It solves the problem of fragmented space subdivisions making empty regions hard to skip and the infeasibility of pre-computing well-adapted space subdivisions due to frequent changes in active objects. It sets itself apart from its siblings by exploiting hardware-accelerated rasterization to create per-pixel ray segment lists, thereby avoiding hierarchy traversal on the GPU during the ray-casting stage. Unlike standard octree empty space skipping, SparseLeap scales better to large, sparse data. It is agnostic of the underlying data structure used for managing volume data (e.g., page-table hierarchies or octrees).

### [Texture Box Approximation](13615-texture-box-approximation.md)
Texture Box Approximation is a method for accelerating volume rendering by approximating the volume data with a set of bounding boxes, called texture boxes, which are rasterized to efficiently clip empty space from the rendering pipeline. It differs from level-of-detail methods that rasterize lower-resolution blocks by focusing on spatial occupancy to discard empty regions entirely before fragment shader execution.

## Operating characteristics
*   **Time Complexity**: The complexity depends on the number of active grid cells and the cost of rasterization. Generating the bounding geometry is $O(N)$ where $N$ is the number of grid cells [S1, unofficial]. Rasterization is generally efficient on modern GPUs [S2, unofficial].
*   **Memory Complexity**: Requires storage for `gridTex`, the bounding geometry, and the offscreen buffer [S1, unofficial].
*   **Measured Performance**: Can lead to fewer look-ups into the empty-space skipping data structure during rendering when compared to octree structures [S2, unofficial].
*   **Strengths**: Avoids expensive tree traversal on the GPU; leverages hardware-accelerated rasterization [S2, unofficial].
*   **Limitations**: Requires regeneration of bounding geometry when iso values change [S1, unofficial]. May have overhead for generating the geometry if the volume is dense [S1, unofficial].
*   **When to Use**: Suitable for scenarios where tree traversal is a bottleneck and the volume data can be approximated by coarse grid cells [S2, unofficial].
*   **When Not to Use**: Not suitable for highly dynamic volumes where the active regions change frequently, as the geometry regeneration cost may outweigh the benefits [S1, unofficial].

## Background
The technique was proposed by Liu et al. [LCDP13] and Hadwiger et al. [HAAB+ 18] as a way to avoid hierarchy traversal on the GPU by using rasterization hardware [S2, unofficial]. It is part of the broader field of object-order empty space skipping [S2, unofficial].

## Sources
[S1] implement volume rendering with object order empty space skipping, https://stackoverflow.com/questions/12216008/implement-volume-rendering-with-object-order-empty-space-skipping, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
