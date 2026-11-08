# Silhouette-based Bounding Box Initialization


## Overview
Silhouette-based Bounding Box Initialization computes an Axis-Aligned Bounding Box (AABB) that encloses the visible geometry of a scene by projecting ground truth masks from training cameras. It serves as a robust initialization step for volume rendering, particularly when sparse representations are used, ensuring the bounding volume tightly fits the object of interest rather than the entire scene space. It differs from simple min-max point accumulation by using binary silhouette masks to determine visibility, making it resilient to background noise and sparse sampling artifacts.

## How it works
The method is implemented as `RenderUtils::bbox_silhouette`. It takes training camera poses and corresponding ground truth silhouette masks to determine the spatial extent of the scene.

**Inputs:**
- `train_cameras`: Array of camera extrinsics and intrinsics.
- `groundTruth`: Array of binary silhouette masks (1 for foreground, 0 for background) corresponding to `train_cameras`.
- `scene`: A temporary or target scene object containing the AABB fields (`bbox_depth_max/min`, `bbox_height_max/min`, `bbox_width_max/min`).
- `datasetSize`: Number of training views.
- `silhouetteRadius`: Radius parameter for expanding the silhouette projection (used to account for depth uncertainty or anti-aliasing).
- `skipIndex`, `skips`: Parameters for subsampling views during initialization.

**Data Structures:**
The `scene` object contains AABB bounds. In the sparse case, a `tempScene` is initialized with specific parameters before calling the core function. The initialization of `tempScene` uses `initSelf` with arguments: `sceneWidth`, `sceneHeight`, `sceneDepth`, `true`, `true`, `0.1`, `-10.`, `0.5`, `1`. These likely correspond to spatial dimensions, flags for sparse/adaptive structures, and initial scale/offset parameters for the voxel grid or octree.

**Algorithm Steps:**
1. **Preprocessing**: If the scene is sparse, create a `tempScene` and initialize it with standard parameters. This temporary structure is used to compute the bounds safely before copying them to the final scene.
2. **Projection**: For each training camera $i$ in `train_cameras`:
   a. Retrieve the ground truth silhouette mask $M_i$.
   b. Project the 2D silhouette pixels into 3D space using the camera's projection matrix $P_i$.
   c. For each foreground pixel $(u, v)$ in $M_i$, compute the ray direction $d_i$ from the camera center.
   d. Intersect this ray with the current AABB bounds. If the ray passes through the volume, update the AABB to include the intersection points.
   e. Expand the bounds by `silhouetteRadius` to account for depth uncertainty.
3. **Aggregation**: The AABB is updated incrementally. The final bounds are the union of all rays passing through foreground pixels across all training views.
4. **Post-processing**: If a `tempScene` was used, copy its `bbox_*` fields to the final `scene`. Copy `adaptiveLRDevice` if applicable. Free `tempScene`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bbox_silhouette(train_cameras, ground_truth, scene, dataset_size, silhouette_radius, is_sparse, skip_index, skips):
    # Initialize bounds to infinity/negative infinity
    bbox_min = [inf, inf, inf]
    bbox_max = [-inf, -inf, -inf]
    
    # Determine which cameras to use based on skip parameters
    cameras_to_use = []
    for i in range(dataset_size):
        if i % skips == skip_index or skips == 1:
            cameras_to_use.append(i)
            
    for idx in cameras_to_use:
        camera = train_cameras[idx]
        mask = ground_truth[idx]
        
        # Get camera projection matrix and intrinsics
        P = camera.projection_matrix
        K = camera.intrinsics
        C = camera.center
        
        # Iterate over foreground pixels in the mask
        for u, v in mask.foreground_pixels():
            # Back-project pixel to 3D ray
            # x = K^-1 * [u, v, 1]
            ray_dir_cam = inv(K) @ [u, v, 1]
            ray_dir_world = camera.rotation @ ray_dir_cam
            ray_dir_world = normalize(ray_dir_world)
            
            # Intersect ray with current bounding box
            # Compute t_min and t_max for ray-box intersection
            t_min, t_max = ray_box_intersect(C, ray_dir_world, bbox_min, bbox_max)
            
            # If ray intersects, update bounds with intersection points
            if t_max > 0:
                # Point at t_max is further out, expand bounds
                point_far = C + ray_dir_world * t_max
                bbox_min = min(bbox_min, point_far)
                bbox_max = max(bbox_max, point_far)
                
                # Also consider point at t_min if it's outside current bounds
                if t_min > 0:
                    point_near = C + ray_dir_world * t_min
                    bbox_min = min(bbox_min, point_near)
                    bbox_max = max(bbox_max, point_near)
                    
    # Expand bounds by silhouette radius
    bbox_min = bbox_min - silhouette_radius
    bbox_max = bbox_max + silhouette_radius
    
    # Update scene bounds
    scene.bbox_min = bbox_min
    scene.bbox_max = bbox_max
```

**Edge Cases and Pitfalls:**
- **Sparse Scene**: If the scene is sparse, a temporary scene must be used to avoid corrupting the final scene's internal state during initialization. The bounds are copied after computation.
- **No Foreground**: If no training views have foreground pixels, the bounds remain at initial values (infinity). This should be handled by a fallback to scene dimensions.
- **Precision**: Use double precision for intermediate calculations to avoid jitter in bounds updates.
- **Silhouette Radius**: A radius of 0 may result in tight but potentially incomplete bounds if the silhouette masks are noisy or undersampled.

## Operating characteristics
- **Time Complexity**: $O(N \cdot P)$, where $N$ is the number of training views and $P$ is the average number of foreground pixels per view.
- **Memory Complexity**: $O(N \cdot P)$ to store the silhouette masks. The bounding box itself is $O(1)$.
- **Strengths**: Robust to background noise; works well with sparse data; provides a tight fit for the object of interest.
- **Limitations**: Requires ground truth silhouettes; sensitive to the `silhouetteRadius` parameter; may be slow for very high-resolution masks.
- **When to Use**: When initializing bounding boxes for novel view synthesis or NeRF-like methods where the object extent is unknown or sparse.
- **When Not to Use**: When ground truth silhouettes are unavailable; when the scene is dense and the bounding box can be computed from point clouds.

## Background
This method is derived from the Clear-Plenoxels repository [S1, unofficial]. It is used in the context of 3D Gaussian Splatting and Plenoxels to initialize the spatial extent of the scene. The approach leverages the fact that silhouettes provide a strong constraint on the visible volume, allowing for efficient initialization without requiring dense point clouds.

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
