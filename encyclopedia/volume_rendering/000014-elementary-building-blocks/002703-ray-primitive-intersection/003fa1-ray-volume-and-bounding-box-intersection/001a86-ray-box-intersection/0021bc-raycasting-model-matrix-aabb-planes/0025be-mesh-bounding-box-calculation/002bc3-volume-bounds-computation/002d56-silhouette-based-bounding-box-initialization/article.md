# Silhouette-based Bounding Box Initialization


## Overview
Silhouette-based Bounding Box Initialization computes an Axis-Aligned Bounding Box (AABB) and initializes the adaptive learning rate using silhouette information from training cameras and ground truth masks [S1, unofficial]. It serves as the entry point for volume bounds computation when explicit scene geometry is unavailable [S1, unofficial].

## How it works
**Inputs:** `train_cameras` (array of camera extrinsics and intrinsics), `groundTruth` (array of binary silhouette masks), `scene` (target or temporary scene object), `datasetSize`/`viewCount` (number of training views), `silhouetteRadius` (expansion parameter), `skipIndex` and `skips` (view subsampling indices), `silhouetteInitial` (control flag) [S1, unofficial].
**Outputs:** Updated `scene` AABB fields (`bbox_depth_max/min`, `bbox_height_max/min`, `bbox_width_max/min`) and `adaptiveLRDevice` [S1, unofficial].

**Data Structures:** The scene object stores AABB bounds and an adaptive learning rate device pointer. For sparse or disabled initialization, a temporary scene `tempScene` is allocated and initialized via `initSelf(sceneWidth, sceneHeight, sceneDepth, true, true, 0.1, -10., 0.5, 1)` [S1, unofficial].

**Algorithm Steps:**
1. Evaluate `configs->silhouetteInitial`.
2. If true and the scene is not sparse, invoke `bbox_silhouette` directly on the target `scene` [S1, unofficial].
3. If true and the scene is sparse, or if `silhouetteInitial` is false, allocate `tempScene` and call `initSelf` with standard spatial and structural parameters [S1, unofficial].
4. Call `bbox_silhouette` with `train_cameras`, `groundTruth`, `&tempScene`, `viewCount`, `silhouetteRadius`, `true`, `skipIndex`, `skips` [S1, unofficial].
5. Copy `bbox_depth_max/min`, `bbox_height_max/min`, `bbox_width_max/min` from `tempScene` to the target `scene` [S1, unofficial].
6. If the sparse path was taken, copy `adaptiveLRDevice` from `tempScene` to `scene` using `cudaMemcpy`, then call `tempScene.freeSelf()` [S1, unofficial].

**Formulas:**
- Ray direction in world space: $\mathbf{d} = \mathbf{R} \cdot \mathbf{K}^{-1} \cdot [u, v, 1]^\top$, normalized [G].
- AABB update: $\mathbf{b}_{\min} = \min(\mathbf{b}_{\min}, \mathbf{p})$, $\mathbf{b}_{\max} = \max(\mathbf{b}_{\max}, \mathbf{p})$ for all intersection points $\mathbf{p}$ [G].
- Final expansion: $\mathbf{b}_{\min} \leftarrow \mathbf{b}_{\min} - \text{silhouetteRadius}$, $\mathbf{b}_{\max} \leftarrow \mathbf{b}_{\max} + \text{silhouetteRadius}$ [S1, unofficial].

**Parameters:**
- `silhouetteRadius`: Radius parameter passed to `bbox_silhouette` for expanding bounds [S1, unofficial].
- `skipIndex`, `skips`: Parameters for subsampling views during initialization [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Sparse Scene**: Requires temporary allocation and CUDA memory copy for the learning rate pointer to avoid corrupting the final scene's internal state [S1, unofficial].
- **No Foreground**: If no training views contain foreground pixels, bounds remain at initial extremes; a fallback to scene dimensions is required [G].
- **Precision**: Use double precision for ray generation and intersection to avoid jitter in bounds updates [G].
- **Silhouette Radius**: A radius of zero yields tight bounds but may exclude geometry if masks are noisy or undersampled [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bbox_silhouette_wrapper(train_cameras, ground_truth, scene, dataset_size, 
                            silhouette_radius, skip_index, skips, silhouette_initial, is_sparse):
    if silhouette_initial and not is_sparse:
        bbox_silhouette_core(train_cameras, ground_truth, scene, dataset_size, 
                             silhouette_radius, skip_index, skips)
    else:
        temp_scene = Scene()
        temp_scene.init_self(scene_width, scene_height, scene_depth, True, True, 0.1, -10.0, 0.5, 1.0)
        bbox_silhouette_core(train_cameras, ground_truth, temp_scene, dataset_size, 
                             silhouette_radius, skip_index, skips)
        
        scene.bbox_depth_min = temp_scene.bbox_depth_min
        scene.bbox_depth_max = temp_scene.bbox_depth_max
        scene.bbox_height_min = temp_scene.bbox_height_min
        scene.bbox_height_max = temp_scene.bbox_height_max
        scene.bbox_width_min = temp_scene.bbox_width_min
        scene.bbox_width_max = temp_scene.bbox_width_max
        
        if is_sparse:
            scene.adaptive_lr_device = cuda_memcpy(temp_scene.adaptive_lr_device)
        temp_scene.free_self()

def bbox_silhouette_core(cameras, masks, scene, dataset_size, radius, skip_idx, skips):
    b_min = [float('inf'), float('inf'), float('inf')]
    b_max = [float('-inf'), float('-inf'), float('-inf')]
    
    for i in range(dataset_size):
        if i % skips == skip_idx:
            cam = cameras[i]
            mask = masks[i]
            K_inv = inverse_3x3(cam.intrinsics)
            R = cam.rotation_matrix
            C = cam.center
            
            for u, v in mask.foreground_pixels():
                ray_dir_cam = K_inv @ [u, v, 1.0]
                ray_dir_world = R @ ray_dir_cam
                ray_dir_world = ray_dir_world / norm(ray_dir_world)
                
                t_min, t_max = ray_box_intersect(C, ray_dir_world, b_min, b_max)
                
                if t_max > 0:
                    p_far = C + ray_dir_world * t_max
                    b_min = [min(b_min[j], p_far[j]) for j in range(3)]
                    b_max = [max(b_max[j], p_far[j]) for j in range(3)]
                    if t_min > 0:
                        p_near = C + ray_dir_world * t_min
                        b_min = [min(b_min[j], p_near[j]) for j in range(3)]
                        b_max = [max(b_max[j], p_near[j]) for j in range(3)]
    
    b_min = [b_min[j] - radius for j in range(3)]
    b_max = [b_max[j] + radius for j in range(3)]
    
    scene.bbox_depth_min = b_min[0]
    scene.bbox_depth_max = b_max[0]
    scene.bbox_height_min = b_min[1]
    scene.bbox_height_max = b_max[1]
    scene.bbox_width_min = b_min[2]
    scene.bbox_width_max = b_max[2]

def ray_box_intersect(origin, direction, b_min, b_max):
    t_min = float('-inf')
    t_max = float('inf')
    for i in range(3):
        if abs(direction[i]) > 1e-8:
            t1 = (b_min[i] - origin[i]) / direction[i]
            t2 = (b_max[i] - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
        else:
            if origin[i] < b_min[i] or origin[i] > b_max[i]:
                return float('-inf'), float('-inf')
    return t_min, t_max
```

## Operating characteristics
- **Time Complexity**: Proportional to the number of foreground pixels across the sampled training views [S1, unofficial].
- **Memory Complexity**: $O(1)$ for the bounding box itself; $O(N)$ for temporary scene allocation when `is_sparse` is true [S1, unofficial].
- **Strengths**: Tightly fits visible geometry using ground truth silhouettes; safely handles sparse representations via temporary allocation [S1, unofficial].
- **Limitations**: Requires ground truth silhouette masks; performance scales with mask resolution and view count; depends on `silhouetteRadius` tuning [S1, unofficial].
- **When to Use**: Initializing spatial bounds and adaptive learning rates for volume rendering when explicit point clouds are unavailable [S1, unofficial].
- **When Not to Use**: When ground truth masks are missing or when scene bounds can be derived directly from dense geometry [S1, unofficial].

## Background
This method originates from the Clear-Plenoxels repository [S1, unofficial]. It is used in volume rendering pipelines to initialize the spatial extent of the scene and the adaptive learning rate before training begins, leveraging silhouette constraints to avoid arbitrary bounding volumes.

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
