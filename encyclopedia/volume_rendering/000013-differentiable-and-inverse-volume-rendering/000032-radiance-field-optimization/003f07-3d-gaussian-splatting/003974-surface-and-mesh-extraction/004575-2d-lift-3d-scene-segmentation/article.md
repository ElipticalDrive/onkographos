# 2D-Lift-3D Scene Segmentation

Also called: 2D-Lift-3D Segmentation for Partial Scene Editing, PSE with 2D-lift-3D segmentation

## Overview
This method segments 3D Gaussian primitives in a volume visualization scene by lifting 2D segmentation masks from a reference view into 3D space. It solves the problem of partial scene editing, allowing users to apply distinct color, opacity, and lighting effects to specific regions of a scene represented by Gaussian splats. Unlike sibling methods that may require learnable feature vectors or pre-defined meshes, this approach uses a training-free strategy based on multi-view label voting [S1].

## How it works
**Inputs:**
*   A set of 3D Gaussian primitives $\mathcal{G} = \{g_1, \dots, g_N\}$, where each primitive $g_i$ has center $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $o_i$, and spherical harmonic coefficients for color $\mathbf{c}_i$ [S1].
*   A set of training views $\mathcal{V}_{train}$ with known camera poses.
*   A reference view $v_{ref} \in \mathcal{V}_{train}$ (or an arbitrary view rendered from the current state).
*   User-specified point prompts on the reference view.

**Outputs:**
*   A label map $L: \mathcal{G} \rightarrow \{1, \dots, K\}$, assigning each Gaussian primitive to one of $K$ semantic segments.

**Data Structures:**
*   **Gaussian List:** Array of primitives. Each primitive stores its 3D position, rotation (quaternion), scale, opacity, and SH coefficients.
*   **Label Buffer:** An integer array of size $N$ (number of Gaussians), initialized to $-1$ (unassigned).

**Steps:**

1.  **Render Reference Image:** Render an image from the current Gaussian scene using the standard differentiable rasterization pipeline defined in [3D Gaussian Splatting](1795-3d-gaussian-splatting.md) for the reference view [S1].
2.  **Generate 2D Mask:** Use the Segment Anything Model (SAM) with user-specified point prompts on the rendered reference image to generate a 2D segmentation mask $M_{2D}$ [S1].
3.  **Generate Multi-View Masks:** For each training view $v \in \mathcal{V}_{train}$, render the image and apply SAM with corresponding point prompts (or propagated prompts) to generate 2D masks $M_{2D}^v$ [S1].
4.  **Label Propagation:** For each training view $v$:
    a. Identify the set of Gaussian primitives $\mathcal{G}_v$ that contribute to the pixels covered by the mask $M_{2D}^v$. A Gaussian $g_i$ contributes if its projected 2D center $\mathbf{p}'_i$ falls within the mask region and its opacity $\alpha_i$ is significant.
    b. Assign a temporary label $l_v$ to all $g_i \in \mathcal{G}_v$ [S1].
5.  **Multi-View Label Voting:** For each Gaussian primitive $g_i$:
    a. Collect all labels assigned to $g_i$ across all views where it was visible and covered by a mask.
    b. Determine the final label $L(g_i)$ by majority voting among the collected labels. If no views cover $g_i$ or voting is inconclusive, $g_i$ remains unassigned or retains a default background label [S1].
6.  **Decomposition:** Group the Gaussian primitives by their final labels $L(g_i)$ to form distinct 3D segments [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def segment_gaussians(gaussians, training_views, reference_view, point_prompts):
    """
    Segments 3D Gaussians using 2D-lift-3D segmentation.
    
    Args:
        gaussians: List of Gaussian primitives.
        training_views: List of camera poses for training views.
        reference_view: Camera pose for the reference view.
        point_prompts: List of 2D points for SAM on the reference view.
        
    Returns:
        labels: Array of integer labels for each Gaussian.
    """
    N = len(gaussians)
    labels = np.full(N, -1, dtype=int) # -1 indicates unassigned
    
    # Step 1 & 2: Generate 2D mask on reference view
    ref_image = render_image(gaussians, reference_view)
    ref_mask = sam_segment(ref_image, point_prompts)
    
    # Step 3: Generate masks for all training views
    # Note: In practice, point prompts for other views may need to be 
    # projected or manually specified. The source implies SAM is used 
    # for "corresponding parts across all training views" [S1].
    train_masks = []
    for view in training_views:
        view_image = render_image(gaussians, view)
        # Assuming prompts are available or propagated for this view
        view_mask = sam_segment(view_image, get_prompts_for_view(view, point_prompts))
        train_masks.append((view, view_mask))
        
    # Step 4 & 5: Label Propagation and Voting
    # We accumulate votes for each Gaussian
    vote_counts = np.zeros((N, len(set([l for m in train_masks for l in range(1, 10)]))), dtype=int) 
    # Simplified: assume labels are 1..K. In reality, SAM outputs masks, 
    # we need to map mask regions to labels.
    
    # Let's assume SAM returns a mask with distinct regions labeled 1..K
    # We iterate through each view's mask
    for view, mask in train_masks:
        # Project all Gaussians to this view's 2D space
        projected_gaussians = project_to_2d(gaussians, view)
        
        for i, g_proj in enumerate(projected_gaussians):
            # Check if Gaussian i is visible and covered by the mask
            if is_visible(g_proj) and is_in_mask(g_proj.center, mask):
                # Determine the label of the mask region at this pixel
                # SAM might return a single binary mask or multi-instance.
                # Assuming we map the mask to a specific semantic label 'l'
                # for this specific segmentation task.
                # If SAM returns multiple objects, we might need to iterate 
                # over unique mask values or user-defined regions.
                # For simplicity, assume the mask corresponds to one segment 'l'.
                l = get_mask_label(mask, g_proj.center) 
                if l != -1:
                    vote_counts[i, l] += 1
                    
    # Step 5: Majority Voting
    final_labels = np.argmax(vote_counts, axis=1)
    
    # Handle ties or no votes
    max_votes = np.max(vote_counts, axis=1)
    no_vote_mask = max_votes == 0
    if np.any(no_vote_mask):
        # Assign default label or leave as -1
        final_labels[no_vote_mask] = 0 
        
    return final_labels
```

**Parameters:**
*   **SAM Model:** Used for 2D segmentation. Requires point prompts [S1].
*   **Training Views:** The set of views used for label voting. More views generally improve robustness [S1].

**Edge Cases and Pitfalls:**
*   **Inefficiency:** The method cannot immediately generate a 3D segmentation from a single 2D mask created with point prompts. It may require multiple rounds of segmentation for certain scenes, which is inefficient [S1].
*   **Visibility:** Gaussians occluded in all training views will not receive labels.
*   **Prompt Accuracy:** Inaccurate point prompts can lead to incorrect 2D masks, propagating errors to the 3D segmentation [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by rendering all training views and running SAM inference for each. Rendering is $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels. SAM inference is $O(P^2)$ or similar depending on the model. Label voting is $O(N \cdot V)$ where $V$ is the number of views.
*   **Memory Complexity:** $O(N)$ to store labels. Additional memory for rendering buffers and SAM features.
*   **Strengths:** Training-free; does not require optimizing learnable feature vectors per Gaussian [S1]. Enables flexible partial scene editing (PSE) and non-photorealistic scene editing (NPSE) with texture attributes [S1].
*   **Limitations:** Relies on the accuracy of SAM and user prompts. Inefficient for scenes requiring many segmentation rounds. Cannot adjust light direction after training in some baseline methods, but TexGS-VolVis supports this via Blinn-Phong shading [S1].
*   **When to use:** When real-time partial editing is needed and the scene is represented by 3D Gaussians. When a training-free approach is preferred over optimizing additional features.

## Background
The method is introduced in the context of **TexGS-VolVis**, a framework for expressive scene editing in volume visualization using 2D Gaussian Splatting [S1]. It builds upon the concept of 2D-lift-3D segmentation [5,8,25,35] to enable partial scene editing (PSE) and non-photorealistic scene editing (NPSE) [S1]. Unlike StyleRF-VolVis, which only supports color editing before stylization, TexGS-VolVis uses a textures attribute to support post-stylization color and lighting edits [S1]. The segmentation strategy is inspired by SAGD [25] and uses multi-view label voting [25].

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
