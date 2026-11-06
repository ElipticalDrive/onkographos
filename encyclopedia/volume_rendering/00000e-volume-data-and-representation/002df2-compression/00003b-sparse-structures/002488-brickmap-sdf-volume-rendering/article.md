# Brickmap SDF Volume Rendering

Also called: CSG SDF Raymarching, SDF-based volume algorithm

## Overview
Brickmap SDF Volume Rendering is a hybrid technique that represents 3D scenes as Constructive Solid Geometry (CSG) trees of Signed Distance Function (SDF) primitives, evaluates the resulting distance field on the CPU, compresses it into a sparse Brickmap structure, and raymarches through this compressed data on the GPU [S1, unofficial]. It solves the problem of high memory consumption and GPU branching overhead associated with direct GPU-based CSG evaluation or dense volume grids [S1, unofficial]. It differs from sibling sparse structures by explicitly decoupling the geometric evaluation (CPU) from the rendering traversal (GPU) and using a fixed-size brick compression scheme tailored for SDF data, achieving significant memory reduction compared to dense arrays [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A CSG tree composed of SDF primitives (e.g., cylinders, boxes) and boolean operations (union, intersection, difference), defined within a bounding box (typically $[-1, 1]^3$) [S1, unofficial].
*   **Output:** A compressed Brickmap data structure containing distance and color data, and a GPU shader capable of raymarching through this structure [S1, unofficial].

**Data Structures:**
The Brickmap consists of four dynamic arrays that store only non-empty or relevant data. The arrays resize dynamically based on the number of used elements ($n_{used}$) versus available capacity ($n_{available}$) [S1, unofficial]:
1.  **Brick Indices:** A 3D grid of indices mapping logical brick positions to the data arrays. For a $73^3$ grid, this uses $73^3 \times 4$ bytes [S1, unofficial].
2.  **Distance Bricks:** Stores the SDF values. Each brick contains $8^3$ voxels, packed as 1 byte per voxel. The number of bricks depends on the surface area complexity (e.g., 14,716 used out of 19,683 available) [S1, unofficial].
3.  **Color Index Remapping:** An array mapping each distance brick to a corresponding color brick. Size is one 4-byte index per distance brick (e.g., 14,716/16,384 elements) [S1, unofficial].
4.  **Color Bricks:** Stores RGB color data. Each brick contains $8^3$ voxels, packed as 3 bytes per voxel (RGB). The number of bricks depends on color variation (e.g., 2,312 used out of 2,744 available) [S1, unofficial].

**Algorithm Steps:**
1.  **CSG Tree Construction:** Define the scene geometry using SDF primitives and boolean operations. Example primitives include `Cylinder`, `Rect`, and operations like `union`, `intersect`, `diff`, and `smooth_union` [S1, unofficial].
2.  **CPU Distance Field Evaluation:** Evaluate the CSG tree over a dense 3D grid (e.g., $512^3$ resolution) within the bounding box. This generates a raw distance field array [S1, unofficial].
3.  **Brickmap Compression:**
    *   Divide the dense grid into $8^3$ voxel bricks.
    *   Identify non-empty bricks (those containing surface data or relevant SDF values).
    *   Store these bricks in the Distance Bricks array.
    *   Map each distance brick to a color brick via the Color Index Remapping array.
    *   Store the color data in the Color Bricks array.
    *   Populate the Brick Indices array to allow fast lookup of brick locations [S1, unofficial].
4.  **GPU Raymarching:**
    *   Cast rays from the camera.
    *   Traverse the Brickmap using the Brick Indices to locate the relevant distance and color bricks.
    *   Sample the SDF value from the Distance Bricks to determine step size and surface proximity.
    *   Sample color from the Color Bricks.
    *   Accumulate color and depth until the ray hits the surface or exits the bounding box [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU Side: CSG Evaluation and Compression

def evaluate_csg_tree(tree, grid_resolution, bounding_box):
    """
    Evaluates the CSG tree on a dense 3D grid.
    """
    # Initialize dense grid with default values (e.g., infinity)
    dense_grid = zeros(grid_resolution, grid_resolution, grid_resolution)
    
    # Define voxel size based on bounding box
    voxel_size = (bounding_box.max - bounding_box.min) / grid_resolution
    
    # Iterate over all voxels in the grid
    for x in range(grid_resolution):
        for y in range(grid_resolution):
            for z in range(grid_resolution):
                # Convert grid index to world coordinate
                world_pos = bounding_box.min + voxel_size * (x + 0.5, y + 0.5, z + 0.5)
                
                # Evaluate SDF at this position
                distance = evaluate_sdf_at(tree, world_pos)
                
                # Store distance in dense grid
                dense_grid[x, y, z] = distance
                
    return dense_grid

def compress_to_brickmap(dense_grid, brick_size=8):
    """
    Compresses the dense grid into a Brickmap structure.
    """
    brick_indices = {} # Maps (brick_x, brick_y, brick_z) to index in data arrays
    distance_bricks = []
    color_bricks = []
    color_indices = [] # Maps distance brick index to color brick index
    
    grid_res = dense_grid.shape[0]
    
    # Iterate over bricks
    for bx in range(0, grid_res, brick_size):
        for by in range(0, grid_res, brick_size):
            for bz in range(0, grid_res, brick_size):
                # Extract brick data
                brick_data = dense_grid[bx:bx+brick_size, by:by+brick_size, bz:bz+brick_size]
                
                # Check if brick is non-empty (simplified check: any value < infinity)
                if not is_empty_brick(brick_data):
                    # Store distance brick
                    dist_brick_index = len(distance_bricks)
                    distance_bricks.append(brick_data)
                    
                    # Determine color for this brick (simplified: average or first non-background color)
                    # In practice, color is evaluated alongside distance or stored separately
                    color_brick = compute_color_for_brick(bx, by, bz, brick_data)
                    color_brick_index = len(color_bricks)
                    color_bricks.append(color_brick)
                    color_indices.append(color_brick_index)
                    
                    # Store index in brick indices map
                    brick_indices[(bx, by, bz)] = dist_brick_index
                    
    return Brickmap(brick_indices, distance_bricks, color_bricks, color_indices)

def is_empty_brick(brick_data):
    """
    Checks if a brick contains no surface data.
    """
    # Assuming background distance is infinity or a large value
    return all(d == float('inf') for d in brick_data.flatten())

def compute_color_for_brick(bx, by, bz, brick_data):
    """
    Computes color for a brick.
    """
    # Simplified: return a default color or average color of surface voxels
    # This is a placeholder for actual color evaluation logic
    return zeros(brick_size, brick_size, brick_size, 3)

# GPU Side: Raymarching

def raymarch_brickmap(ray_origin, ray_direction, brickmap):
    """
    Raymarches through the Brickmap to render the scene.
    """
    t = 0.0
    max_steps = 1000
    step_size = 0.01 # Adaptive step size based on SDF
    
    for _ in range(max_steps):
        # Current ray position
        pos = ray_origin + t * ray_direction
        
        # Find brick index for current position
        brick_coords = world_to_brick_coords(pos, brickmap.brick_indices)
        
        if brick_coords not in brickmap.brick_indices:
            # Ray is in empty space, skip to next brick or exit
            t += step_size * 10 # Skip large empty regions
            continue
            
        # Get distance brick index
        dist_brick_index = brickmap.brick_indices[brick_coords]
        
        # Sample SDF from distance brick
        local_pos = world_to_local_brick_pos(pos, brick_coords)
        sdf_value = sample_distance_brick(brickmap.distance_bricks[dist_brick_index], local_pos)
        
        if sdf_value < 0.001:
            # Hit surface
            color = sample_color_brick(brickmap.color_bricks[brickmap.color_indices[dist_brick_index]], local_pos)
            return color
            
        # Advance ray
        t += sdf_value # Use SDF value for adaptive step size
        
    return background_color
```

**Parameters:**
*   **Grid Resolution:** $512^3$ effective grid evaluated within a $[-1, 1]^3$ bounding box [S1, unofficial].
*   **Brick Dimensions:** $8^3$ voxels per brick [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **GPU Branching Overhead:** Direct GPU computation of CSG operations causes high branching overhead due to decoding operation types (e.g., `if (operation == op_translation)`) and evaluating multiple operations per raymarch step [S1, unofficial].
*   **Precision:** The compression from dense grid to bricks may introduce quantization errors if not handled carefully, though 1-byte distance storage is used [S1, unofficial].
*   **Empty Regions:** Efficiently skipping empty regions is crucial for performance. The Brickmap structure relies on identifying non-empty bricks to achieve compression [S1, unofficial].

## Operating characteristics
*   **Memory Usage:** Significant reduction compared to dense arrays. For a $512^3$ grid, a dense array requires 640MB (256MB distance + 384MB RGB). The Brickmap reduces this to ~13.2MB (1.5MB indices + 7.6MB distances + 0.06MB color remapping + 4.0MB colors) [S1, unofficial].
*   **Performance:** CPU generation time for the example scene (32 operations) was 19.3s unoptimized, reduced to 0.28s with optimizations (skipping empty regions and locally irrelevant SDF nodes) [S1, unofficial].
*   **Compression Ratio:** Typically achieves 10-100x compression [S1, unofficial].
*   **Strengths:** High quality on the GPU, faster than direct GPU CSG evaluation, lower memory footprint than dense grids [S1, unofficial].
*   **Limitations:** Requires CPU preprocessing, which may not be suitable for highly dynamic scenes where the CSG tree changes frequently. The current implementation lacks localized edits, requiring full regeneration on change [S1, unofficial].
*   **When to Use:** Suitable for static or slowly changing scenes with complex CSG geometry where memory efficiency and high-quality rendering are priorities [S1, unofficial].
*   **When Not to Use:** Unsuitable for real-time dynamic scenes with frequent CSG updates, or when GPU-only pipelines are required [S1, unofficial].

## Background
*   **Origin:** Inspired by a gamedev YouTube video on SDF-based raymarching [S1, unofficial].
*   **Development:** Implemented in Makie.jl (Julia) as `csgplot` with `Brickmap` compression [S1, unofficial]. The shape functions and CSG operations are derived from Inigo Quilez's articles on 3D signed distance functions [S1, unofficial].
*   **History:** The technique evolved through three prototypes: direct GPU CSG evaluation (expensive), CPU evaluation with dense arrays, and finally CPU evaluation with Brickmap compression [S1, unofficial].

## Sources
[S1] constructive solid geometry / SDF based volume algorithm, https://github.com/MakieOrg/Makie.jl/pull/5516, fetched 2026-10-26 (unofficial)
