# Scene Editing and Decomposition
Also called: Object Re-rendering, Object-NeRF Scene Decomposition, Scene Decomposition

## Overview
This method decomposes a composite radiance field into individual object layers and distinct background layers (near and far) to enable per-object editing and re-rendering [S1]. It differs from standard volume rendering by explicitly partitioning the ray integration domain based on 3D bounding boxes, allowing objects to be isolated from the background without requiring explicit objectness priors during training [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A trained hybrid voxel-MLP model (from [Object-NeRF](11987-object-nerf.md)), 3D bounding boxes for each object in the scene (obtained from a detector), and camera ray parameters $(\mathbf{o}, \mathbf{d})$ [S1].
*   **Outputs:** Decomposed color $\mathbf{c}$ and opacity $\alpha$ for individual objects, near background, and far background [S1].

**Data Structures**
*   **Triplanar Feature Grid ($\mathbf{S}$):** Stores features for bilinear interpolation at query locations [S1].
*   **3D Bounding Boxes:** Axis-aligned boxes defining the spatial extent of each object $i$ [S1].

**Parameters**
*   **Negative Density Threshold:** $\sigma_{\text{neg}} = -1 \times 10^{-5}$, used to prune rays inside object bounding boxes for near-background calculation [S1].

**Steps**
1.  **Ray Partitioning:** For a given ray, determine which segments intersect the 3D bounding boxes of the objects [S1].
2.  **Far Background Rendering:** For ray segments outside all object bounding boxes, query the standard scene MLP to retrieve color $\mathbf{c}^b_i$ and density $\sigma^b_i$ [S1]. These values are unchanged from the original rendering formulation [S1].
3.  **Near Background Rendering:** To isolate the near background, prune rays inside the object bounding boxes [S1]. This is achieved by setting the density $\sigma^{nb}_i$ to a large negative value ($-1 \times 10^{-5}$) before volumetric rendering [S1]. Query the MLP for color $\mathbf{c}^{nb}_i$ and this modified density [S1].
4.  **Object Rendering:** For each object $i$, restrict sampling to the interior of its 3D bounding box [S1]. Sample the foreground MLP (using features from grid $\mathbf{S}$) to retrieve object color $\mathbf{c}^o_i$ and density $\sigma^o_i$ [S1].
5.  **Compositing:** Aggregate the contributions from far background ($b$), near background ($nb$), and objects ($o$) using alpha compositing weights $w$ [S1].

**Formula**
The composited color $\mathbf{c}$ is calculated as [S1]:
$$
\mathbf{c} = \sum_i w^b_i \mathbf{c}^b_i + \sum_i w^{nb}_i \mathbf{c}^{nb}_i + \sum_i w^o_i \mathbf{c}^o_i
$$
Where $w^b_i, w^{nb}_i, w^o_i$ are the accumulated weights for the far background, near background, and object $i$ respectively along the ray [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_decomposed(ray_origin, ray_direction, object_boxes, scene_mlp, object_mlp, feature_grid):
    # 1. Compute ray-box intersections
    object_intervals = []
    for box in object_boxes:
        t_min = max((box.min[i] - ray_origin[i]) / ray_direction[i] if ray_direction[i] != 0 else float('inf') for i in range(3))
        t_max = min((box.max[i] - ray_origin[i]) / ray_direction[i] if ray_direction[i] != 0 else float('-inf') for i in range(3))
        t_min, t_max = max(t_min, t_max), min(t_min, t_max)
        if t_min < t_max:
            object_intervals.append((box.id, max(0.0, t_min), t_max))
    object_intervals.sort(key=lambda x: x[1])

    # 2. Compute background intervals (complement of object intervals)
    background_intervals = []
    t_curr = 0.0
    for _, t_start, t_end in object_intervals:
        if t_start > t_curr:
            background_intervals.append((t_curr, t_start))
        t_curr = max(t_curr, t_end)
    if t_curr < float('inf'):
        background_intervals.append((t_curr, float('inf')))

    # 3. Initialize accumulators
    c_far = [0.0, 0.0, 0.0]
    w_far = 0.0
    c_near = [0.0, 0.0, 0.0]
    w_near = 0.0
    c_obj = {box.id: [0.0, 0.0, 0.0] for box in object_boxes}
    w_obj = {box.id: 0.0 for box in object_boxes}

    # 4. Render Far Background
    for t_start, t_end in background_intervals:
        for t in linspace(t_start, t_end, 128):
            pos = ray_origin + t * ray_direction
            color, density = scene_mlp.query(pos)
            alpha = 1.0 - exp(-density * (t_end - t_start) / 128)
            weight = alpha * exp(-sum(w_far))
            c_far = [c_far[i] + weight * color[i] for i in range(3)]
            w_far += weight

    # 5. Render Near Background
    for t_start, t_end in background_intervals:
        for t in linspace(t_start, t_end, 128):
            pos = ray_origin + t * ray_direction
            color, density = scene_mlp.query(pos)
            alpha = 1.0 - exp(-density * (t_end - t_start) / 128)
            weight = alpha * exp(-sum(w_near))
            c_near = [c_near[i] + weight * color[i] for i in range(3)]
            w_near += weight
    for _, t_start, t_end in object_intervals:
        for t in linspace(t_start, t_end, 128):
            pos = ray_origin + t * ray_direction
            color, _ = scene_mlp.query(pos)
            density = -1e-5
            alpha = 1.0 - exp(-density * (t_end - t_start) / 128)
            weight = alpha * exp(-sum(w_near))
            c_near = [c_near[i] + weight * color[i] for i in range(3)]
            w_near += weight

    # 6. Render Objects
    for box_id, t_start, t_end in object_intervals:
        for t in linspace(t_start, t_end, 128):
            pos = ray_origin + t * ray_direction
            color, density = object_mlp.query(pos, feature_grid)
            alpha = 1.0 - exp(-density * (t_end - t_start) / 128)
            weight = alpha * exp(-sum(w_obj[box_id]))
            c_obj[box_id] = [c_obj[box_id][i] + weight * color[i] for i in range(3)]
            w_obj[box_id] += weight

    # 7. Composite
    c = [0.0, 0.0, 0.0]
    for i in range(3):
        c[i] = w_far * c_far[i] + w_near * c_near[i] + sum(w_obj[bid] * c_obj[bid][i] for bid in c_obj)
    return c
```

**Edge Cases and Pitfalls**
*   **No Objectness Prior:** The decomposition is purely learned from multi-view image-based rendering. No explicit objectness prior is enforced during training, which may lead to artifacts if the detector provides inaccurate bounding boxes [S1].
*   **Pruning Value:** The specific negative density value ($-1 \times 10^{-5}$) must be consistent to ensure rays are correctly pruned for the near-background pass without causing numerical instability [S1].

## Subtopics

### Scene Composition and Editing via Bounding Boxes
Methods that build a full 3D radiance field for each object contained within a bounding box, treating objects' radiance fields as independent entities [S2]. Render scenes efficiently by focusing only on relevant points where rays intersect with these bounding boxes (raybox intersection) [S2]. Enable image editing through manipulation of bounding boxes, allowing repositioning (rotation and translation) of objects in a scene without altering their visual appearance [S2]. For operations like removal or replication, users can adjust the scene's layout by deleting or cloning bounding boxes [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of objects and the resolution of the feature grid. Requires additional ray-box intersection tests and separate integration passes for far background, near background, and each object [G].
*   **Memory Complexity:** Requires storing 3D bounding boxes for all objects. The feature grid size is determined by the parent Object-NeRF model [G].
*   **Performance:** On the NeRDS 360 dataset with 3-view input, NeO 360 achieves a PSNR of 19.35 and SSIM of 0.59, outperforming baselines like mipNeRF360 (PSNR 13.25, SSIM 0.31) and PixelNeRF (PSNR 13.25, SSIM 0.31) [S1].
*   **Strengths:** Enables accurate object re-rendering and editing from very few input views. Provides full control over object editability [S1].
*   **Limitations:** Relies on accurate 3D bounding boxes from a detector. Artifacts may appear in regions with no overlap in sparse views [S1]. No explicit objectness prior ensures decomposition is data-driven, which may fail on ambiguous scenes [S1].

## Background
The method was introduced in "NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes" (ICCV 2023) [S1]. It builds upon the Object-NeRF framework to address few-shot novel view synthesis and scene decomposition tasks [S1]. The approach was evaluated on the NeRDS 360 dataset, demonstrating superior performance in both full scene synthesis and object-level decomposition compared to generalizable NeRF baselines like PixelNeRF and MVSNeRF [S1].

## Sources
[S1] NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes, https://doi.org/10.1109/iccv51070.2023.00843, fetched 2026-10-26
[S2] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
