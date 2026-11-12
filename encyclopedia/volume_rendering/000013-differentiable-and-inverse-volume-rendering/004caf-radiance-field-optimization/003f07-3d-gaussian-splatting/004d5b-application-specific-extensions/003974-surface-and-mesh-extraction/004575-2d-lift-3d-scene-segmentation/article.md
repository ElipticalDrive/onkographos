# 2D-Lift-3D Scene Segmentation

Also called: 2D-Lift-3D Segmentation for Partial Scene Editing, PSE with 2D-lift-3D segmentation

## Overview
This method segments 3D Gaussian primitives in a volume visualization scene by lifting 2D segmentation masks from reference views into 3D space. It solves the problem of partial scene editing, allowing users to apply distinct color, opacity, and lighting effects to specific regions of a scene represented by Gaussian kernels. Unlike methods that require learnable feature vectors or iterative optimization for segmentation, this approach is training-free and relies on multi-view label voting to assign 3D labels to primitives based on 2D user prompts.

## How it works

### Inputs and Outputs
**Inputs:**
*   A trained 3D Gaussian Splatting model consisting of $M$ anisotropic 3D Gaussian kernels $\mathcal{G} = \{G_i\}_{i=1}^M$ [S1].
*   A set of training views with known camera poses and intrinsics [S1].
*   User-specified point prompts on a reference view image [S1].
*   A Segment Anything Model (SAM) instance [S1].

**Outputs:**
*   A label assignment $L_i \in \{1, \dots, K\}$ for each Gaussian primitive $G_i$, where $K$ is the number of segmented parts [S1].
*   Segmented groups of Gaussian primitives enabling independent editing [S1].

### Data Structures
The core data structure is the set of Gaussian primitives $\mathcal{G}$. Each primitive $G_i$ retains the standard 3DGS attributes defined in the parent article [Surface and Mesh Extraction](8498-surface-and-mesh-extraction.md):
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation.
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale.
*   $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$: Covariance matrix [S1].

Additionally, the method implicitly or explicitly maintains a label vector $\mathbf{l} \in \{0, \dots, K\}^M$, where $0$ indicates an unassigned or background primitive, and $1 \dots K$ correspond to specific semantic parts.

### Algorithm Steps
The segmentation process follows a training-free, inference-time strategy inspired by SAGD [S1]:

1.  **Reference View Rendering:** Render an image from the 3DGS model from an arbitrary reference view [S1].
2.  **2D Mask Generation:** Use SAM with user-specified point prompts on the rendered reference view to generate a 2D segmentation mask for the target object/part [S1].
3.  **Multi-View 2D Mask Generation:** Generate 2D segmentation masks for the corresponding parts across all available training views using SAM. This requires projecting the user's intent or using consistent prompts across views, though the source notes SAM is used on "corresponding parts across all training views" [S1].
4.  **Label Propagation (2D):** For each training view, assign distinct labels to the 2D pixels belonging to the segmented mask.
5.  **3D Label Assignment via Rasterization:** For each Gaussian primitive $G_i$:
    *   Project $G_i$ to each training view to determine its 2D footprint.
    *   Check if the 2D footprint of $G_i$ overlaps with the 2D segmentation mask in that view.
    *   If it overlaps, assign the label associated with that mask to $G_i$ for that view.
6.  **Multi-View Label Voting:** Aggregate the labels assigned from all training views for each primitive $G_i$. The final label $L_i$ is determined by majority voting among the views where $G_i$ was visible and overlapped with a mask [S1].
7.  **Segmentation Decomposition:** Group primitives by their final label $L_i$ to form distinct 3D segments [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def segment_gaussians_2d_lift_3d(gaussians, training_views, sam_model, point_prompts_ref_view):
    """
    Segments 3D Gaussians using 2D-lift-3D strategy.
    
    Args:
        gaussians: List of M Gaussian primitives, each with position, rotation, scale.
        training_views: List of camera parameters (extrinsics, intrinsics) for all training views.
        sam_model: Pre-trained Segment Anything Model.
        point_prompts_ref_view: List of (x, y) point coordinates on the reference view image.
        
    Returns:
        labels: Array of shape (M,) containing integer labels for each Gaussian.
    """
    M = len(gaussians)
    # Initialize labels to 0 (unassigned/background)
    labels = np.zeros(M, dtype=int)
    
    # Step 1: Render reference view and get 2D mask
    ref_view = training_views[0] # Assume first view is reference
    ref_image = render_gaussians(gaussians, ref_view)
    mask_ref = sam_model.predict(ref_image, point_prompts_ref_view)
    
    # Step 2: Generate masks for all training views
    # Note: The source implies SAM is used on corresponding parts across all views.
    # In practice, this might require consistent prompting or automatic propagation.
    # Here we assume we have masks for all views.
    all_masks = []
    for view in training_views:
        view_image = render_gaussians(gaussians, view)
        # Prompting strategy for other views is not detailed in source [S1].
        # We assume a function get_mask_for_view exists which handles SAM prompting.
        mask_view = get_mask_for_view(sam_model, view_image, view, point_prompts_ref_view)
        all_masks.append(mask_view)
        
    # Step 3: Label Propagation and Voting
    # For each Gaussian, check overlap with masks in all views
    for i in range(M):
        G_i = gaussians[i]
        view_votes = []
        
        for v_idx, view in enumerate(training_views):
            mask = all_masks[v_idx]
            
            # Project Gaussian to 2D
            # Compute projected covariance and bounding box
            proj_cov, center_2d = project_gaussian_2d(G_i, view)
            
            # Determine if Gaussian overlaps with the segmentation mask
            # A simple check: is the center inside the mask? Or bounding box intersection?
            # Source says "identifying and segmenting the corresponding Gaussian primitives"
            # and "multi-view label voting".
            if is_overlapping(center_2d, proj_cov, mask):
                # Assign the label from the mask. 
                # Assuming mask has a single label for the target part in this context.
                label_v = 1 
                view_votes.append(label_v)
                
        # Step 4: Majority Voting
        if view_votes:
            # Simple majority vote
            unique, counts = np.unique(view_votes, return_counts=True)
            labels[i] = unique[np.argmax(counts)]
        else:
            labels[i] = 0 # Background or not segmented
            
    return labels

def is_overlapping(center_2d, proj_cov, mask):
    """
    Checks if a projected Gaussian overlaps with a binary mask.
    [G] Implementation detail: Check if the center lies within the mask,
    or if the bounding box of the Gaussian intersects the mask region.
    """
    # Simple implementation: check center
    h, w = mask.shape
    x, y = center_2d
    if 0 <= x < w and 0 <= y < h:
        return mask[y, x] > 0
    return False

def project_gaussian_2d(gaussian, view):
    """
    Projects a 3D Gaussian to 2D using the standard 3DGS projection pipeline.
    Returns 2D center and 2D covariance.
    """
    # 1. Transform position to camera space
    p_cam = view.extrinsic @ gaussian.position
    
    # 2. Compute 2D covariance
    # J is Jacobian of affine approximation of projection
    # W is rotation part of extrinsic
    # Sigma_3D = R S S^T R^T
    Sigma_3D = gaussian.rotation @ gaussian.scale @ gaussian.scale.T @ gaussian.rotation.T
    W = view.extrinsic[:3, :3]
    J = view.intrinsic @ W # Simplified; actual J involves depth and focal length
    # Standard 3DGS formula: Sigma_hat = J W Sigma_3D W^T J^T
    Sigma_2D = J @ W @ Sigma_3D @ W.T @ J.T
    
    # 2D Center
    center_2d = view.intrinsic @ p_cam[:3] / p_cam[2]
    
    return Sigma_2D, center_2d
```

### Parameters
*   **SAM Model:** The specific variant of Segment Anything Model used. Not given in the sources.
*   **Point Prompts:** User-defined 2D coordinates. Not given in the sources.
*   **Voting Threshold:** The method uses "multi-view label voting." The exact threshold for majority vote is not specified, implying a simple majority is used [S1].

### Edge Cases and Pitfalls
*   **Inefficiency with Point Prompts:** The method cannot immediately generate a perfect 3D segmentation from 2D masks created with point prompts on a single reference view. It may require multiple rounds of segmentation for certain scenes, which is inefficient [S1].
*   **Visibility:** Gaussians not visible in any training view where the mask is generated will not be segmented.
*   **SAM Accuracy:** The quality of segmentation depends entirely on SAM's ability to segment the object in all training views based on the prompts.

## Operating characteristics

### Time and Memory Complexity
*   **Time:** The method is training-free, so it has no training time overhead. The segmentation time depends on the number of training views $V$ and Gaussians $M$. It requires rendering $V$ images and running SAM $V$ times. The voting step is $O(M \cdot V)$.
*   **Memory:** Requires storing the masks for all training views and the label array for $M$ Gaussians. Not given in the sources.

### Measured Performance
*   The source does not provide specific timing metrics for the segmentation step alone. It mentions rendering times for the overall VolVis system (e.g., 29.68 ms for 256x256x256 volume) [S1].

### Strengths and Limitations
*   **Strengths:** Training-free, flexible, allows partial scene editing (PSE) and non-photorealistic stylization (NPSE) on specific parts [S1]. Supports post-stylization color and lighting edits via the textures attribute and Blinn-Phong model, unlike StyleRF-VolVis [S1].
*   **Limitations:** Inefficient for scenes requiring multiple rounds of segmentation due to the reliance on 2D point prompts and SAM [S1].

## Background

### Origin
The method is introduced in the context of **TexGS-VolVis**, a framework for expressive scene editing in volume visualization using 2D Gaussian Splatting. It is cited as a technique to enable partial scene editing by segmenting Gaussian primitives [S1]. The approach is inspired by SAGD [25] (cited in S1).

### History and Development
The method addresses the limitation of previous works like StyleRF-VolVis, which could not apply distinct edits to different parts of a basic scene. By lifting 2D segmentations to 3D, TexGS-VolVis achieves higher granularity in editing [S1].

### Usage
Used in volume visualization (VolVis) for scientific data (e.g., jets, mantle, supernova, beetle, chameleon, engine, ionization volume) to allow users to isolate and edit specific features [S1].

## Sources

[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
