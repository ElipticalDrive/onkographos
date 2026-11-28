# DM-NeRF


## Overview
DM-NeRF is a method that simultaneously reconstructs continuous 3D scene geometry, decomposes all individual objects in 3D space, and supports flexible object shape manipulation (translation, rotation, size adjustment, deformation) from 2D images only [S1, unofficial]. It sets itself apart from standard NeRF by introducing an explicit "object field" that predicts a one-hot object code for every 3D point, enabling instance-level decomposition without 3D ground truth, and an "object manipulator" that handles occlusions during editing [S1, unofficial]. Unlike [Fig-NeRF](12006-fig-nerf.md) which focuses on image-guided generation, or [Unified Decompositional and Compositional NeRF](14413-unified-decompositional-and-compositional-nerf.md) which uses unified latent codes, DM-NeRF explicitly learns per-point object assignments via a dedicated MLP branch supervised by 2D masks [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $L$ images for a static scene with known camera poses and intrinsics, denoted as $\{(I_1, \xi_1, K_1) \cdots (I_L, \xi_L, K_L)\}$ [S1, unofficial].
*   **Input:** 2D object masks for the $L$ images, denoted as $\{\bar{I}_1 \cdots \bar{I}_L\}$, where $\bar{I}_l \in \mathbb{R}^{U \times V \times T}$ [S1, unofficial]. $U, V$ are spatial width and height; $T$ is the number of ground truth objects [S1, unofficial].
*   **Output:** Volume density $\sigma$, object code $\mathbf{o}$, and color $\mathbf{c}$ for any query 3D point $\mathbf{p}$ and viewing direction $\mathbf{v}$ [S1, unofficial].

**Data Structures**
*   **Object Code $\mathbf{o}$:** A one-hot vector of size $H+1$, where $H$ is the predefined number of object channels [S1, unofficial]. The first $H$ channels correspond to specific objects, and the $(H+1)$-th channel corresponds to empty/background space [S1, unofficial].
*   **Network Architecture:** A single MLP with the following layer dimensions:
    *   Input: Positional Encoded location + Positional Encoded direction.
    *   Hidden Layers: 128 $\to$ 256 $\to$ 256 $\to$ 256 $\to$ 256 $\to$ 256 $\to$ 256 $\to$ 256 $\to$ 256 $\to$ 256 $\to$ 128 [S1, unofficial].
    *   Output Heads:
        *   Density $\sigma \in \mathbb{R}^1$ (function of location only) [S1, unofficial].
        *   Object Code $\mathbf{o} \in \mathbb{R}^{H+1}$ (function of location only) [S1, unofficial].
        *   Color $\mathbf{c} \in \mathbb{R}^3$ (function of location and viewing direction) [S1, unofficial].

**Algorithm Steps**

1.  **Positional Encoding:** Compute positional encoding $\text{PE}(\mathbf{p})$ for 3D location $\mathbf{p}=(x,y,z)$ and $\text{PE}(\mathbf{v})$ for viewing direction $\mathbf{v}=(\theta, \phi)$ [S1, unofficial].
2.  **Forward Pass:** Feed $\text{PE}(\mathbf{p})$ and $\text{PE}(\mathbf{v})$ into the MLP.
    *   The network outputs raw density $\sigma$, raw object code logits $\mathbf{o}_{raw}$, and color $\mathbf{c}$ [S1, unofficial].
    *   Apply sigmoid to $\sigma_{raw}$ to get $\sigma \in [0, 1]$ [G] (standard NeRF practice, implied by "volume density").
    *   Apply softmax to $\mathbf{o}_{raw}$ to get the object code probability distribution $\mathbf{o} \in \mathbb{R}^{H+1}$ [G].
3.  **Volume Rendering:** Render the 2D image using standard volumetric rendering equation:
    $$ C(\mathbf{r}) = \sum_{i=1}^N T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) \mathbf{c}_i $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, $\delta_i$ is the distance between samples, and $\mathbf{r}$ is the camera ray [S1, unofficial] (standard NeRF rendering, assumed from "leveraging vanilla NeRF as backbone").
4.  **2D Object Matching (Supervision):**
    *   Render 2D object prediction images $\{I_1 \cdots I_L\}$ where $I_l \in \mathbb{R}^{U \times V \times (H+1)}$ [S1, unofficial].
    *   Reshape predicted object channels $M \in \mathbb{R}^{N \times H}$ and ground truth labels $\bar{M} \in \mathbb{R}^{N \times T}$, where $N = U \times V$ [S1, unofficial].
    *   Compute cost matrices using Soft Intersection-over-Union (sIoU) and Cross-Entropy Score (CES) [S1, unofficial].
    *   Use the Hungarian algorithm to associate each ground truth 2D object mask with a unique predicted 2D object mask [S1, unofficial].
5.  **Loss Computation:**
    *   Compute standard photometric loss between rendered color and ground truth color [G].
    *   Compute object supervision losses $\ell_{3d}^{empty}$ and $\ell_{3d}^{obj}$ based on the matched 2D masks and the rendered object codes [S1, unofficial].
6.  **Optimization:** Minimize the total loss to update MLP weights [S1, unofficial].
7.  **Object Manipulation:**
    *   To edit an object, identify the 3D points belonging to the target object code [S1, unofficial].
    *   Apply transformation (translation, rotation, scale, deformation) to these points [S1, unofficial].
    *   **Tackle Visual Occlusions:** Use a lightweight inverse query algorithm to handle occlusions and collisions without evaluating dense 3D points explicitly [S1, unofficial].
        *   *Note:* The specific steps of the inverse query algorithm are not detailed in the provided text, only that it is "lightweight" and "automatically edits the scene geometry" [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.optimize import linear_sum_assignment

def positional_encoding(x, min_deg=0, max_deg=10, append_identity=True):
    """Standard NeRF positional encoding."""
    scales = np.array([2**i for i in range(min_deg, max_deg + 1)])
    xb = np.reshape((x[..., None, :] * scales[:, None]), (x.shape[0], -1))
    fb = np.sin(xb * np.pi)
    if append_identity:
        fb = np.concatenate([x, fb], axis=-1)
    return fb

def forward_pass(p, v, mlp_weights):
    """
    p: (N, 3) query points
    v: (N, 2) viewing directions
    Returns: sigma (N,), o (N, H+1), c (N, 3)
    """
    pe_p = positional_encoding(p)
    pe_v = positional_encoding(v)
    # Concatenate PE of position and direction
    input_vec = np.concatenate([pe_p, pe_v], axis=-1)
    
    # Pass through MLP defined by mlp_weights
    # Architecture: 128 -> 256 (x9) -> 128 -> Heads
    # Note: Specific weight loading omitted for brevity, assuming standard MLP forward
    hidden = mlp_forward(input_vec, mlp_weights)
    
    # Split outputs
    sigma_raw, o_raw, c = split_heads(hidden)
    
    sigma = 1.0 / (1.0 + np.exp(-sigma_raw)) # Sigmoid
    o = np.exp(o_raw) / np.sum(np.exp(o_raw), axis=-1, keepdims=True) # Softmax
    
    return sigma, o, c

def render_ray(ray_origin, ray_direction, t_near, t_far, num_samples, mlp_weights):
    """Standard volumetric rendering for a single ray."""
    t_vals = np.linspace(t_near, t_far, num_samples)
    deltas = t_vals[1:] - t_vals[:-1]
    deltas = np.append(deltas, np.finfo(float).eps)
    
    xyz = ray_origin + ray_direction * t_vals[:, None]
    sigma, o, c = forward_pass(xyz, ray_direction, mlp_weights)
    
    # Alpha values
    alpha = 1.0 - np.exp(-sigma * deltas)
    T = np.cumprod(np.concatenate([np.ones((1,)), 1.0 - alpha + np.finfo(float).eps], axis=0))[:-1]
    
    weights = alpha * T[:, None]
    color = np.sum(weights * c, axis=0)
    
    # Render object channels
    obj_channels = np.sum(weights * o, axis=0) # Shape (H+1,)
    
    return color, obj_channels

def match_2d_masks(pred_obj_channels, gt_masks, H, T):
    """
    pred_obj_channels: (N, H+1) rendered object probs for one image
    gt_masks: (N, T) ground truth one-hot masks
    Returns: matched indices
    """
    # Reshape to (N, H) and (N, T)
    pred = pred_obj_channels[:, :H]
    gt = gt_masks
    
    # Compute cost matrix using sIoU and CES
    # Note: Exact formula for sIoU/CES cost matrix construction is complex,
    # simplified here as a placeholder for the Hungarian algorithm input
    cost_matrix = compute_cost_matrix(pred, gt)
    
    row_ind, col_ind = linear_sum_assignment(cost_matrix)
    return row_ind, col_ind

def train_step(images, masks, poses, intrinsics, mlp_weights):
    """Single training step."""
    total_loss = 0
    for i in range(len(images)):
        # Setup ray
        ray_origins, ray_directions = get_rays(poses[i], intrinsics[i], images[i].shape)
        
        # Render
        colors, obj_channels = render_batch(ray_origins, ray_directions, mlp_weights)
        
        # Compute photometric loss
        photometric_loss = np.mean((colors - images[i])**2)
        
        # Match 2D masks
        # Flatten obj_channels for matching
        N = images[i].shape[0] * images[i].shape[1]
        pred_flat = obj_channels.reshape(-1, H + 1)
        gt_flat = masks[i].reshape(-1, T)
        
        row_ind, col_ind = match_2d_masks(pred_flat, gt_flat, H, T)
        
        # Compute object loss based on matched indices
        # This involves mapping 2D matched masks back to 3D points along the ray
        # and comparing with the 3D object codes o
        object_loss = compute_object_loss(obj_channels, gt_flat, row_ind, col_ind)
        
        # Add empty space regularization loss
        empty_loss = compute_empty_loss(obj_channels, gt_flat)
        
        total_loss += photometric_loss + object_loss + empty_loss
        
    return total_loss

def manipulate_object(scene_points, object_code_index, transformation):
    """
    scene_points: (N, 3)
    object_code_index: int, index of the object to manipulate
    transformation: function or matrix to apply
    """
    # Identify points belonging to the object
    # This requires querying the object field for all points in the scene
    # or using the inverse query algorithm mentioned in the paper
    mask = get_object_mask(scene_points, object_code_index)
    
    # Apply transformation
    manipulated_points = scene_points[mask].copy()
    manipulated_points = transformation(manipulated_points)
    
    # Tackle visual occlusions
    # The paper mentions a lightweight inverse query algorithm for this
    # Implementation detail not fully specified in sources
    manipulated_points = tackle_occlusions(manipulated_points, scene_points)
    
    return manipulated_points
```

**Parameters**
*   $H$: Predefined number of objects, determines the channel dimension of 2D object prediction tensors [S1, unofficial].
*   $T$: Number of ground truth objects, determines the channel dimension of 2D ground truth label tensors [S1, unofficial].
*   $L$: Number of generated 2D images used for 2D object matching and supervision [S1, unofficial].
*   $U, V$: Spatial width and height dimensions of the 2D object prediction and ground truth label images [S1, unofficial].
*   Typical values for $H$ and $T$ are not explicitly given in the sources, but $H$ must be $\ge T$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **No 3D Ground Truth:** Learning the object code faces the critical issue that no ground truth 3D object codes are available for full supervision [S1, unofficial].
*   **Variable Object Count:** The number of total objects is variable and there is no fixed order for objects [S1, unofficial].
*   **Empty Space:** Non-occupied (empty) 3D space must be taken into account, but there are no labels for supervision as well [S1, unofficial]. This is addressed by jointly learning correct codes for empty 3D points using $\ell_{3d}^{empty}$ [S1, unofficial].
*   **Occlusion Artifacts:** Manipulating originally occluded objects or parts may produce artifacts due to inaccuracies in the learned radiance fields [S1, unofficial].
*   **Lighting Discrepancies:** Lighting factors are not decomposed, causing illumination discrepancies on edited objects compared to non-manipulated scenes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method uses a standard NeRF backbone, implying similar training time to NeRF, plus the overhead of 2D mask matching (Hungarian algorithm) per image per step [S1, unofficial].
*   **Memory Complexity:** Not given in the sources. Requires storing the MLP weights and rendering object channels for all rays [S1, unofficial].
*   **Measured Performance:** Novel view rendering quality is clearly better than PointNeRF, though it decreases after manipulation compared to non-manipulation baselines [S1, unofficial].
*   **Strengths:** Simultaneously reconstructs, decomposes, manipulates, and renders complex 3D scenes in a single pipeline from 2D images [S1, unofficial]. Robust object decomposition from 2D views [S1, unofficial].
*   **Limitations:** Lighting factors are not decomposed [S1, unofficial]. Manipulation of occluded objects may produce artifacts [S1, unofficial].
*   **When to use:** When 3D object-level manipulation and decomposition are required from 2D images only, and 3D ground truth is unavailable [S1, unofficial].
*   **When not to use:** When high-fidelity lighting manipulation is required, or when the scene contains many occluded objects that need precise editing [S1, unofficial].

## Background
DM-NeRF was published as a conference paper at ICLR 2023 [S1, unofficial]. It addresses the problem of 3D scene geometry decomposition and manipulation from 2D images [S1, unofficial]. It builds on the success of NeRF by adding an object field component [S1, unofficial]. The method was developed to enable users to virtually manipulate objects in 3D scenes (e.g., moving a chair) in applications like mixed reality [S1, unofficial].

## Sources
[S1] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
