# Template-Free NeRF (TFS-NeRF)

Also called: TFS-NeRF

## Overview
TFS-NeRF is a template-free 3D semantic NeRF method for reconstructing dynamic scenes from sparse or single-view RGB videos, specifically handling interactions between two or more entities (rigid, non-rigid, or deformable) [S1]. It distinguishes itself from sibling methods like Nerfies by explicitly disentangling the motions of interacting entities and from LBS-based approaches like TAVA by using an Invertible Neural Network (INN) to predict Linear Blend Skinning (LBS) weights directly, thereby bypassing computationally expensive root-finding optimizations [S1]. This enables time-efficient training and semantically separable geometry reconstruction without relying on pre-defined template models (e.g., SMPL) or latent codes [S1].

## How it works
**Inputs:** Sparse or single-view RGB video frames and corresponding 2D semantic segmentation masks [S1].
**Outputs:** A composite scene rendering and semantically separable Signed Distance Fields (SDFs) for deformable and non-deformable entities [S1].

**Data Structures:**
*   **Canonical Space:** A static reference frame where geometry is defined independently of time and pose.
*   **Deformed/View Space:** The coordinate system of the observed frame at a specific timestamp.
*   **Bone Transformation Matrices ($B$):** A set of $n_b$ rigid transformation matrices representing the bones of the deformable entity [S1].
*   **Skining Weights ($w(x_c)$):** A function mapping a canonical point $x_c$ to a weight vector, predicted by the INN [S1].
*   **SDF Networks ($F_{c \to \Omega}^{(\theta)}$):** Two independent neural networks (MLPs) parameterized by $\theta$, one for deformable entities ($j=d$) and one for non-deformable entities ($j=nd$), predicting canonical SDFs [S1].

**Formulas:**
The forward Linear Blend Skinning (LBS) computes the deformed point $x_v$ in view space from a canonical point $x_c$ using skinning weights $w(x_c)$ and bone transformation matrices $B$:
$$
x_v = \text{LBS}(w(x_c), B, x_c) = \sum_{b=1}^{n_b} w(x_c)_b \cdot B_b \cdot x_c
$$
where $n_b$ is the total number of bones [S1].

**Steps:**
1.  **Semantic-Aware Ray Sampling:** Given a 2D semantic segmentation mask for the current frame, shoot two distinct sets of rays from the camera:
    *   One set samples 3D points $\{x_d^v\}$ corresponding to deformable entities.
    *   Another set samples 3D points $\{x_v\}$ corresponding to non-deformable entities [S1].
2.  **Inverse Warping to Canonical Space:** Transform the sampled points from the deformed/view space to their respective canonical spaces. This is achieved by inverting the forward LBS mapping. Since TFS-NeRF uses an INN to predict the forward LBS weights $w(x_c)$, the inverse warping is enabled by the invertibility of the INN, avoiding the iterative root-finding procedures used in prior work [S1].
3.  **Individual Geometry Prediction:** At the canonical space, use two independent SDF prediction networks $F_{c \to \Omega}^{(\theta)}$ to predict the Signed Distance Fields:
    *   $F_{c \to \Omega}^{(\theta, d)}$ predicts the SDF for the deformable entity.
    *   $F_{c \to \Omega}^{(\theta, nd)}$ predicts the SDF for the non-deformable entity [S1].
4.  **Joint Optimization:** Optimize both branches (deformable and non-deformable) jointly. The optimization minimizes the RGB reconstruction loss between the rendered composite image and the ground truth image, while simultaneously learning the per-entity skinning weights and SDFs [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tfs_nerf_forward(frame_rgb, semantic_mask, camera_params, inn_model, sdf_net_deform, sdf_net_nondeform, bones):
    """
    Performs one forward pass of TFS-NeRF.
    
    Args:
        frame_rgb: RGB image of the current frame.
        semantic_mask: 2D semantic segmentation mask for the current frame.
        camera_params: Intrinsics and extrinsics for the current view.
        inn_model: Invertible Neural Network predicting LBS weights w(x_c).
        sdf_net_deform: MLP predicting SDF for deformable entities.
        sdf_net_nondeform: MLP predicting SDF for non-deformable entities.
        bones: List of bone transformation matrices B_b.
        
    Returns:
        rendered_color: Rendered RGB color.
        loss: Reconstruction loss.
    """
    # 1. Semantic-Aware Ray Sampling
    # Shoot rays based on semantic mask to separate deformable and non-deformable regions
    rays_deform, rays_nondeform = semantic_ray_sampling(semantic_mask, camera_params)
    
    # Sample 3D points along these rays in the deformed/view space
    # Note: Depth sampling strategy (e.g., uniform or importance sampling) is not specified in sources [G]
    points_deform_v = sample_points_along_rays(rays_deform)
    points_nondeform_v = sample_points_along_rays(rays_nondeform)
    
    # 2. Inverse Warping to Canonical Space
    # Predict LBS weights w(x_c) using INN. 
    # The INN takes canonical points as input and outputs weights. 
    # However, we have deformed points. The INN is used to bypass root-finding.
    # The source states: "inverse warping enabled by the learned forward LBS".
    # This implies the INN allows direct computation or efficient inversion.
    # We assume the INN predicts weights for canonical points, and we need to find x_c such that LBS(w(x_c), B, x_c) = x_v.
    # TFS-NeRF uses INN to predict LBS directly. The source contrasts with TAVA which uses root-finding.
    # Implementation detail: The INN likely maps x_c -> w. To go x_v -> x_c, we might need an inverse INN or 
    # the INN is structured to allow direct inversion. Given "INN for LBS prediction", we assume:
    # w = inn_model(x_c). 
    # To find x_c from x_v, we solve x_v = sum(w(x_c)_b * B_b * x_c).
    # The key contribution is that this is solved efficiently via INN, avoiding iterative root-finding [S1].
    # For the purpose of the algorithm, we denote the inverse mapping as:
    points_deform_c = inverse_warp_deform(points_deform_v, inn_model, bones)
    points_nondeform_c = inverse_warp_nondeform(points_nondeform_v) # Non-deformable may not need LBS inversion if static, 
                                                                    # but source says "transform... to respective canonical spaces" [S1]
    
    # 3. Individual Geometry Prediction
    # Predict SDFs at canonical points
    sdf_deform = sdf_net_deform(points_deform_c)
    sdf_nondeform = sdf_net_nondeform(points_nondeform_c)
    
    # 4. Volume Rendering
    # Render the composite scene from the SDFs
    # Standard volume rendering from SDF (e.g., using marching cubes or differentiable ray marching) is implied [G]
    rendered_color = volume_render_sdf(sdf_deform, sdf_nondeform, rays_deform, rays_nondeform)
    
    # 5. Loss Calculation
    loss = rgb_loss(rendered_color, frame_rgb)
    
    return rendered_color, loss

def inverse_warp_deform(points_v, inn_model, bones):
    """
    Maps deformed points to canonical space using INN-predicted LBS.
    Avoids iterative root-finding by leveraging INN properties [S1].
    """
    # The source does not provide the exact mathematical inversion formula, 
    # only that INN bypasses root-finding. 
    # A common approach with INN is that the network is bijective.
    # If INN predicts w from x_c, we need x_c from x_v.
    # This step is the core efficiency gain [S1].
    # Placeholder for the specific INN inversion logic not detailed in source [G]
    return inn_model.inverse(points_v)
```

**Parameters:**
*   **Number of Bones ($n_b$):** Total number of bones in the deformable entity. Not given in the sources.
*   **Semantic Mask:** Required input for ray sampling. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Occlusions:** Complex interactions between entities can cause occlusions, which the method handles by disentangling motions [S1].
*   **Topological Variations:** The method is designed for dynamic scenes but does not explicitly handle topological changes within a frame (unlike Tensor4D which also lacks this) [S1].
*   **Precision:** The use of INN avoids the numerical instability and convergence issues associated with Broyden’s method used in prior LBS-based NeRFs [S1].

## Subtopics

### Linear Blend Skinning (LBS) Prediction via INN
A technique where an Invertible Neural Network predicts skinning weights directly, replacing iterative root-finding. It sets itself apart by offering time-efficient training compared to methods like TAVA. Choose this when training speed is critical and the scene involves deformable entities with known bone structures. [Link to TFS-NeRF](52-tfs-nerf.md).

### Semantic-Aware Ray Sampling
A sampling strategy that uses 2D semantic masks to separate rays into deformable and non-deformable categories. It sets itself apart by enabling independent optimization of interacting entities. Choose this when semantic separation of dynamic and static parts is required. [Link to TFS-NeRF](52-tfs-nerf.md).

## Operating characteristics
*   **Time Efficiency:** TFS-NeRF is significantly more time-efficient than LBS-based approaches like TAVA [28] because it bypasses the computationally demanding root-finding optimization (e.g., Broyden’s method) for LBS inversion [S1].
*   **Memory:** Not given in the sources.
*   **Performance:** Produces high-quality reconstructions of both deformable and non-deformable objects in complex interactions (e.g., human-object, hand-object) [S1].
*   **Strengths:** Template-free (no need for SMPL or similar priors), semantically separable, handles interactions between multiple entities, efficient training.
*   **Limitations:** Requires 2D semantic segmentation masks as input. Relies on LBS, which assumes a skeletal structure for deformable entities.
*   **Scaling:** Not given in the sources.

## Background
TFS-NeRF was introduced in the paper "TFS-NeRF: Template-Free NeRF for Semantic 3D Reconstruction of Dynamic Scene" [S1]. It builds upon prior work such as TAVA [28], which used computationally expensive root-finding for LBS, and addresses the challenge of reconstructing dynamic scenes with interacting entities without relying on template models like SMPL [S1]. The method was developed to handle sparse or single-view RGB videos, extending the capabilities of NeRFies [Parent] by adding semantic separation and efficient LBS prediction via INN [S1].

## Sources
[S1] TFS-NeRF: Template-Free NeRF for Semantic 3D Reconstruction of Dynamic Scene, https://arxiv.org/abs/2409.17459v4, fetched 2026-10-26
