# Scene Editing and Decomposition
Also called: Object Re-rendering, Object-NeRF Scene Decomposition, Scene Decomposition

## Overview
This method decomposes a composite radiance field into individual object layers and distinct background layers (near and far) to enable per-object editing and re-rendering. It differs from standard volume rendering by explicitly partitioning the ray integration domain based on 3D bounding boxes, allowing objects to be isolated from the background without requiring explicit objectness priors during training [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A trained hybrid voxel-MLP model (from [Object-NeRF](11987-object-nerf.md)), 3D bounding boxes for each object in the scene (obtained from a detector), and camera ray parameters $(\mathbf{o}, \mathbf{d})$.
*   **Outputs:** Decomposed color $\mathbf{c}$ and opacity $\alpha$ for individual objects, near background, and far background.

**Data Structures**
*   **Triplanar Feature Grid ($\mathbf{S}$):** Stores features for bilinear interpolation at query locations [S1].
*   **3D Bounding Boxes:** Axis-aligned or oriented boxes defining the spatial extent of each object $i$.

**Parameters**
*   **Negative Density Threshold:** $\sigma_{\text{neg}} = -1 \times 10^{-5}$, used to prune rays inside object bounding boxes for near-background calculation [S1].

**Steps**
1.  **Ray Partitioning:** For a given ray, determine which segments intersect the 3D bounding boxes of the objects.
2.  **Far Background Rendering:** For ray segments outside all object bounding boxes, query the standard scene MLP to retrieve color $\mathbf{c}^b_i$ and density $\sigma^b_i$. These values are unchanged from the original rendering formulation [S1].
3.  **Near Background Rendering:** To isolate the near background, prune rays inside the object bounding boxes. This is achieved by setting the density $\sigma^{nb}_i$ to a large negative value ($-1 \times 10^{-5}$) before volumetric rendering. Query the MLP for color $\mathbf{c}^{nb}_i$ and this modified density [S1].
4.  **Object Rendering:** For each object $i$, restrict sampling to the interior of its 3D bounding box. Sample the foreground MLP (using features from grid $\mathbf{S}$) to retrieve object color $\mathbf{c}^o_i$ and density $\sigma^o_i$ [S1].
5.  **Compositing:** Aggregate the contributions from far background ($b$), near background ($nb$), and objects ($o$) using alpha compositing weights $w$.

**Formula**
The composited color $\mathbf{c}$ is calculated as [S1]:
$$
\mathbf{c} = \sum_i w^b_i \mathbf{c}^b_i + \sum_i w^{nb}_i \mathbf{c}^{nb}_i + \sum_i w^o_i \mathbf{c}^o_i
$$
Where $w^b_i, w^{nb}_i, w^o_i$ are the accumulated weights for the far background, near background, and object $i$ respectively along the ray.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_decomposed(ray_origin, ray_direction, object_boxes, scene_mlp, feature_grid):
    """
    Renders a ray decomposed into far background, near background, and objects.
    """
    # 1. Calculate intersection intervals for all object boxes
    object_intervals = []
    for box in object_boxes:
        intervals = ray_box_intersection(ray_origin, ray_direction, box)
        if intervals:
            object_intervals.append((box.id, intervals))
    
    # Sort intervals by t_start
    object_intervals.sort(key=lambda x: x[1][0])
    
    # 2. Define background intervals (outside all boxes)
    background_intervals = get_complement_intervals(
        ray_origin, ray_direction, object_intervals
    )
    
    # Initialize accumulators
    color_far = [0.0, 0.0, 0.0]
    alpha_far = 0.0
    color_near = [0.0, 0.0, 0.0]
    alpha_near = 0.0
    object_colors = {} # id -> color
    object_alphas = {} # id -> alpha
    weights_far = []
    weights_near = []
    weights_objects = []
    
    current_t = ray_origin.t_min # Start of ray
    
    # Helper to accumulate color/alpha for a segment
    def accumulate_segment(t_start, t_end, query_fn):
        nonlocal current_t, color_far, alpha_far, color_near, alpha_near
        if t_end <= t_start:
            return
        
        # Sample points along segment
        num_samples = 128
        ts = linspace(t_start, t_end, num_samples)
        
        for t in ts:
            pos = ray_origin + t * ray_direction
            color, density = query_fn(pos)
            
            # Alpha compositing step
            alpha = 1.0 - exp(-density * (t_end - t_start) / num_samples) # Approximation
            weight = alpha * exp(-sum(alphas_so_far))
            
            # This is a simplified accumulation logic; standard NeRF uses cumulative alpha
            # For decomposition, we separate the contributions based on the segment type
            pass 
            
        # Note: Full implementation requires standard volumetric integration loop
        # Here we conceptualize the separation logic described in [S1]
        pass

    # 3. Render Far Background
    for interval in background_intervals:
        # Standard MLP query
        color, density = scene_mlp.query(interval.start, interval.end)
        # Accumulate to far background buffers
        accumulate_far(interval, color, density)

    # 4. Render Near Background
    # Prune by setting density to negative high value inside object boxes
    # This effectively makes the object boxes transparent/invisible to the near background pass
    for interval in background_intervals:
        # Standard MLP query
        color, density = scene_mlp.query(interval.start, interval.end)
        accumulate_near(interval, color, density)
        
    for box_id, intervals in object_intervals:
        for interval in intervals:
            # Query MLP but force density to negative value
            pos = ray_origin + interval.start * ray_direction
            color, _ = scene_mlp.query(pos)
            density = -1e-5 # Pruning value
            accumulate_near(interval, color, density)

    # 5. Render Objects
    for box_id, intervals in object_intervals:
        for interval in intervals:
            # Sample inside foreground MLP / feature grid
            pos = ray_origin + interval.start * ray_direction
            color, density = object_mlp.query(pos, feature_grid)
            accumulate_object(box_id, interval, color, density)

    # 6. Composite
    # Combine far, near, and object contributions
    final_color = composite(weights_far, color_far, weights_near, color_near, weights_objects, object_colors)
    return final_color
```

**Edge Cases and Pitfalls**
*   **No Objectness Prior:** The decomposition is purely learned from multi-view image-based rendering. No explicit objectness prior is enforced during training, which may lead to artifacts if the detector provides inaccurate bounding boxes [S1].
*   **Pruning Value:** The specific negative density value ($-1 \times 10^{-5}$) must be consistent to ensure rays are correctly pruned for the near-background pass without causing numerical instability [S1].

## Subtopics

### Object-NeRF
The parent method that provides the hybrid voxel-MLP representation and jointly optimized features required for this decomposition. [Object-NeRF](11987-object-nerf.md)

### Volumetric Rendering
The underlying integration technique used to compute the weights and colors for each segment. [Volumetric Rendering](52-volumetric-rendering.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of objects and the resolution of the feature grid. Requires additional ray-box intersection tests and separate integration passes for far background, near background, and each object.
*   **Memory Complexity:** Requires storing 3D bounding boxes for all objects. The feature grid size is determined by the parent Object-NeRF model.
*   **Performance:** On the NeRDS 360 dataset with 3-view input, NeO 360 achieves a PSNR of 19.35 and SSIM of 0.59, outperforming baselines like mipNeRF360 (PSNR 13.25, SSIM 0.31) and PixelNeRF (PSNR 13.25, SSIM 0.31) [S1].
*   **Strengths:** Enables accurate object re-rendering and editing from very few input views. Provides full control over object editability.
*   **Limitations:** Relies on accurate 3D bounding boxes from a detector. Artifacts may appear in regions with no overlap in sparse views. No explicit objectness prior ensures decomposition is data-driven, which may fail on ambiguous scenes [S1].

## Background
The method was introduced in "NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes" (ICCV 2023) [S1]. It builds upon the Object-NeRF framework to address few-shot novel view synthesis and scene decomposition tasks. The approach was evaluated on the NeRDS 360 dataset, demonstrating superior performance in both full scene synthesis and object-level decomposition compared to generalizable NeRF baselines like PixelNeRF and MVSNeRF [S1].

## Sources
[S1] NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes, https://doi.org/10.1109/iccv51070.2023.00843, fetched 2026-10-26
