# 2D-Lift-3D Scene Segmentation

Also called: 2D-Lift-3D Segmentation for Partial Scene Editing, PSE with 2D-lift-3D segmentation

## Overview
This method segments 3D Gaussian primitives in a volume visualization scene by lifting 2D segmentation masks from a reference view into 3D space. It solves the problem of partial scene editing, allowing distinct editing effects (color, opacity, lighting) to be applied to specific regions of a scene represented by Gaussian splats [S1]. Unlike sibling methods that may require learnable feature vectors or complex optimization to define regions, this approach uses a training-free strategy based on multi-view label voting [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained 3D Gaussian Splatting model (set of $N$ primitives $\{G_i\}$), a set of training views, and user-specified point prompts on a reference view.
*   **Output:** A label assignment for each Gaussian primitive, effectively partitioning the scene into distinct 3D segments.

**Data Structures:**
*   **Gaussian Primitives:** Defined by the parent article's structure: position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and spherical harmonics for color [S1].
*   **Labels:** An integer label $L_i$ assigned to each Gaussian $i$. Initially undefined or background.

**Steps:**
1.  **Render Reference View:** Render an image from the current Gaussian scene using the standard differentiable rendering pipeline [S1].
2.  **Generate 2D Mask:** Use the Segment Anything Model (SAM) with user-specified point prompts on the rendered reference view to generate a 2D segmentation mask $M_{ref}$ [S1].
3.  **Generate Multi-View Masks:** For each training view $v$, render the scene and use SAM with corresponding point prompts (or propagated prompts) to generate 2D segmentation masks $M_v$ [S1]. *Note: The source implies using SAM across all training views to gather evidence [S1].*
4.  **Label Propagation:** For each training view $v$, identify which 2D Gaussians (screen-space projections of 3D primitives) fall within the segmented region of $M_v$. Assign a temporary label to these Gaussians.
5.  **Multi-View Label Voting:** For each 3D Gaussian primitive $G_i$, count the number of views $v$ in which it was segmented. If the count exceeds a threshold (or based on the voting technique described in [25]), assign the final 3D label $L_i$ [S1].
6.  **Segmentation Result:** The set of Gaussians with label $L_i = k$ forms the $k$-th segment of the scene [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def segment_gaussians(gaussians, training_views, reference_view, point_prompts):
    """
    Segments 3D Gaussians using 2D-lift-3D technique.
    
    Args:
        gaussians: List of Gaussian primitives {G_i}.
        training_views: List of camera poses for training views.
        reference_view: Camera pose for the reference view.
        point_prompts: List of (x, y) coordinates for SAM prompts on reference view.
        
    Returns:
        labels: Dictionary mapping Gaussian index to segment label.
    """
    # Initialize labels
    labels = {i: -1 for i in range(len(gaussians))}
    vote_counts = {i: 0 for i in range(len(gaussians))}
    
    # Step 1 & 2: Reference View Segmentation
    ref_image = render_scene(gaussians, reference_view)
    ref_mask = sam_segment(ref_image, point_prompts)
    
    # Identify Gaussians visible in reference view and their screen-space coverage
    # This step requires projecting Gaussians to 2D and checking overlap with ref_mask
    ref_gaussian_indices = get_gaussians_in_mask(gaussians, reference_view, ref_mask)
    
    # Step 3 & 4: Multi-View Processing
    for view in training_views:
        # Render view
        view_image = render_scene(gaussians, view)
        
        # Generate mask for this view
        # Note: Point prompts for other views might need to be projected or manually specified
        # The source mentions "corresponding parts across all training views"
        view_mask = sam_segment(view_image, get_prompts_for_view(view, point_prompts))
        
        # Get Gaussians visible in this view and inside the mask
        visible_gaussians = get_visible_gaussians(gaussians, view)
        segmented_indices = get_gaussians_in_mask(gaussians, view, view_mask)
        
        # Step 5: Voting
        for idx in segmented_indices:
            if idx in visible_gaussians:
                vote_counts[idx] += 1
                
    # Finalize labels based on voting threshold
    # The source cites [25] for the voting technique. 
    # A simple implementation assumes if a Gaussian is segmented in any view, 
    # it belongs to the segment, or requires a majority vote.
    # Given the "inefficiency" note, it implies a binary decision or simple count.
    for idx in range(len(gaussians)):
        if vote_counts[idx] > 0: # Or some threshold T
            labels[idx] = 0 # Assuming single segment for simplicity; 
                            # multiple prompts would yield multiple labels
        else:
            labels[idx] = -1 # Background/Unsegmented
            
    return labels

def get_gaussians_in_mask(gaussians, camera, mask):
    """
    Projects Gaussians to 2D and checks if their center or significant area 
    falls within the mask.
    """
    projected = project_gaussians_2d(gaussians, camera)
    indices = []
    for i, proj in enumerate(projected):
        if is_inside_mask(proj, mask):
            indices.append(i)
    return indices
```

**Parameters:**
*   **SAM Model:** Used for 2D segmentation [S1].
*   **Point Prompts:** User-defined coordinates on the reference view [S1].
*   **Voting Threshold:** Not explicitly defined in the text, but implied by the "multi-view label voting technique [25]" [S1].

**Edge Cases and Pitfalls:**
*   **Inefficiency:** The method cannot immediately generate a 3D segmentation from a single 2D mask created with point prompts. It may require multiple rounds of segmentation for certain scenes, which is inefficient [S1].
*   **Prompt Accuracy:** Inaccurate or vague point prompts can lead to erroneous segmentation [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of training views and Gaussians. Rendering all views and running SAM for each view is computationally expensive. The voting step is linear in the number of Gaussians and views.
*   **Memory Complexity:** Requires storing masks for all training views and vote counts for all Gaussians.
*   **Strengths:** Training-free; does not require optimizing learnable feature vectors per Gaussian (unlike [5, 8] cited in [S1]); enables flexible partial scene editing (PSE) and non-photorealistic style editing (NPSE) [S1].
*   **Limitations:** Inefficient for complex scenes requiring multiple segmentation rounds; relies on the accuracy of SAM and user prompts; cannot adjust light direction after training in the same way as optimized methods (though TexGS-VolVis supports lighting adjustments via Blinn-Phong model [S1]).
*   **When to use:** When real-time partial editing is needed and the scene is represented by 3D Gaussians, and user interaction via point prompts is acceptable.
*   **When not to use:** When high efficiency is critical and multiple segmentation rounds are likely; when automatic segmentation without user prompts is required.

## Background
The method is introduced in the context of **TexGS-VolVis**, a framework for expressive scene editing in volume visualization using 2D Gaussian Splatting [S1]. It builds upon previous works like StyleRF-VolVis, which lacked partial editing capabilities and post-stylization lighting adjustments [S1]. The 2D-lift-3D segmentation technique is inspired by SAGD [25] and multi-view label voting [25]. It addresses the limitation of previous methods that could only apply edits to the entire scene or required complex optimization for region-specific edits [S1].

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
