# Training Pipeline Consistency

Also called: Simulated finite grid approach, Training-time block rendering

## Overview
Training Pipeline Consistency is a strategy to minimize the discrepancy between the training and inference phases of block-based volume rendering by enforcing the identical rendering pipeline during both stages [S1]. It solves the problem where training with a continuous, global coordinate system leads to artifacts or quality degradation when the model is deployed in a constrained, block-based rendering environment [S1]. It sets itself apart from standard neural rendering methods by explicitly simulating the discrete, hardware-bound constraints of the target rendering hardware (such as texture unit limits and quantization) during the optimization process [S1].

## How it works
The method ensures that the data flow, sampling logic, and integration order during training are identical to those used in the final web-based rendering stage [S1].

**Inputs and Outputs:**
*   **Input:** Camera ray $\boldsymbol{r}(t)$, scene bounding box, block partitioning parameters, proposal MLPs $\psi^k$, and attribute MLPs $f^k$ for each block $k$ [S1].
*   **Output:** Final rendered color and opacity for the ray [S1].

**Data Structures:**
*   **Blocks ($B_k$):** The scene is partitioned into spatial blocks, each with an associated proposal MLP $\psi^k$ and attribute MLP $f^k$ [S1].
*   **Ray Samples:** Points sampled along the ray $\boldsymbol{r}(t)$ between near and far boundaries [S1].

**Numbered Steps:**
1.  **Uniform Sampling:** For a given ray $\boldsymbol{r}(t)$, uniformly sample points between the near and far boundaries defined by the scene’s bounding box [S1].
2.  **Block Determination:** For each sample point, determine which block $B_k$ it resides in using an xy-coordinate projection, referred to as Eq. (4) in the source [S1].
3.  **Proposal Sampling:** Query the proposal MLP $\psi^k$ corresponding to the determined block $B_k$ to sample probability distributions along the ray [S1].
4.  **Attribute Querying:** Query the attribute MLP $f^k$ corresponding to block $B_k$ to obtain the rendering attributes for the sample points [S1].
5.  **Sequential Rendering:** Render each block sequentially to obtain the color and opacity for the ray in $B_k$ [S1].
6.  **Final Composition:** Use Eq. (5) to derive the final rendering result by combining the block contributions [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_ray_consistent(ray_origin, ray_direction, scene_bbox_min, scene_bbox_max, block_size, proposal_mlps, attribute_mlps):
    """
    Renders a ray using the same pipeline as the training stage to ensure consistency.
    
    Args:
        ray_origin: 3D array/tuple of ray origin coordinates.
        ray_direction: 3D array/tuple of ray direction vector.
        scene_bbox_min: 3D array/tuple of bounding box minimum coordinates.
        scene_bbox_max: 3D array/tuple of bounding box maximum coordinates.
        block_size: Width/height of each spatial block.
        proposal_mlps: Dictionary mapping block index to proposal MLP function.
        attribute_mlps: Dictionary mapping block index to attribute MLP function.
        
    Returns:
        final_color: Rendered color for the ray.
        final_opacity: Rendered opacity for the ray.
    """
    # Step 1: Uniform sampling between near and far boundaries
    t_min = max((scene_bbox_min[i] - ray_origin[i]) / ray_direction[i] if ray_direction[i] != 0 else float('inf') for i in range(3))
    t_max = min((scene_bbox_max[i] - ray_origin[i]) / ray_direction[i] if ray_direction[i] != 0 else float('-inf') for i in range(3))
    t_near = max(t_min, 0.0)
    t_far = t_max
    if t_near >= t_far:
        return [0.0, 0.0, 0.0], 0.0

    num_samples = 128  # [G] Typical value for volume rendering; not explicitly given in S1
    t_samples = [t_near + i * (t_far - t_near) / (num_samples - 1) for i in range(num_samples)]
    points = [[ray_origin[i] + t * ray_direction[i] for i in range(3)] for t in t_samples]

    final_color = [0.0, 0.0, 0.0]
    alpha_accum = 0.0

    # Group points by block
    block_points = {}
    for i, point in enumerate(points):
        # Step 2: Determine block Bk using xy-coordinate projection (Eq. 4)
        block_idx = get_block_index_from_xy(point[0], point[1], scene_bbox_min, scene_bbox_max, block_size)
        if block_idx not in block_points:
            block_points[block_idx] = []
        block_points[block_idx].append((i, point))

    # Process blocks in order (assuming blocks are ordered along the ray or spatially)
    # Note: The exact ordering logic for sequential block rendering is not detailed in S1 [S1]
    sorted_block_indices = sorted(block_points.keys())

    for block_idx in sorted_block_indices:
        if block_idx not in block_points:
            continue

        points_in_block = block_points[block_idx]
        proposal_mlp = proposal_mlps[block_idx]
        attribute_mlp = attribute_mlps[block_idx]

        # Step 3: Query proposal MLP to sample probability distributions
        proposal_weights = proposal_mlp(points_in_block)

        # Step 4: Query attribute MLP to obtain attributes
        attributes = attribute_mlp(points_in_block)
        colors = [attr[:3] for attr in attributes]
        densities = [attr[3] for attr in attributes]

        # Step 5: Render block sequentially
        for idx_in_block, (global_i, point) in enumerate(points_in_block):
            # Calculate delta distance
            if idx_in_block == 0:
                delta_t = t_samples[points_in_block[1][0]] - t_samples[global_i] if len(points_in_block) > 1 else 0.0
            elif idx_in_block == len(points_in_block) - 1:
                delta_t = t_samples[global_i] - t_samples[points_in_block[-2][0]]
            else:
                delta_t = (t_samples[points_in_block[idx_in_block + 1][0]] - t_samples[points_in_block[idx_in_block - 1][0]]) / 2.0

            # Convert density to opacity
            alpha = 1.0 - math.exp(-densities[idx_in_block] * delta_t)

            # Compositing
            weight = alpha * (1.0 - alpha_accum)
            final_color[0] += weight * colors[idx_in_block][0]
            final_color[1] += weight * colors[idx_in_block][1]
            final_color[2] += weight * colors[idx_in_block][2]
            alpha_accum += weight

            if alpha_accum >= 1.0:
                break

    return final_color, alpha_accum

def get_block_index_from_xy(x, y, bbox_min, bbox_max, block_size):
    # [G] Assuming a simple grid-based partitioning based on xy-coordinate projection
    # as described in the source for Eq. (4).
    width = bbox_max[0] - bbox_min[0]
    height = bbox_max[1] - bbox_min[1]
    cols = int(width / block_size)
    rows = int(height / block_size)
    col = int((x - bbox_min[0]) / block_size)
    row = int((y - bbox_min[1]) / block_size)
    return row * cols + col
```

**Parameters:**
*   **Number of Samples:** Not given in the sources [S1].
*   **Loss Weights:** The training loss includes $\lambda_1 \mathcal{L}_{\text{s3im}}$, $\lambda_2 \mathcal{L}_{\text{prop}}$, $\lambda_3 \mathcal{L}_{\text{dist}}$, $\lambda_4 \mathcal{L}_{\text{s}}$, and $\lambda_5 \mathcal{L}_{\text{opacity}}$ [S1]. Specific values for $\lambda_i$ are not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on simulating quantization during training to mimic discrete texture lookups [S1]. Failure to accurately simulate this can lead to a gap between training and rendering [S1].
*   **Boundaries:** The method assumes a well-defined scene bounding box for uniform sampling [S1].
*   **Block Assignment:** The block determination (Eq. 4) must be consistent between training and rendering [S1].

## Subtopics

### MERF Rendering Pipeline
A memory-optimized rendering pipeline that combines a low-resolution sparse voxel representation with high-resolution 2D triplanes, bridging training and rendering by simulating grid and quantization constraints during training [S1]. It reduces memory requirements compared to SNeRG by replacing the dense 3D voxel grid with a hybrid sparse voxel and 2D triplane representation [S1]. During training, it simulates a finite grid approach by querying MLPs at virtual grid corners instead of continuous coordinates, and simulates quantization to closely mimic the discrete texture lookup behavior of the actual rendering pipeline [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** The MERF variant reduces memory requirements compared to SNeRG by using a hybrid sparse voxel and 2D triplane representation [S1].
*   **Measured Performance:** Not given in the sources [S1].
*   **Scaling:** Not given in the sources [S1].
*   **Strengths:** Minimizes the gap between training and rendering quality by ensuring pipeline consistency [S1]. The MERF variant offers significant memory reduction [S1].
*   **Limitations:** The block-based approach is constrained by web texture unit limits, requiring sequential rendering [Parent].
*   **When to use:** When deploying neural radiance fields on web platforms with strict hardware constraints (texture unit limits) and where training-inference consistency is critical for quality [S1, Parent].

## Background
The method originates from the "City-on-Web" project, which focuses on real-time neural rendering of large-scale scenes on the web [S1]. It was developed to address the limitations of existing methods like SNeRG, which demand substantial memory [S1]. The approach was introduced in the paper "City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web" [S1].

## Sources
[S1] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
