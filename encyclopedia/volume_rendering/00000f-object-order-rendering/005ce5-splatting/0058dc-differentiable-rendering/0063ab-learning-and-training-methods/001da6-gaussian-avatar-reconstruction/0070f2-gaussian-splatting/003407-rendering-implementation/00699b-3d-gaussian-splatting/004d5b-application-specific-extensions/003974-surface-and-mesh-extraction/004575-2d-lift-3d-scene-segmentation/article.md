# 2D-Lift-3D Scene Segmentation

Also called: 2D-Lift-3D Segmentation for Partial Scene Editing, PSE with 2D-lift-3D segmentation

## Overview
2D-Lift-3D Scene Segmentation is a training-free method for segmenting 3D Gaussian primitives in volumetric scenes by projecting 2D segmentation masks from a reference view into 3D space [S1]. It solves the problem of partial scene editing (PSE) by decomposing a scene into distinct Gaussian groups, allowing independent color, opacity, and lighting adjustments to specific parts [S1]. Unlike methods that optimize learnable feature vectors for each Gaussian [S1], this approach relies on multi-view label voting to assign labels without additional training. It sets itself apart from sibling mesh extraction methods by preserving the volumetric Gaussian representation rather than converting it to a surface mesh [Parent].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained 3D Gaussian Splatting model $\mathcal{G} = \{G_i\}_{i=1}^M$ [Parent]; a set of training views $\mathcal{V}$; a reference view $v_{ref} \in \mathcal{V}$; user-specified point prompts on $v_{ref}$; the Segment Anything Model (SAM) [S1].
*   **Output:** A label assignment $L_i \in \{1, \dots, K\}$ for each Gaussian primitive $G_i$, where $K$ is the number of segmented parts [S1].

**Data Structures:**
*   **Gaussian Primitive $G_i$:** Defined by position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and color attributes (including textures for stylized scenes) [Parent, S1].
*   **Label Map:** An array of size $M$ storing the integer label for each Gaussian.

**Steps:**
1.  **Render Reference View:** Render an image $I_{ref}$ from the reference view $v_{ref}$ using the current Gaussian model [S1].
2.  **Generate 2D Mask:** Use SAM with user-specified point prompts on $I_{ref}$ to generate a 2D segmentation mask $M_{ref}$ [S1].
3.  **Generate Multi-View Masks:** For every training view $v \in \mathcal{V}$:
    a. Render the image $I_v$ [S1].
    b. Use SAM with corresponding point prompts (derived from the 3D geometry or transferred from $v_{ref}$) to generate a 2D mask $M_v$ [S1]. *Note: The sources state "corresponding parts across all training views" are generated using SAM [S1]. In practice, this often involves projecting the 2D mask or using consistent prompts.*
4.  **Label Propagation (Per View):** For each view $v$, assign a temporary label to every Gaussian $G_i$ that contributes to the pixels covered by $M_v$.
    *   A Gaussian $G_i$ is considered "hit" by mask $M_v$ if its projection onto view $v$ overlaps with the foreground pixels of $M_v$.
    *   Let $L_{i,v}$ be the label assigned to $G_i$ from view $v$. If $G_i$ is not hit, $L_{i,v} = \emptyset$.
5.  **Multi-View Label Voting:** Determine the final label $L_i$ for each Gaussian $G_i$ by aggregating labels from all views [S1].
    *   Collect all non-empty labels $\{L_{i,v} \mid v \in \mathcal{V}, L_{i,v} \neq \emptyset\}$.
    *   Select the label that appears most frequently (majority vote).
    *   If no view hits $G_i$, $L_i$ remains undefined or background.
6.  **Segmentation:** Group Gaussians by their final label $L_i$. Each group constitutes a segmented part of the scene [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def segment_gaussians(gaussians, training_views, reference_view, point_prompts_ref, sam_model):
    """
    gaussians: List of Gaussian primitives, each with position, covariance, etc.
    training_views: List of camera poses for all training views.
    reference_view: Camera pose for the reference view.
    point_prompts_ref: List of 2D points for SAM on the reference view.
    sam_model: The Segment Anything Model instance.
    """
    M = len(gaussians)
    # Initialize label counts for each Gaussian
    # Using a dictionary of counters for each Gaussian to handle voting
    label_votes = [Counter() for _ in range(M)]
    
    # Step 1 & 2: Process Reference View
    I_ref = render_scene(gaussians, reference_view)
    mask_ref = sam_model.predict(I_ref, point_prompts_ref)
    
    # We need to map 2D mask pixels back to Gaussians.
    # This requires knowing which Gaussians contribute to which pixels.
    # In standard 3DGS, this is done during rasterization.
    # Here we assume a function that returns the set of Gaussians 
    # contributing to the foreground of a mask for a given view.
    
    # Helper to get contributing gaussians for a mask
    def get_contributing_gaussians(view, mask):
        # Render depth/visibility or use rasterization indices
        # For implementation, one typically iterates Gaussians and checks 
        # if their projected center/bounding box intersects the mask.
        # A more precise method uses the rasterization output which lists 
        # the Gaussian indices sorted by depth for each pixel.
        contributing = set()
        # Pseudocode for precise intersection:
        # For each pixel (u,v) where mask[u,v] == 1:
        #   For each Gaussian G in the rasterization stack at (u,v):
        #     contributing.add(G.id)
        # Since we don't have the rasterizer here, we assume a function:
        return rasterizer.get_gaussians_in_mask(view, mask)

    # Step 3 & 4: Process All Views
    # Note: The source says "generate 2D segmentation masks for corresponding parts 
    # across all training views using SAM" [S1]. 
    # This implies we need prompts for other views. 
    # If prompts are not available, one might project the 3D points of the 
    # reference mask back to other views, or use the same semantic prompt.
    # We assume 'point_prompts_v' are available or derived.
    
    for view in training_views:
        I_v = render_scene(gaussians, view)
        
        # Determine prompts for this view. 
        # If not provided, this step requires additional logic not detailed in [S1].
        # We assume point_prompts_v is computed or provided.
        point_prompts_v = derive_prompts_for_view(point_prompts_ref, reference_view, view)
        
        mask_v = sam_model.predict(I_v, point_prompts_v)
        
        # Get Gaussians hit by this mask
        hit_gaussians = get_contributing_gaussians(view, mask_v)
        
        # Assign a temporary label (e.g., 1 for foreground, or specific ID)
        # The source implies distinct labels for different parts. 
        # If segmenting one part, label is 1. If multiple, SAM returns multiple masks.
        # Assuming binary segmentation for one part for simplicity:
        label_id = 1 
        
        for g_id in hit_gaussians:
            label_votes[g_id].update([label_id])

    # Step 5: Voting
    final_labels = []
    for i in range(M):
        if label_votes[i]:
            # Get the most common label
            final_labels.append(label_votes[i].most_common(1)[0][0])
        else:
            final_labels.append(0) # Background or unsegmented
            
    return final_labels
```

**Parameters:**
*   **SAM Model:** Pre-trained Segment Anything Model [S1].
*   **Point Prompts:** User-defined 2D coordinates on the reference view [S1].
*   **Textures Attribute:** Used in TexGS-VolVis to uniformly represent non-stylized and stylized scenes, enabling color editing after stylization [S1].

**Edge Cases and Pitfalls:**
*   **Inefficiency:** The method cannot immediately generate a 3D segmentation from 2D masks created with point prompts for all scenes. It may require multiple rounds of segmentation, which is inefficient for certain volumetric scenes [S1].
*   **Prompt Sensitivity:** Inaccurate or vague point prompts lead to erroneous 2D masks, propagating errors to the 3D segmentation [S1].
*   **View Consistency:** If SAM generates inconsistent masks across views for the same object, the voting mechanism may fail or produce noisy labels.

## Subtopics

### SA3D
SA3D is a framework for 3D object segmentation in radiance fields (NeRF, TensoRF, 3D-GS) that uses SAM to generate initial 2D masks from text or point prompts in a single view [S1]. It distinguishes itself by alternating between mask inverse rendering and cross-view self-prompting to iteratively refine the 3D mask, avoiding manual multi-view annotation [S1]. It builds on [2D-Lift-3D Scene Segmentation](11513-2d-lift-3d-scene-segmentation.md) and differs from [Multi-View Semantic Segmentation Strategy](18471-multi-view-semantic-segmentation-strategy.md) by using self-generated cross-view prompts rather than external multi-view labels.

### [Multi-View Semantic Segmentation Strategy](18471-multi-view-semantic-segmentation-strategy.md)
This strategy enables open-vocabulary querying and interactive scene editing in Volume Visualization (VolVis) by segmenting multi-view images to train independent, editable 3D Gaussian Splatting (iVR-GS) models for each object. It differs from direct 3DGS segmentation methods by maintaining method-agnostic segmentation and scalability, avoiding modifications to the underlying Gaussian structure. It addresses the view inconsistencies inherent in framewise 2D segmentation by treating multi-view images as a video sequence for tracking.

### Fine-Level Affinity-Based Segmentation
This method refines coarse segmentation by learning affinity features from 2D masks generated by SAM, enabling fine-grained structure distinction [S2]. It lifts 2D masks to a 3D affinity field and uses this field to subdivide coarse-level color-based segmentation results [S2]. It is employed when color-based segmentation cannot distinguish fine-grained structures within components that share similar colors [S2].

## Operating characteristics
*   **Time Complexity:** The method is training-free, so it does not require gradient descent optimization over the Gaussians for segmentation [S1]. The cost is dominated by rendering all training views and running SAM inference for each view [S1].
*   **Memory Complexity:** Requires storing the Gaussian primitives and the label array. No additional learnable features are stored per Gaussian, unlike feature-based methods [S1].
*   **Performance:** Supports real-time partial scene editing (PSE) during inference [S1]. Rendering times vary by dataset resolution (e.g., 29.68 ms for 256³ resolution, 153.69 ms for 1024³ resolution) [S1].
*   **Strengths:** Enables flexible and controllable VolVis scene editing by decomposing scenes into parts [S1]. Supports post-stylization color and lighting edits via the textures attribute and Blinn-Phong model, which previous methods like StyleRF-VolVis could not do [S1].
*   **Limitations:** Inefficient for scenes requiring multiple rounds of segmentation due to the inability to immediately generate 3D segmentation from 2D masks [S1]. Relies on the accuracy of SAM and user prompts [S1].

## Background
The method was introduced in the context of TexGS-VolVis, a framework for expressive scene editing in volume visualization using 2D Gaussian Splatting [S1]. It builds on prior work in 2D-lift-3D segmentation [5, 8, 25, 35] cited in [S1]. The approach was developed to address the limitations of StyleRF-VolVis, which lacked fine-grained partial editing capabilities [S1]. The technique is particularly relevant for scientific visualization where decomposing complex volumetric data into semantic parts is required for analysis and editing [S1, S2].

## Sources
[S1] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
[S2] VolSegGS: Segmentation and Tracking in Dynamic Volumetric Scenes via Deformable 3D Gaussians, https://doi.org/10.1109/tvcg.2025.3642516, fetched 2026-10-26
