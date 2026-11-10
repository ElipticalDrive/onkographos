# Point Cloud Voxelization Visualization

Also called: PCL VoxelGrid, PCL Voxelization Viewer, Voxel Grid Filtering and Visualization, Voxel Grid Rendering, Voxelization via PCL

## Overview
This method visualizes the occupancy of a point cloud after voxelization using the Point Cloud Library (PCL) [S1, S2, unofficial]. It supports axis-aligned bounding boxes (AABB) or custom fixed-resolution grids, and offers three rendering modes: point cloud coloring, solid cubes, or wireframe cubes [S1, unofficial]. It differs from standard hard voxelization by focusing on the visual representation of voxel centers and occupancy rather than producing a dense tensor for deep learning [S1, S2, unofficial].

## How it works
**Inputs:**
- `cloud`: Input point cloud (e.g., from a PCD file) [S2, unofficial].
- `voxelSize`: Side length of the voxel grid cell [S1, S2, unofficial].
- `box_mode`: "AABB" (computed from min/max points) or "CUSTOM" (fixed resolution) [S1, unofficial].
- `rendering_mode`: "A" (points only), "B" (solid cubes), or "C" (wireframe cubes + points) [S1, unofficial].

**Outputs:**
- Visualizer window displaying the selected rendering mode [S1, S2, unofficial].

**Data Structures:**
- `center_cloud`: Array of points with fields `point_X`, `point_Y`, `point_Z` representing the center coordinates of occupied voxels [S1, unofficial].
- `cloud_filtered`: Downsampled point cloud resulting from the VoxelGrid filter [S2, unofficial].

**Steps:**
1. Load the point cloud data from a PCD file into `cloud` [S2, unofficial].
2. Create a VoxelGrid filter object `sor` [S2, unofficial].
3. Set the input cloud to `sor` [S2, unofficial].
4. Set the leaf size (voxel size) to `voxelSize` [S2, unofficial].
5. Execute the filter to produce `cloud_filtered` [S2, unofficial].
6. Compute voxel centers for visualization:
   - For each point $p = (x, y, z)$ in the filtered cloud (or original cloud for Mode A/C in CUSTOM mode):
     $$
     c_x = \left\lfloor \frac{x}{v} \right\rfloor \cdot v + \frac{v}{2}
     $$
     $$
     c_y = \left\lfloor \frac{y}{v} \right\rfloor \cdot v + \frac{v}{2}
     $$
     $$
     c_z = \left\lfloor \frac{z}{v} \right\rfloor \cdot v + \frac{v}{2}
     $$
     where $v$ is `voxelSize` [S1, S2, unofficial].
   - Store these centers in `center_cloud` if using CUSTOM mode [S1, unofficial].
7. Initialize a PCL Visualizer and set the background color (e.g., white or black) [S1, S2, unofficial].
8. Add the bounding box wireframe:
   - If `box_mode` is "AABB", compute min/max points and add a cube wireframe [S1, unofficial].
   - If `box_mode` is "CUSTOM", add a fixed-size cube wireframe [S1, unofficial].
   - Set the rendering property to wireframe [S1, unofficial].
9. Render based on `rendering_mode`:
   - **Mode A**: Color the point cloud by the Z-axis field and add it to the viewer [S1, S2, unofficial].
   - **Mode B**: Iterate through `cloud_filtered` (or `center_cloud` for CUSTOM), compute voxel centers, and add solid cubes at each center [S1, S2, unofficial].
   - **Mode C**: Combine Mode A and Mode B, adding wireframe rendering to each individual cube [S1, S2, unofficial].
10. Spin the viewer in a loop with a sleep interval (e.g., 100ms) [S1, unofficial].

**Parameters:**
- `voxelSize`: Typical value for hairstyle point data is between 0.011 and 0.016 [S1, unofficial]. In some examples, it is set to 3 (with a comment suggesting 0.01) [S2, unofficial].
- `BOX_MODE`: "AABB" or "CUSTOM" [S1, unofficial].
- `RENDERING_MODE`: "A", "B", or "C" [S1, unofficial].

**Edge Cases and Pitfalls:**
- Invalid `BOX_MODE` or `RENDERING_MODE` values cause immediate program termination with an error message [S1, unofficial].
- The code visualizes a cube for *every* point in the filtered cloud, even if multiple points map to the same voxel center, resulting in overlapping cubes rather than a single representative cube per unique voxel [S2, unofficial].
- Precision issues may arise if `voxelSize` is not consistent between the filter and the visualization calculation [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def visualize_voxelization(cloud, voxel_size, box_mode, rendering_mode):
    """
    cloud: list of [x, y, z] points
    voxel_size: float
    box_mode: str ("AABB" or "CUSTOM")
    rendering_mode: str ("A", "B", or "C")
    """
    if rendering_mode not in ["A", "B", "C"]:
        raise ValueError("Invalid rendering mode")
    if box_mode not in ["AABB", "CUSTOM"]:
        raise ValueError("Invalid box mode")

    # Step 1: Filter cloud (simplified)
    # In practice, use PCL's VoxelGrid filter
    cloud_filtered = filter_voxel_grid(cloud, voxel_size)

    # Step 2: Compute voxel centers
    centers = []
    for p in cloud_filtered:
        x, y, z = p
        cx = math.floor(x / voxel_size) * voxel_size + voxel_size / 2
        cy = math.floor(y / voxel_size) * voxel_size + voxel_size / 2
        cz = math.floor(z / voxel_size) * voxel_size + voxel_size / 2
        centers.append([cx, cy, cz])

    # Step 3: Initialize viewer
    viewer = PCLVisualizer()
    viewer.set_background_color(255, 255, 255) # White

    # Step 4: Add bounding box
    if box_mode == "AABB":
        min_pt = np.min(cloud, axis=0)
        max_pt = np.max(cloud, axis=0)
        viewer.add_cube_wireframe(min_pt, max_pt)
    else: # CUSTOM
        # Assume fixed bounds for CUSTOM mode
        viewer.add_cube_wireframe([-64, -96, -64], [64, 96, 64]) # Example dimensions

    # Step 5: Render
    if rendering_mode == "A":
        # Color by Z-axis
        viewer.add_point_cloud(cloud_filtered, color_field="z")
    elif rendering_mode == "B":
        # Solid cubes
        for center in centers:
            viewer.add_cube(center, voxel_size, voxel_size, voxel_size, solid=True)
    elif rendering_mode == "C":
        # Wireframe cubes + points
        viewer.add_point_cloud(cloud_filtered, color_field="z")
        for center in centers:
            viewer.add_cube(center, voxel_size, voxel_size, voxel_size, solid=False)

    # Step 6: Spin
    while not viewer.wasStopped():
        viewer.spin_once(100)
```

## Subtopics

### [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md)
Uses multiple voxel sizes for hierarchical representation. Choose when scene complexity varies significantly.

### [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md)
Ensures reproducible voxel assignment. Choose when deterministic output is critical for training.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of points $N$ for filtering and center calculation [S1, S2, unofficial].
- **Memory Complexity:** Linear with respect to $N$ for storing filtered points and centers [S1, S2, unofficial].
- **Performance:** Visualization speed depends on the number of cubes rendered; overlapping cubes in Mode B/C can reduce performance [S2, unofficial].
- **Strengths:** Simple, intuitive visualization of voxel occupancy; supports multiple rendering styles [S1, S2, unofficial].
- **Limitations:** Does not deduplicate voxel centers, leading to redundant rendering; limited to PCL-based pipelines [S1, S2, unofficial].
- **When to use:** For debugging voxelization parameters or visualizing point cloud density [S1, S2, unofficial].
- **When not to use:** When efficient tensor generation for deep learning is required; use [Hard Voxelization](5345-hard-voxelization.md) instead.

## Background
Originating from the Point Cloud Library (PCL), this visualization method is commonly used in robotics and 3D perception pipelines [S1, S2, unofficial]. It leverages PCL's `VoxelGrid` filter for downsampling and `PCLVisualizer` for rendering [S1, S2, unofficial]. The method is often cited in unofficial repositories for educational purposes [S1, S2, unofficial].

## Sources
[S1] ericmaic/Point-Cloud-Voxelization, https://github.com/ericmaic/Point-Cloud-Voxelization, fetched 2026-10-27 (unofficial)
[S2] jsLi1/Voxelization, https://github.com/jsLi1/Voxelization, fetched 2026-10-27 (unofficial)
