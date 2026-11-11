# 2D-Lift-3D Scene Segmentation

Also called: 2D-Lift-3D Segmentation for Partial Scene Editing, PSE with 2D-lift-3D segmentation

## Overview
This method segments 3D Gaussian primitives in a volume visualization scene by lifting 2D segmentation masks from reference views into 3D space. It solves the problem of partial scene editing by decomposing a scene into distinct parts, allowing independent color, opacity, and lighting adjustments [S1]. Unlike sibling methods that may require learnable feature vectors or distance-based metrics, this approach is training-free and relies on multi-view label voting [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained set of $N$ 3D Gaussian primitives $\mathcal{G} = \{g_1, \dots, g_N\}$; a set of training view images; user-specified point prompts on a reference view.
*   **Output:** A label assignment $L_i \in \{0, 1, \dots, K\}$ for each Gaussian $g_i$, where $K$ is the number of segments.

**Data Structures:**
*   **Gaussian Primitives:** Defined by center $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$, opacity $o_i$, and SH coefficients $\mathbf{c}_i$ [S1].
*   **Labels:** An integer array of size $N$ storing the segment ID for each Gaussian.

**Steps:**
1.  **Render Reference View:** Render an image from the current Gaussian scene using the standard differentiable rasterization pipeline [S1].
2.  **Generate 2D Mask:** Use the Segment Anything Model (SAM) with user-specified point prompts on the rendered reference view to generate a 2D segmentation mask [S1].
3.  **Generate Multi-View Masks:** Use SAM to generate 2D segmentation masks for the corresponding parts across all available training views [S1].
4.  **Label Propagation:** Assign distinct labels to each 2D Gaussian based on the 2D masks. For each training view $v$, project the 3D Gaussians to 2D. If a Gaussian $g_i$ projects to a pixel covered by a specific segment in the 2D mask, assign a temporary label $l_{i,v}$ [S1].
5.  **Multi-View Label Voting:** Apply multi-view label voting to determine the final 3D segmentation. For each Gaussian $g_i$, aggregate the labels $l_{i,v}$ from all views where $g_i$ is visible. The final label $L_i$ is the majority vote among the valid projections [S1].
6.  **Decomposition:** Group Gaussians by their final label $L_i$ to form distinct segments [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def segment_gaussians(gaussians, training_views, point_prompts_ref):
    """
    gaussians: List of Gaussian objects with properties:
               - position: (3,)
               - covariance: (3,3)
               - opacity: float
               - sh_coeffs: array
    training_views: List of view objects with properties:
               - camera_matrix: (4,4)
               - image: (H, W, 3)
    point_prompts_ref: List of (x, y) coordinates for SAM on the reference view.
    """
    N = len(gaussians)
    # Initialize labels to -1 (unassigned)
    labels = np.full(N, -1, dtype=int)
    
    # Step 1: Get reference view mask
    ref_view = training_views[0] # Assume first view is reference
    ref_image = render_scene(gaussians, ref_view.camera_matrix)
    ref_mask = sam_segment(ref_image, point_prompts_ref)
    
    # Step 2: Get masks for all training views
    all_masks = []
    for view in training_views:
        view_image = render_scene(gaussians, view.camera_matrix)
        # Note: SAM requires prompts for each view. 
        # In practice, these might be derived from the reference prompt via 
        # geometric correspondence or user input. The source implies 
        # "corresponding parts across all training views" are generated.
        # We assume a function `get_corresponding_prompts` exists or 
        # prompts are provided for all views.
        prompts_view = get_corresponding_prompts(point_prompts_ref, ref_view, view)
        mask = sam_segment(view_image, prompts_view)
        all_masks.append(mask)
        
    # Step 3: Label Propagation and Voting
    # We need to map 2D mask regions to 3D Gaussians.
    # Since Gaussians are projected, we check which Gaussian covers which pixel.
    
    # Initialize vote counters for each Gaussian and each potential label
    # Assuming labels are 0..K-1 based on mask output
    num_labels = len(np.unique(ref_mask))
    votes = np.zeros((N, num_labels), dtype=int)
    visibility_counts = np.zeros(N, dtype=int)
    
    for view_idx, view in enumerate(training_views):
        mask = all_masks[view_idx]
        
        # Project all Gaussians to 2D for this view
        # This returns a list of projected 2D Gaussians with screen-space covariance
        projected = project_gaussians(gaussians, view.camera_matrix)
        
        # For each Gaussian, determine if it falls within a labeled region
        for i, proj_g in enumerate(projected):
            if proj_g.opacity < 0.01: # Skip invisible/transparent
                continue
                
            # Get the center of the projected Gaussian in pixel coordinates
            cx, cy = proj_g.center
            
            # Check bounds
            if not (0 <= cx < mask.shape[1] and 0 <= cy < mask.shape[0]):
                continue
                
            # Get the label at the center of the Gaussian in the 2D mask
            label_id = mask[int(cy), int(cx)]
            
            # Accumulate vote
            votes[i, label_id] += 1
            visibility_counts[i] += 1
            
    # Step 4: Determine final labels via voting
    final_labels = np.full(N, -1, dtype=int)
    for i in range(N):
        if visibility_counts[i] > 0:
            # Majority vote
            final_labels[i] = np.argmax(votes[i])
        else:
            # Gaussian not visible in any view with a valid mask
            final_labels[i] = -1 # Or keep as unsegmented
            
    return final_labels

def render_scene(gaussians, camera_matrix):
    """Standard differentiable rasterization of 3D Gaussians to 2D image."""
    # Implementation details of 2DGS rasterization are assumed to be available
    # from the parent method's context.
    pass

def sam_segment(image, prompts):
    """Call SAM model with image and point prompts."""
    pass

def get_corresponding_prompts(ref_prompts, ref_view, target_view):
    """
    Map point prompts from reference view to target view.
    This typically involves projecting the 3D points implied by the 
    reference prompts (if depth is known) or using feature matching.
    The source notes do not specify the exact mechanism for generating 
    prompts for other views, only that they are generated.
    [G] A common approach is to use the inverse projection of the 
    reference points if depth is available, or rely on user input.
    Here we assume a helper exists.
    """
    pass

def project_gaussians(gaussians, camera_matrix):
    """
    Project 3D Gaussians to 2D screen space.
    Returns list of objects with 'center' (x,y) and 'opacity'.
    """
    pass
```

**Parameters:**
*   **Point Prompts:** User-defined 2D coordinates on the reference view.
*   **SAM Model:** Segment Anything Model used for 2D segmentation [S1].
*   **Texture Attribute:** Used to uniformly represent non-stylized and stylized scenes, enabling color editing after stylization [S1].

**Edge Cases and Pitfalls:**
*   **Inefficiency:** The method cannot immediately generate a 3D segmentation from 2D masks created with point prompts on a single reference view. It may require multiple rounds of segmentation for certain scenes, which is inefficient [S1].
*   **Visibility:** Gaussians not visible in any training view with a valid mask will not be segmented.

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$, the number of training views $V$, and the cost of SAM inference. The voting step is $O(N \cdot V)$.
*   **Memory Complexity:** $O(N \cdot K)$ for storing votes, where $K$ is the number of segments.
*   **Strengths:** Training-free; allows real-time partial scene editing (PSE) and non-photorealistic scene editing (NPSE) with high granularity [S1]. Supports post-stylization color and lighting edits via the textures attribute and Blinn-Phong model, unlike StyleRF-VolVis [S1].
*   **Limitations:** Inefficient for scenes requiring multiple segmentation rounds; relies on SAM accuracy [S1].

## Background
The method is part of the TexGS-VolVis framework, introduced in 2025 [S1]. It builds on training-free segmentation strategies inspired by SAGD [25] and multi-view label voting [25]. It addresses the limitations of StyleRF-VolVis, which could not apply distinct edits to different parts of a basic scene or adjust lighting after training [S1].

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
