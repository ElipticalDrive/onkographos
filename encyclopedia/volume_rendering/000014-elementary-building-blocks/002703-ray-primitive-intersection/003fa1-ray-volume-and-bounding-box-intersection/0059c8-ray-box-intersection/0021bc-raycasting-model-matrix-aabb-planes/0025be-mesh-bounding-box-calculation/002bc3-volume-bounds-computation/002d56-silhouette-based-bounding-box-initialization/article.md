# Silhouette-based Bounding Box Initialization


## Overview
Silhouette-based Bounding Box Initialization computes an Axis-Aligned Bounding Box (AABB) that tightly encloses the visible geometry of a scene by projecting ground truth silhouettes from training cameras onto the scene space [S1, unofficial]. It serves as a robust initialization step for volume rendering, particularly when the scene is dense or when precise spatial bounds are required before adaptive learning or sparse voxel grid construction begins [S1, unofficial]. Unlike simple extent-based methods, it accounts for camera visibility and occlusion, ensuring the bounding volume only encompasses regions actually observed by the training set [S1, unofficial].

## How it works
The method determines the bounding box by analyzing the intersection of camera frustums with ground truth silhouette masks. It supports two primary modes: direct initialization for dense scenes and temporary-scene initialization for sparse scenes or when specific padding constraints are needed.

### Inputs and Outputs
**Inputs:**
*   `train_cameras`: A list of training camera poses and intrinsic parameters.
*   `groundTruth`: Ground truth image masks or silhouettes corresponding to `train_cameras`.
*   `scene`: The target scene object to be initialized with bounding box coordinates.
*   `datasetSize`: The number of training views.
*   `silhouetteRadius`: A radius parameter used to expand the silhouette projection, accounting for boundary uncertainty [S1, unofficial].
*   `is_sparse`: A boolean flag indicating if the scene representation is sparse (e.g., 3D Gaussian Splatting or sparse voxels) [S1, unofficial].
*   `skipIndex`, `skips`: Parameters controlling which views are skipped during initialization [S1, unofficial].
*   `configs->silhouetteInitial`: A configuration flag enabling this initialization method [S1, unofficial].

**Outputs:**
*   `scene.bbox_depth_max/min`, `scene.bbox_height_max/min`, `scene.bbox_width_max/min`: The computed AABB limits [S1, unofficial].
*   `scene.adaptiveLRDevice`: Adaptive learning rate parameters copied from the initialization process [S1, unofficial].

### Data Structures
The implementation relies on a `Camera` structure and a `Scene` structure.
*   `Camera`: Contains projection matrix components (`cameraKernel[9]`), translation (`translateMat[3]`), and image dimensions (`targetImageSize[2]`) [S1, unofficial].
*   `Scene`: Contains bounding box limits (`bbox_depth_max/min`, `bbox_height_max/min`, `bbox_width_max/min`) and device memory for learning rates (`adaptiveLRDevice`) [S1, unofficial].

### Algorithm Steps
The core logic branches based on the sparsity of the scene and the configuration flag.

1.  **Check Configuration**: If `configs->silhouetteInitial` is false, the method may fall back to a default initialization or skip silhouette-based logic, though the notes indicate a temporary scene is still created in some fallback paths [S1, unofficial].
2.  **Determine Mode**:
    *   **Dense Mode**: If `configs->silhouetteInitial` is true and the scene is **not** sparse, call `RenderUtils::bbox_silhouette` directly with the original `scene` [S1, unofficial].
    *   **Sparse/Temporary Mode**: If `configs->silhouetteInitial` is true and the scene **is** sparse, or if the configuration requires it, create a `tempScene` [S1, unofficial].
3.  **Temporary Scene Initialization (if applicable)**:
    *   Initialize `tempScene` with dimensions `sceneWidth`, `sceneHeight`, `sceneDepth`.
    *   Set flags for sparse and other internal states to `true`.
    *   Set padding/constraint parameters: `0.1`, `-10.`, `0.5`, `1.` [S1, unofficial].
4.  **Compute Bounding Box**:
    *   Call `RenderUtils::bbox_silhouette(train_cameras, groundTruth, scene_or_tempScene, viewCount, silhouetteRadius, is_sparse, skipIndex, skips)` [S1, unofficial].
    *   This function projects the ground truth silhouettes from each camera, expands them by `silhouetteRadius`, and computes the union of all projected volumes to determine the AABB [S1, unofficial].
5.  **Finalize (if temporary scene was used)**:
    *   Copy `bbox_depth_max/min`, `bbox_height_max/min`, `bbox_width_max/min` from `tempScene` to `scene` [S1, unofficial].
    *   Copy `adaptiveLRDevice` from `tempScene` to `scene` using `cudaMemcpy` [S1, unofficial].
    *   Free `tempScene` resources [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_based_bbox_initialization(train_cameras, ground_truth, scene, 
                                         dataset_size, silhouette_radius, 
                                         is_sparse, skip_index, skips, 
                                         configs):
    """
    Initializes the scene's bounding box using silhouette information.
    
    Args:
        train_cameras: List of camera objects with projection matrices and intrinsics.
        ground_truth: List of ground truth silhouette masks.
        scene: The scene object to update with bounding box limits.
        dataset_size: Number of training views.
        silhouette_radius: Radius to expand silhouette projections.
        is_sparse: Boolean, true if the scene representation is sparse.
        skip_index: Index of the view to skip (if any).
        skips: List of indices to skip.
        configs: Configuration object containing silhouetteInitial flag.
    """
    if not configs.silhouette_initial:
        # Fallback or alternative initialization path as per source logic
        # Note: Source [S1, unofficial] shows a tempScene creation even in some else-if branches
        # We follow the explicit sparse/non-sparse branching for the primary path.
        pass

    if configs.silhouette_initial:
        if not is_sparse:
            # Dense scene: Direct initialization
            RenderUtils.bbox_silhouette(
                train_cameras=train_cameras,
                ground_truth=ground_truth,
                scene=scene,
                dataset_size=dataset_size,
                silhouette_radius=silhouette_radius,
                is_sparse=False,
                skip_index=skip_index,
                skips=skips
            )
        else:
            # Sparse scene: Use temporary scene to handle padding/constraints
            temp_scene = Scene()
            # Initialize tempScene with specific parameters
            # Parameters: width, height, depth, sparse_flag, other_flags, 
            # param1, param2, param3, param4
            temp_scene.init_self(
                scene_width=scene.width,
                scene_height=scene.height,
                scene_depth=scene.depth,
                flag1=True,
                flag2=True,
                param1=0.1,
                param2=-10.0,
                param3=0.5,
                param4=1.0
            )
            
            RenderUtils.bbox_silhouette(
                train_cameras=train_cameras,
                ground_truth=ground_truth,
                scene=temp_scene,
                dataset_size=dataset_size,
                silhouette_radius=silhouette_radius,
                is_sparse=True,
                skip_index=skip_index,
                skips=skips
            )
            
            # Copy results back to main scene
            scene.bbox_depth_max = temp_scene.bbox_depth_max
            scene.bbox_depth_min = temp_scene.bbox_depth_min
            scene.bbox_height_max = temp_scene.bbox_height_max
            scene.bbox_height_min = temp_scene.bbox_height_min
            scene.bbox_width_max = temp_scene.bbox_width_max
            scene.bbox_width_min = temp_scene.bbox_width_min
            
            # Copy adaptive learning rate from device memory
            scene.adaptive_lr_device = cuda_memcpy(temp_scene.adaptive_lr_device)
            
            # Cleanup
            temp_scene.free_self()
```

### Parameters
*   `silhouetteRadius`: Controls the expansion of the projected silhouette. A larger radius creates a more conservative (larger) bounding box [S1, unofficial].
*   `skipIndex`: Specific view index to exclude from the bounding box calculation [S1, unofficial].
*   `skips`: List of view indices to exclude [S1, unofficial].
*   `configs->silhouetteInitial`: Boolean flag to enable this initialization method [S1, unofficial].

### Edge Cases and Pitfalls
*   **Sparse Scene Handling**: Direct initialization on sparse scenes may fail or produce incorrect bounds due to lack of dense voxel occupancy. The temporary scene approach with specific padding parameters (`0.1`, `-10.`, etc.) is required [S1, unofficial].
*   **Camera Calibration**: The method assumes camera projection matrices are correctly aligned with the ground truth silhouettes. Miscalibration leads to incorrect bounding box estimates [S1, unofficial].
*   **Memory Overflow**: In large-scale scenes, computing bounding boxes for all cameras simultaneously may exceed GPU memory. This is addressed by sub-scene decomposition (see Subtopics) [S2].

## Subtopics

### Sub-scene Spatial Decomposition and Bounding Box Computation
A spatial decomposition methodology that assigns cameras to sub-scenes to facilitate precise resource allocation and spatial indexing for large-scale scene reconstruction [S2]. It computes the initial bounding box for a sub-scene as the union of the individual camera projection bounding boxes associated with that sub-scene, then refines the region by computing the minimum bounding rectangle that encompasses the entire scene range observed by all cameras in the sub-scene [S2]. This resulting rectangle is used as the scene bounding box for the sub-scene [S2]. It uses Instant-NGP as the baseline model for training sub-scene models, chosen for its speed to meet near-real-time requirements for large-scale reconstruction [S2]. Default parameters include `MaxNum` (maximum cameras per sub-scene) of 113 to prevent GPU memory overflow on 24GB GPUs, `TopNum` (initial maximum cameras per cluster) of 90, and `Training iterations` of $5 \times 10^3$ [S2].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of training cameras and resolution of ground truth silhouettes. Sub-scene decomposition reduces per-step complexity by partitioning the problem [S2].
*   **Memory Complexity**: Proportional to the number of cameras and the size of the temporary scene structure. Sub-scene decomposition limits peak memory usage by restricting the number of cameras processed simultaneously (e.g., MaxNum=113) [S2].
*   **Strengths**: Provides tight, visibility-aware bounding boxes that improve convergence and reduce unnecessary computation in empty space [S1, unofficial]. Handles sparse and dense representations differently to ensure robustness [S1, unofficial].
*   **Limitations**: Requires ground truth silhouettes, which may not be available in all datasets. Computationally expensive for very large scenes without decomposition [S1, unofficial][S2].
*   **When to Use**: When initializing volume rendering for scenes with known camera poses and ground truth masks, especially in large-scale or sparse scenarios [S1, unofficial][S2].

## Background
Silhouette-based initialization is used in neural radiance field (NeRF) and 3D Gaussian Splatting pipelines to establish initial scene bounds. It was notably employed in Clear-plenoxels for initializing bounding boxes and adaptive learning rates [S1, unofficial]. The sub-scene decomposition variant was introduced in BirdNeRF for fast neural reconstruction of large-scale scenes from aerial imagery, addressing GPU memory constraints and enabling near-real-time reconstruction [S2].

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S2] BirdNeRF: fast neural reconstruction of large-scale scenes from aerial imagery., https://doi.org/10.1038/s41598-025-21206-z, fetched 2026-10-26
