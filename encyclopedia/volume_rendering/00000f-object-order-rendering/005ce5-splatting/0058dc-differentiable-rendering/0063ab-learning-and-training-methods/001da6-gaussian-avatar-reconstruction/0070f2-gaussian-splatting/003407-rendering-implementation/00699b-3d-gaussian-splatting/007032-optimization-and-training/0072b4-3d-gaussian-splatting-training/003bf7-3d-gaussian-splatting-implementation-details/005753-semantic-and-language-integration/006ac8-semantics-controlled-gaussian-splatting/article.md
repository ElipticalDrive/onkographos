# Semantics-Controlled Gaussian Splatting

Also called: SCGS

## Overview
Semantics-Controlled Gaussian Splatting (SCGS) is a method that integrates semantic segmentation directly into the 3D Gaussian Splatting representation by assigning class IDs to individual Gaussians [S1]. It solves the problem of accurate scene editing and large-scale object removal in non-circular capturing setups, where traditional convex hull methods fail [S1]. Unlike sibling methods that rely on separate classifiers or feature similarity grouping, SCGS alters the rasterization process to allow direct classification in both 2D image space and 3D Gaussian space, enabling targeted removal of specific classes without feature-based ambiguity [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse point cloud (from COLMAP), camera poses, RGB images, and ground-truth semantic segmentation maps (continuous labels from SAM DEVA) [S1].
*   **Outputs:** Optimized 3D Gaussians with semantic class IDs, rendered RGB images, and rendered semantic maps [S1].

**Data Structures:**
Each Gaussian $G$ is defined by the standard 3D GS parameters plus a semantic attribute:
*   Center position: $\mathbf{x} \in \mathbb{R}^3$
*   Covariance matrix: $\Sigma \in \mathbb{R}^{3 \times 3}$ (derived from rotation and scaling)
*   Density: $\sigma$
*   Spherical Harmonics (SH) coefficients: $\mathbf{c}$ for RGB color
*   Semantic class ID: $s_i$ (assigned directly to the Gaussian) [S1]

**Numbered Steps:**

1.  **Initialization:** Initialize 3D Gaussians from a sparse point cloud. Each Gaussian $G$ is represented by its 3D center position $\mathbf{x}$, a 3D covariance matrix $\Sigma$, a density value $\sigma$, and SH coefficients for RGB [S1].

2.  **2D Rasterization (RGB):** Convert SH coefficients into RGB values. Splat Gaussians onto the 2D image plane using alpha blending to retrieve pixel color [S1]:
    $$ \text{RGB} = \sum_{i \in \mathcal{N}} T_i \alpha_i c_i $$
    where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance, $\alpha_i$ is opacity, and $c_i$ is the color [S1].

3.  **2D Rasterization (Semantic):** Perform a parallel rasterization process for the semantic map. Set SH components to zero to isolate semantic attributes from RGB. The segmentation value for a pixel is computed via alpha blending of class probabilities [S1]:
    $$ \text{SEGMENTATION} = \sum_{i \in \mathcal{N}} T_i \alpha_i s_{ic} $$
    where $s_{ic}$ is the predicted probability for class $c$ at pixel $i$ [S1].

4.  **Covariance Transformation:** Calculate the 2D rasterized Gaussian covariance $\Sigma'$ from the 3D covariance $\Sigma$ using the Jacobian $J$ of the affine approximation and the world-to-camera transformation matrix $W$ [S1]:
    $$ \Sigma' = J W \Sigma W^T J^T $$
    [S1]

5.  **Loss Computation:** Apply cross-entropy loss to supervise the 3D GS segmentation through the 2D loss function, enabling backpropagation of the segmentation map similarly to RGB values [S1]. The loss function is defined as:
    $$ \mathcal{L}_{seg} = - \sum_{i} (s_{ic} \log(\hat{s}_{ic})) $$
    where $s_{ic}$ is the ground truth segmentation for class $c$ at pixel $i$, and $\hat{s}_{ic}$ is the predicted probability [S1].

6.  **Object Removal (Inference):** For object removal, filter Gaussians using an indicator function to retain only those not belonging to the removed class ID $c_r$:
    $$ G_{new}(\mathbf{x}; \Sigma) = G(\mathbf{x}; \Sigma) \cdot I(s_i \neq c_r) $$
    where $I$ is the indicator function, ensuring only Gaussians not belonging to the removed class ID are retained [S1].

**Parameters:**
*   Hardware: Single RTX 4090 with 24GB VRAM [S1].
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   Traditional convex hull methods for object removal may remove too much of the scene or fail in non-circular capturing setups; SCGS avoids this by using direct class-based removal [S1].
*   Feature similarity issues in outdoor scenes with significant reflections are addressed by direct class assignment rather than feature space grouping [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scgs_render(gaussians, camera, image_width, image_height):
    """
    Renders RGB and Semantic maps for a single view.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - center: (3,)
            - cov3d: (3, 3)
            - opacity: (1,)
            - sh_rgb: (K,) SH coefficients for color
            - semantic_id: (1,) Class ID
        camera: Camera object with attributes:
            - W2C: (4, 4) World-to-camera matrix
            - fov: Field of view
        image_width, image_height: int
        
    Returns:
        rgb_image: (H, W, 3)
        semantic_image: (H, W, C) One-hot or probability map
    """
    # 1. Project Gaussians to 2D
    # Calculate 2D covariance Sigma' = J * W * Sigma * W^T * J^T
    # J is Jacobian of affine approximation, W is world-to-camera
    # This step is handled by the CUDA rasterizer kernel
    projected_gaussians = project_gaussians(gaussians, camera)
    
    # 2. Rasterize RGB
    # RGB = sum(T_i * alpha_i * c_i)
    rgb_image = rasterize_alpha_blend(
        projected_gaussians, 
        attribute_key='sh_rgb', 
        alpha_key='opacity',
        width=image_width,
        height=image_height
    )
    
    # 3. Rasterize Semantic
    # Isolate semantic attributes by setting SH components to zero for color
    # SEGMENTATION = sum(T_i * alpha_i * s_ic)
    # s_ic is the probability for class c. Since SCGS assigns direct IDs,
    # we treat the ID as a one-hot vector or probability distribution over classes.
    semantic_image = rasterize_alpha_blend(
        projected_gaussians, 
        attribute_key='semantic_id', # Treated as class probabilities/one-hot
        alpha_key='opacity',
        width=image_width,
        height=image_height
    )
    
    return rgb_image, semantic_image

def scgs_remove_class(gaussians, class_id_to_remove):
    """
    Removes all Gaussians belonging to a specific class.
    
    Args:
        gaussians: List of Gaussian objects
        class_id_to_remove: int
        
    Returns:
        filtered_gaussians: List of Gaussian objects
    """
    filtered_gaussians = []
    for g in gaussians:
        # G_new(x; Sigma) = G(x; Sigma) * I(s_i != c_r)
        if g.semantic_id != class_id_to_remove:
            filtered_gaussians.append(g)
    return filtered_gaussians
```

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Outperforms baseline using continuous labels from SAM DEVA on all tested outdoor scenes. On the 3D-OVS dataset, it outperforms existing work on one out of five scenes and performs competitively on others [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Enables accurate scene editing and extraction of large scene parts; works well in non-"circling" setups; avoids feature similarity issues in reflective outdoor scenes; allows direct removal of complete classes at a large scale [S1].
*   **Limitations:** Requires ground-truth semantic segmentation maps for training (continuous labels from SAM DEVA) [S1].
*   **When to use:** When accurate large-scale object removal or scene editing is required in non-circular capturing setups, especially for outdoor scenes with reflections [S1].
*   **When not:** When ground-truth semantic labels are unavailable or when computational resources are severely limited (compared to methods requiring less VRAM, though SCGS fits on 24GB) [S1].

## Background
*   **Origin:** SCGS was introduced in the paper "Semantics-Controlled Gaussian Splatting for Outdoor Scene Reconstruction and Rendering in Virtual Reality" [S1].
*   **History and Development:** The method builds on 3D Gaussian Splatting [20] and addresses limitations in object removal for non-circular capturing setups [S1]. It diverges from traditional classification-based approaches [60, 63] by assigning class IDs directly during training [S1].
*   **Where it is used:** Outdoor scene reconstruction and rendering in Virtual Reality, specifically for scenarios where the user is surrounded by the virtual environment [S1].

## Sources
[S1] Semantics-Controlled Gaussian Splatting for Outdoor Scene Reconstruction and Rendering in Virtual Reality, https://doi.org/10.1109/vr59515.2025.00056, fetched 2026-10-30
