# Cross-view Self-enhancement

Also called: CSE

## Overview
Cross-view Self-enhancement (CSE) is a training strategy for open-vocabulary neural radiance fields that mitigates view-inconsistent semantic supervision by replacing noisy 2D vision-language model outputs with 3D-consistent pseudo-labels generated from the model itself [S1]. It addresses the problem where 2D CLIP models produce inconsistent object identities across different perspectives, which degrades multi-view consistent rendering [S1]. CSE distinguishes itself from direct 2D supervision by leveraging the inherent 3D consistency of the NeRF representation to iteratively refine and synthesize semantic maps, thereby reducing ambiguity in semantic class assignment [S1].

## How it works
CSE operates by generating two types of pseudo-relavancy maps: updated maps for training views and synthesized maps for novel views, both regularized by the RSR strategy [S1]. These maps serve as supervision targets for the semantic field, replacing the original 2D CLIP outputs [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Trained semantic field $\Gamma_\theta$.
    *   Training view poses $I = \{I_i \in \mathbb{R}^{H \times W \times 3} \mid i \in \{1, \dots, N\}\}$ [S1].
    *   RSR regularizer $\Phi$ (referenced from RSR article) [S1].
*   **Outputs:**
    *   Refined pseudo relevancy maps $\bar{S}_i$ for training views [S1].
    *   Novel pseudo relevancy maps $S^+_i$ for unseen viewpoints [S1].
    *   CSE Loss $L_{CSE}$ used to update the semantic field [S1].

### Data Structures
*   **Relevancy Map:** A 2D tensor of size $H \times W$ containing logits or probabilities for each semantic class at each pixel.
*   **Pseudo Map:** Same structure as the relevancy map, but derived from rendered NeRF outputs and regularized [S1].

### Steps
1.  **Iterative Pseudo Map Update:**
    *   After training the semantic field for a few iterations, render semantic relevancy maps $\hat{S}_i$ for all training view poses $I$ [S1].
    *   Apply the RSR regularizer $\Phi$ to these rendered maps to generate refined pseudo relevancy maps $\bar{S}_i$:
        $$ \bar{S}_i = \Phi(\hat{S}_i) $$
        where $\hat{S}_i$ is the prediction from the learned semantic field [S1].
    *   Replace the 2D CLIP-derived relevancy maps with these updated pseudo maps $\bar{S}_i$ for semantic field learning [S1].
    *   Update the pseudo maps at a regular interval $N$ to incorporate more view-consistent supervision as training progresses [S1].

2.  **Novel Pseudo Map Synthesis:**
    *   Generate novel viewpoints (unseen poses) distinct from the training set [S1]. The number of novel views $N^+$ is set to 60 in the source implementation [S1].
    *   Render semantic relevancy maps $\hat{S}^+_i$ for these novel viewpoints [S1].
    *   Apply the RSR regularizer to these novel rendered maps to obtain fine edges and integral objects, producing novel pseudo relevancy maps $S^+_i$ [S1].

3.  **Loss Computation:**
    *   Compute the Pseudo Map Update Loss ($L_{PMU}$) by minimizing the cross-entropy between the rendered prediction $\hat{S}(r)$ and the updated pseudo map $\bar{S}(r)$ for all rays $r$ sampled from the training views:
        $$ L_{PMU} = \sum_{r} L_{ce}(\hat{S}(r), \bar{S}(r)) $$
        where $L_{ce}$ is the cross-entropy loss [S1].
    *   Compute the Pseudo Map Synthesis Loss ($L_{PMS}$) by supervising the novel rendered relevancy maps $\hat{S}^+_i$ using the novel pseudo maps $S^+_i$ in a per-ray manner:
        $$ L_{PMS} = \sum_{r^+} L_{ce}(\hat{S}^+(r^+), S^+(r^+)) $$
        where $r^+$ indicates rays sampled from the novel pseudo maps [S1].
    *   Combine the losses to form the total CSE loss:
        $$ L_{CSE} = L_{PMU} + L_{PMS} $$
        [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cross_view_self_enhancement(semantic_field, training_poses, rsr_regularizer, update_interval=10, num_novel_views=60):
    """
    Implements the Cross-view Self-enhancement (CSE) strategy.
    
    Args:
        semantic_field: The trained semantic NeRF model.
        training_poses: List of camera poses for training views.
        rsr_regularizer: Function implementing RSR regularization.
        update_interval: Number of iterations between pseudo map updates.
        num_novel_views: Number of novel viewpoints to synthesize.
        
    Returns:
        cse_loss: The computed CSE loss value.
    """
    # 1. Iterative Pseudo Map Update
    # Render semantic relevancy maps for training views
    # Note: Rendering involves volume rendering of semantic features f_k
    # as per the parent article's semantic rendering formula.
    rendered_maps_training = []
    for pose in training_poses:
        # Render semantic logits for all rays in the view
        # This produces a map of shape (H, W, num_classes)
        rendered_map = semantic_field.render_semantic(pose)
        rendered_maps_training.append(rendered_map)
        
    # Regularize with RSR to get refined pseudo maps
    pseudo_maps_training = []
    for rendered_map in rendered_maps_training:
        # Phi is the RSR regularizer
        refined_map = rsr_regularizer(rendered_map)
        pseudo_maps_training.append(refined_map)
        
    # Compute L_PMU
    # Sample rays from the training views and compare rendered vs pseudo
    l_pmu = 0.0
    for pose, pseudo_map in zip(training_poses, pseudo_maps_training):
        # Sample rays r from this view
        rays = sample_rays_from_view(pose)
        for r in rays:
            # Get rendered semantic logit for ray r
            rendered_logit = semantic_field.get_semantic_logit(r)
            # Get pseudo logit for the corresponding pixel
            pseudo_logit = pseudo_map[r.pixel_coords]
            # Cross-entropy loss
            l_pmu += cross_entropy(rendered_logit, pseudo_logit)
            
    # 2. Novel Pseudo Map Synthesis
    # Generate novel viewpoints
    novel_poses = generate_novel_poses(num_novel_views, training_poses)
    
    rendered_maps_novel = []
    for pose in novel_poses:
        rendered_map = semantic_field.render_semantic(pose)
        rendered_maps_novel.append(rendered_map)
        
    # Regularize novel maps
    pseudo_maps_novel = []
    for rendered_map in rendered_maps_novel:
        refined_map = rsr_regularizer(rendered_map)
        pseudo_maps_novel.append(refined_map)
        
    # Compute L_PMS
    l_pms = 0.0
    for pose, pseudo_map in zip(novel_poses, pseudo_maps_novel):
        rays = sample_rays_from_view(pose)
        for r in rays:
            rendered_logit = semantic_field.get_semantic_logit(r)
            pseudo_logit = pseudo_map[r.pixel_coords]
            l_pms += cross_entropy(rendered_logit, pseudo_logit)
            
    # 3. Total CSE Loss
    cse_loss = l_pmu + l_pms
    
    return cse_loss

def generate_novel_poses(num_views, training_poses):
    """
    Generates novel camera poses distinct from training poses.
    Implementation detail: Not given in the sources.
    Typically involves sampling random poses within the scene bounds
    or interpolating between training poses.
    """
    # Placeholder for pose generation logic
    pass

def sample_rays_from_view(pose):
    """
    Samples rays for a given camera pose.
    Implementation detail: Not given in the sources.
    Standard ray casting from camera center through pixel centers.
    """
    pass

def cross_entropy(logit, target):
    """
    Computes cross-entropy loss between predicted logit and target.
    """
    # Standard cross-entropy calculation
    pass
```

### Parameters
*   **Update Interval ($N$):** The frequency at which pseudo maps are regenerated and updated. Not given in the sources.
*   **Number of Novel Views ($N^+$):** Set to 60 in the source implementation [S1].

### Edge Cases and Pitfalls
*   **View-Inconsistent 2D CLIP:** Direct use of 2D CLIP maps leads to view-inconsistent relevancy maps, challenging multi-view consistent rendering [S1]. CSE specifically addresses this by replacing them with 3D-consistent pseudo maps [S1].
*   **Precision:** The regularization $\Phi$ (RSR) is critical for producing "fine edges and integral objects" in the pseudo maps [S1]. Without RSR, the pseudo maps may lack the necessary structural coherence to serve as effective supervision [S1].

## Operating characteristics
*   **Time Complexity:** CSE adds computational overhead during training due to the rendering of semantic maps for both training and novel views, and the application of the RSR regularizer. The iterative update process requires periodic re-rendering and regularization [S1].
*   **Memory Complexity:** Requires storing the rendered semantic maps for training and novel views, as well as the pseudo maps [S1].
*   **Strengths:** Significantly improves multi-view semantic consistency and reduces ambiguity caused by 2D CLIP inconsistencies [S1]. Enhances the 3D semantic understanding capability of the model [S1].
*   **Limitations:** Relies on the quality of the initial semantic field and the RSR regularizer. If the initial field is poor, the pseudo maps may be inaccurate [S1].
*   **When to use:** When training open-vocabulary NeRFs with 2D vision-language models (like CLIP) that produce noisy or inconsistent semantic labels across views [S1].
*   **When not to use:** When high-quality 3D ground truth semantic labels are available, or when computational resources are extremely limited [S1].

## Background
CSE was proposed in the context of OV-NeRF, an open-vocabulary neural radiance field method that integrates vision and language foundation models for 3D semantic understanding [S1]. The method was introduced to address the specific challenge of view-inconsistent relevancy maps generated by 2D CLIP models [S1]. It builds upon the RSR (Regularized Semantic Rendering) strategy to refine the pseudo labels [S1]. The approach was validated on indoor scene datasets such as Replica and ScanNet [S1].

## Sources
[S1] OV-NeRF: Open-Vocabulary Neural Radiance Fields With Vision and Language Foundation Models for 3D Semantic Understanding, https://doi.org/10.1109/tcsvt.2024.3439737, fetched 2026-10-26
