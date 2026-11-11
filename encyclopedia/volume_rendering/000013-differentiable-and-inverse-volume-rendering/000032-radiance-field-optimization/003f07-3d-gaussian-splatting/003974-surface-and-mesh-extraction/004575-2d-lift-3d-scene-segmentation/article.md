# 2D-Lift-3D Scene Segmentation

Also called: 2D-Lift-3D Segmentation for Partial Scene Editing, PSE with 2D-lift-3D segmentation

## Overview
This method segments 3D Gaussian primitives in a volume visualization scene by lifting 2D segmentation masks from reference views into 3D space. It solves the problem of applying distinct editing effects (color, opacity, lighting) to specific regions of a scene represented by millions of Gaussians. Unlike sibling methods that optimize learnable feature vectors for segmentation [S1], this approach is training-free, relying on multi-view label voting to assign labels to primitives.

## How it works
**Inputs and Outputs:**
*   **Input:** A trained set of $N$ 3D Gaussian primitives $\mathcal{G} = \{g_1, \dots, g_N\}$; a set of training views with known camera poses; a reference view image rendered from the Gaussian splatting pipeline; user-specified point prompts on the reference view.
*   **Output:** A label assignment $L_i \in \{0, \dots, K\}$ for each Gaussian primitive $g_i$, where $K$ is the number of segmented parts.

**Data Structures:**
*   **Gaussian Primitive $g_i$:** Defined by center $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$, opacity $o_i$, and spherical harmonics coefficients $\mathbf{c}_i$ [Parent].
*   **Label Map:** A 1D array of size $N$ storing the final assigned label for each primitive.

**Steps:**
1.  **Render Reference View:** Render an image from the current Gaussian representation using the standard differentiable rendering pipeline [Parent].
2.  **Generate 2D Mask:** Use the Segment Anything Model (SAM) with user-specified point prompts on the rendered reference view to generate a 2D segmentation mask [S1].
3.  **Generate Multi-View Masks:** Use SAM with the same point prompts (mapped to corresponding locations) on all available training views to generate 2D segmentation masks for each view [S1].
4.  **Label Propagation:** For each training view, project all $N$ Gaussian primitives into 2D image space. Assign a temporary label to each Gaussian based on which 2D segmentation mask pixel it overlaps with. If a Gaussian overlaps multiple masks or none, handle according to voting rules [S1].
5.  **Multi-View Label Voting:** For each Gaussian primitive $g_i$, collect the labels assigned from all training views. Determine the final label $L_i$ by majority voting among the views that successfully segmented the primitive [S1].
6.  **Decomposition:** Group Gaussians by their final label $L_i$ to form distinct 3D segments. These segments can then be independently edited using Partial Scene Editing (PSE) techniques [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def segment_gaussians(gaussians, training_views, reference_view, point_prompts):
    """
    gaussians: List of N Gaussian primitives.
    training_views: List of camera poses and intrinsics for training views.
    reference_view: Camera pose and intrinsics for the reference view.
    point_prompts: List of 2D points on the reference view for SAM.
    """
    N = len(gaussians)
    final_labels = [-1] * N  # Initialize with -1 (unassigned)
    
    # Step 1 & 2: Generate 2D mask for reference view
    # Note: In practice, this requires rendering the image first.
    ref_image = render_scene(gaussians, reference_view)
    ref_mask = sam_segment(ref_image, point_prompts)
    
    # Step 3: Generate masks for all training views
    # This assumes point prompts can be projected to training views or 
    # the user provides prompts for each view. The source implies 
    # "corresponding parts across all training views" are segmented.
    training_masks = []
    for view in training_views:
        view_image = render_scene(gaussians, view)
        # Assuming prompts are adapted or SAM is run on corresponding regions
        view_mask = sam_segment(view_image, get_prompts_for_view(view, point_prompts))
        training_masks.append(view_mask)
        
    # Step 4 & 5: Label Propagation and Voting
    for i in range(N):
        g = gaussians[i]
        votes = {}
        
        for view_idx, view in enumerate(training_views):
            mask = training_masks[view_idx]
            
            # Project Gaussian center to 2D
            u, v = project_point(g.center, view.projection_matrix, view.intrinsics)
            
            # Check if projection is within image bounds
            if not is_in_bounds(u, v, mask.shape):
                continue
                
            # Check if the pixel is part of the segmentation
            label = mask[v, u]
            
            # If the pixel is background (label 0) or valid segment
            if label > 0:
                votes[label] = votes.get(label, 0) + 1
                
        # Step 5: Majority Vote
        if votes:
            final_labels[i] = max(votes, key=votes.get)
        else:
            final_labels[i] = 0  # Assign to background if not segmented in any view
            
    return final_labels
```

**Parameters:**
*   **Point Prompts:** User-defined 2D coordinates on the reference view.
*   **SAM Model:** The specific implementation of Segment Anything Model used for mask generation [S1].

**Edge Cases and Pitfalls:**
*   **Inefficiency:** The method cannot immediately generate a 3D segmentation from a single 2D mask created by point prompts. It may require multiple rounds of segmentation for certain scenes, which is inefficient [S1].
*   **Occlusion:** If a Gaussian is occluded in all training views, it may not receive a label.
*   **Boundary Precision:** Projecting 3D Gaussians to 2D pixels can lead to ambiguous assignments if the Gaussian spans multiple segments.

## Operating characteristics
*   **Time Complexity:** Dominated by the rendering of $V$ training views and the inference of SAM for each view. Projection and voting are $O(N \cdot V)$.
*   **Memory Complexity:** $O(N)$ for storing labels. Additional memory for storing $V$ segmentation masks.
*   **Strengths:** Training-free; does not require optimizing learnable feature vectors per Gaussian; enables real-time partial scene editing (PSE) after segmentation [S1].
*   **Limitations:** Relies on the accuracy of SAM and the availability of training views; inefficient for scenes requiring multiple segmentation rounds [S1].
*   **When to use:** When rapid, interactive segmentation of a pre-trained Gaussian scene is needed without retraining or optimizing features.
*   **When not to use:** When high precision is required for complex occlusions or when training views are insufficient.

## Background
The method was introduced in the context of TexGS-VolVis, a framework for expressive scene editing in volume visualization using 2D Gaussian Splatting [S1]. It builds on prior work in 2D-lift-3D segmentation [5, 8, 25, 35] and specifically adapts the training-free strategy from SAGD [25] for Gaussian primitives. It addresses the limitation of previous methods like StyleRF-VolVis, which could not apply distinct edits to different parts of a basic scene [S1].

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
