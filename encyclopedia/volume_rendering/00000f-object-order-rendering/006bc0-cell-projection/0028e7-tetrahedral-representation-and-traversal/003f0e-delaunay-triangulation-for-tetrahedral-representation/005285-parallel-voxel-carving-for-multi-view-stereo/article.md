# Parallel Voxel Carving for Multi-view Stereo

Also called: GPU-based Scalable Volumetric Reconstruction for Multi-view Stereo

## Overview
Parallel Voxel Carving is a scalable volumetric reconstruction algorithm for multi-view stereo that utilizes a GPU to simultaneously execute a large number of threads, each performing voxel carving to integrate depth maps with images from multiple views [S1, unofficial]. It solves the problem of reconstructing large-scale outdoor scenes by converting semi-dense image correspondences into view-dependent depth maps and then projecting these maps into a 3D voxel grid to carve away photo-inconsistent volumes [S1, unofficial]. Unlike direct mesh integration, which struggles with triangle intersections in parallel environments, this method outputs surface voxels with normals and connectivity, making it suitable for point-based rendering and scalable to high-resolution grids via streaming [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1. A set of input images from multiple views [S1, unofficial].
2. Camera poses for each image, estimated prior to reconstruction [S1, unofficial].
3. Depth maps for each view, generated from semi-dense correspondences [S1, unofficial].

**Outputs:**
1. A set of surface voxels. Each surface voxel stores a weighted average plane (normal and distance) among several view-dependent planes intersecting the voxel cube, shifted to a point on this plane, along with an optimally interpolated color from all visible images [S1, unofficial].
2. Alternatively, the full volumetric data (carved, surface, and inner uncarved voxels) [S1, unofficial].

### Data Structures

**Depth Map Structure:**
* Stored per pixel [S1, unofficial].
* Contains depth values representing the distance between the camera viewpoint and the view-dependent surface [S1, unofficial].
* Contains plane information (normal vector) at every pixel [S1, unofficial].

**Surface Voxel Structure:**
* Represents a voxel on the object surface [S1, unofficial].
* Stores a weighted average plane derived from intersecting view-dependent planes [S1, unofficial].
* The voxel position is shifted to a point on this weighted average plane [S1, unofficial].

**Voxel Grid:**
* Configurable dimensions based on scene orientation [S1, unofficial].
* Typical example for aerial/terrain scenes: $8192 \times 512 \times 8192$ (x, y, z), where x and z are horizontal and y is vertical [S1, unofficial].

### Algorithm Steps

#### 1. Preprocessing
1. **Camera Pose Estimation and Image Matching:** Perform dense correspondence using a gradient-based warping and coarse-to-fine hierarchical matching algorithm (e.g., Cluff et al.) [S1, unofficial].
2. **Edge Extraction:** Extract matching points on or near edges using an edge detector [S1, unofficial].
3. **3D Point Computation and Refinement:** Compute 3D points from pairwise correspondences. Refine camera poses and 3D points via bundle adjustment [S1, unofficial].
4. **Terrain Alignment (Optional but recommended for aerial scenes):** Estimate a least-squares plane to align the 3D point cloud onto the x-z plane. Transform all coordinates and camera projection matrices so that x and z are horizontal and y is vertical. This allows using a smaller range in the y-axis (e.g., 512 vs 8192) for efficiency [S1, unofficial].

#### 2. Depth Map Generation
1. **Semi-Dense Correspondences:** Use semi-dense correspondences between two images and their determined 3D points. Semi-dense pairs are sparser than dense matches but sufficient to determine geometry, mitigating errors in textureless or occluded regions [S1, unofficial].
2. **Delaunay Triangulation:** Generate a depth mesh via Delaunay triangulation of the 2D correspondences [S1, unofficial].
3. **View-Dependent Surface Creation:** Replace 2D vertices in the mesh with their corresponding 3D points to create a view-dependent 3D surface [S1, unofficial].
4. **Normal Computation:** Compute each triangle's normal from its 3D vertex coordinates. Choose the sign of the normal to ensure a negative dot product with the viewing direction [S1, unofficial].
5. **Mesh Simplification (Optional):** If input correspondences are noisy, apply mesh simplification to merge triangles with similar normals [S1, unofficial].
6. **Projection:** Project the depth mesh onto the image plane to create a dense depth map. The depth map stores depth values and plane normals at every pixel [S1, unofficial].

#### 3. Parallel Voxel Carving
The volume is partitioned into sub-volumes. A parallel GPU kernel is executed for each sub-volume [S1, unofficial].

**Parallelization Strategy:**
* The GPU kernel uses $V_W \times V_D$ threads in parallel [S1, unofficial].
* Each thread performs $V_H$ voxel carving operations independently [S1, unofficial].
* For a $256^3$ voxel sub-volume, $256^2$ threads run in parallel, with each thread carving 256 voxels along the y-axis. Carving is done along the y-axis to balance workloads, as urban scenes are often flat in this dimension [S1, unofficial].

**Carving Logic (per thread):**
For each voxel in its assigned range:
1. **Visibility Check:** Determine if the voxel is visible in the current view [S1, unofficial].
2. **Projection:** Project the voxel center into the image plane of the current view [S1, unofficial].
3. **Photo-Consistency Test:** Retrieve the depth value $d_{map}$ and normal $n_{map}$ from the depth map at the projected pixel. Retrieve the depth value $d_{voxel}$ from the voxel's position relative to the camera. Check if the voxel is photo-inconsistent. A voxel is carved away if it does not match the depth map/image data. Specifically, if the voxel's depth deviates significantly from the depth map's depth, or if the photo-consistency metric fails. Incorrect view-dependent surfaces are carved away when voxel carving is performed with another viewpoint. Correct surfaces remain [S1, unofficial].
4. **Update State:** If carved: Mark voxel as "Carved." If not carved: Mark voxel as "Uncarved" (Inner) or "Surface" (if it is on the boundary of the uncarved region) [S1, unofficial].
5. **Surface Voxel Calculation (for final output):** For voxels identified as surface voxels, compute a weighted average plane among several view-dependent planes intersecting the voxel cube. Shift the voxel to a point on this plane [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_voxel_carving(depth_maps, poses, grid_dims):
    V_W, V_H, V_D = grid_dims
    # Partition volume into sub-volumes (e.g., 256^3)
    sub_volumes = partition_volume(grid_dims) # Not given in sources.
    reconstructed_voxels = []

    for sub_vol in sub_volumes:
        # GPU Kernel: 256^2 threads for 256^3 sub-volume
        # Each thread handles V_H voxels along Y-axis
        num_threads = V_W * V_D
        voxels_per_thread = V_H
        voxel_states = {}

        for thread_id in range(num_threads):
            u = thread_id % V_W
            v = (thread_id // V_W) % V_D
            weighted_normal = [0.0, 0.0, 0.0]
            weighted_dist = 0.0
            weight_sum = 0.0

            for h in range(voxels_per_thread):
                voxel_id = (u, v, h)
                voxel = get_voxel(sub_vol, voxel_id) # Not given in sources.
                is_carved = False

                for view_idx, (depth_map, pose) in enumerate(zip(depth_maps, poses)):
                    # 3.1 Visibility Check
                    if not is_visible(voxel, pose): # Not given in sources.
                        continue
                    # 3.2 Projection
                    proj_u, proj_v = project_voxel_to_image(voxel, pose) # Not given in sources.
                    # 3.3 Photo-Consistency Test
                    d_map = depth_map[proj_v][proj_u].depth
                    n_map = depth_map[proj_v][proj_u].normal
                    d_voxel = distance(voxel.center, pose.camera_origin)
                    # Check consistency: carved if depth deviates significantly or photo-consistency fails
                    if abs(d_map - d_voxel) > 1.0: # Threshold not given in sources [G]
                        voxel_states[voxel_id] = "CARVED"
                        is_carved = True
                        break
                    else:
                        w = 1.0
                        weighted_normal = [weighted_normal[k] + w * n_map[k] for k in range(3)]
                        weighted_dist += w * d_map
                        weight_sum += w

                if not is_carved:
                    voxel_states[voxel_id] = "UNCARVED"
                    # 3.4 Surface Voxel Calculation
                    if is_boundary(voxel, voxel_states): # Not given in sources.
                        voxel_states[voxel_id] = "SURFACE"
                        if weight_sum > 0:
                            avg_normal = normalize(weighted_normal)
                            avg_dist = weighted_dist / weight_sum
                            # Shift voxel to point on weighted average plane
                            voxel.center = shift_voxel_to_plane(voxel.center, avg_normal, avg_dist)
                            reconstructed_voxels.append(SurfaceVoxel(
                                position=voxel.center,
                                normal=avg_normal,
                                color=interpolate_color(voxel, depth_maps, poses) # Not given in sources.
                            ))
    return reconstructed_voxels
```

### Parameters
* **Voxel Grid Dimensions:** Configurable. Example: $8192 \times 512 \times 8192$ (x, y, z) for aerial scenes [S1, unofficial].
* **GPU Thread Configuration:** $256^2$ threads per sub-volume kernel [S1, unofficial].
* **Sub-Volume Size:** Approximately $256^3$ voxels per kernel to fit GPU memory [S1, unofficial].

### Edge Cases and Pitfalls
* **Textureless Regions:** Dense correspondence algorithms fail here. The method uses semi-dense correspondences to mitigate this [S1, unofficial].
* **Occluded Regions:** Pixels in occluded regions cause matching errors. Semi-dense correspondences help mitigate negative influences from interpolation and smoothing [S1, unofficial].
* **Camera Pose Errors:** Incorrect poses lead to poor reconstruction. The method assumes poses are estimated prior to reconstruction [S1, unofficial].
* **View-Dependent Surfaces:** Some view-dependent surfaces may be incorrect but remain if the angle between the camera ray and the object's true surface normal is favorable. These are carved away by other viewpoints [S1, unofficial].

## Subtopics

### Local Convex Hull (LCH)
* **What it is:** A method for surface reconstruction from point clouds using local convex hulls.
* **Difference:** LCH focuses on mesh generation from points, whereas Parallel Voxel Carving operates directly in voxel space using photo-consistency.
* **When to choose:** When a polygonal mesh is required directly from points without volumetric representation.
* **Link:** [Local Convex Hull (LCH)](11862-local-convex-hull-lch.md)

## Operating characteristics

* **Time Complexity:** Linear with respect to the number of voxels and views, $O(V \cdot N)$, where $V$ is the number of voxels and $N$ is the number of views. Parallelization on GPU makes it effectively linear with respect to the number of threads [S1, unofficial].
* **Memory Complexity:** Proportional to the voxel grid size and the number of depth maps. The method uses streaming to handle large volumes, limiting GPU memory usage to sub-volumes ($256^3$ voxels) [S1, unofficial].
* **Measured Performance:** Achieves high performance for large-scale reconstruction with synthetic and real urban scenes [S1, unofficial].
* **Scaling:** Scalable via partitioning the volume into sub-volumes and using streaming [S1, unofficial].
* **Strengths:**
  * Effective parallelization on GPU [S1, unofficial].
  * Handles large-scale outdoor scenes [S1, unofficial].
  * Outputs surface voxels with normals and connectivity [S1, unofficial].
  * Robust to noise via semi-dense correspondences [S1, unofficial].
* **Limitations:**
  * Requires accurate camera poses [S1, unofficial].
  * Struggles with textureless regions without semi-dense matching [S1, unofficial].
  * Volumetric representation can be memory-intensive for very large scenes without streaming [S1, unofficial].
* **When to use:** Large-scale scene reconstruction in high-resolution voxel grids, specifically for aerial photographs of outdoor scenes and real urban/outdoor scenes [S1, unofficial].
* **When not to use:** When camera poses are unknown or inaccurate, or when real-time performance on small objects is required without GPU acceleration [S1, unofficial].

## Background

* **Origin:** The method was proposed by researchers from Lawrence Livermore National Laboratory and the University of California, Davis [S1, unofficial].
* **History:** It builds on space carving techniques [11, 12] and integrates concepts from previous work [3] to combine depth maps with images for volumetric integration [S1, unofficial].
* **Usage:** Used for large-scale scene reconstruction, particularly for aerial photographs and urban scenes [S1, unofficial]. It can be integrated with existing multi-view stereo algorithms like PMVS2 to fill holes in textureless regions [S1, unofficial].

## Sources

[S1] GPU-based Scalable Volumetric Reconstruction for Multi-view Stereo, https://digital.library.unt.edu/ark:/67531/metadc831231/?q=gpu volume rendering, fetched 2026-10-28 (unofficial)
