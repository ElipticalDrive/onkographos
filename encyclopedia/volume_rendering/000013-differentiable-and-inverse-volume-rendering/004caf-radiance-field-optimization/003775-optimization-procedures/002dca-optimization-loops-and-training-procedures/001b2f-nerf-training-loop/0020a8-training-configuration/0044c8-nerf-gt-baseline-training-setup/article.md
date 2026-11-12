# NeRF-GT Baseline Training Setup
Also called: NeRF-GT, HDR-NeRF Baseline
## Overview
NeRF-GT is a baseline training protocol for Neural Radiance Fields that utilizes High Dynamic Range (HDR) input views to synthesize novel HDR imagery [S1]. It addresses the limitation of standard NeRF, which typically trains on Low Dynamic Range (LDR) images, by directly optimizing the radiance field against HDR ground truth [S1]. It differs from standard LDR-based training by explicitly handling high dynamic range data, though it faces challenges in encoding all scene areas uniformly due to the intensity variance [S1].
## How it works
The method trains a NeRF model directly on HDR views, treating them as the upper bound for performance [S1]. The core procedure involves predicting HDR pixel values and mapping them to LDR space for loss computation against tone-mapped ground truth [S1].

**Inputs:**
*   HDR training views (rendered or captured).
*   Camera poses and intrinsics for the training views.

**Outputs:**
*   Optimized NeRF model parameters (MLP weights, grid embeddings, etc.).

**Data Structures:**
*   **NeRF Model:** Standard MLP-based radiance field representation [S1].
*   **Training Data:** Set of HDR images $I_{hdr}$ and corresponding LDR ground truth $I_{ldr}^{gt}$ [S1].

**Steps:**
1.  **Training Initialization:** Initialize the NeRF model parameters.
2.  **Forward Pass:** For each training ray, query the NeRF to predict the HDR color value $C_{pred}^{hdr}$ [S1].
3.  **Tone Mapping (Prediction):** Apply a tone-mapping operator $T(\cdot)$ to the predicted HDR value to obtain the LDR prediction:
    $$ C_{pred}^{ldr} = T(C_{pred}^{hdr}) $$
    [S1]
4.  **Ground Truth Preparation:** Apply the same tone-mapping operator $T(\cdot)$ to the ground truth HDR image to obtain the LDR ground truth $C_{gt}^{ldr}$ [S1].
5.  **Loss Computation:** Compute the loss by comparing the tone-mapped prediction to the tone-mapped ground truth:
    $$ \mathcal{L} = \| C_{pred}^{ldr} - C_{gt}^{ldr} \| $$
    [S1]
6.  **Optimization:** Update NeRF parameters using gradient descent to minimize $\mathcal{L}$ [S1].

**Parameters:**
*   **Tone Mapping Operator:** The specific implementation of $T(\cdot)$ is not detailed in the sources, but consistency between prediction and ground truth is required [S1].
*   **Exposure:** If using LDR views with consistent exposures, they can be treated similarly, but HDR views are the primary focus for this baseline [S1].

**Edge Cases and Pitfalls:**
*   **Encoding Failure:** Training directly on HDR views can fail to encode all areas of a scene well due to the high dynamic range, even when using tone-mapped colors for loss calculation [S1]. This is a significant limitation of the baseline compared to methods that might use exposure blending or other HDR-specific techniques [S1].
*   **Dataset Availability:** No standard dataset is appropriate for novel HDR view synthesis, requiring custom datasets (e.g., rendered from 3D models or captured with HDR cameras) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_gt_baseline(nerf_model, hdr_dataset, optimizer, num_epochs):
    """
    Trains a NeRF model using HDR views as input.
    
    Args:
        nerf_model: The NeRF model instance.
        hdr_dataset: A dataset yielding (hdr_image, ldr_gt_image, camera_params) tuples.
        optimizer: The optimizer for updating nerf_model parameters.
        num_epochs: Number of training epochs.
    """
    for epoch in range(num_epochs):
        for hdr_image, ldr_gt_image, camera_params in hdr_dataset:
            # 1. Sample rays from the current view
            rays = sample_rays_from_view(hdr_image, camera_params)
            
            # 2. Forward pass to predict HDR colors
            # predict_hdr_color returns the predicted HDR color for each ray
            pred_hdr_colors = nerf_model.forward(rays)
            
            # 3. Tone-map predicted HDR colors to LDR
            # tone_map is a consistent tone-mapping operator
            pred_ldr_colors = tone_map(pred_hdr_colors)
            
            # 4. Get ground truth LDR colors (already tone-mapped in dataset or apply here)
            # Assuming ldr_gt_image is the tone-mapped ground truth for the corresponding view
            gt_ldr_colors = extract_ldr_from_gt(ldr_gt_image, rays)
            
            # 5. Compute loss
            loss = compute_loss(pred_ldr_colors, gt_ldr_colors)
            
            # 6. Backpropagate and update
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```
[G] `tone_map` and `extract_ldr_from_gt` are assumed to be consistent implementations of the tone-mapping operator $T(\cdot)$ mentioned in the source. The source does not specify the exact tone-mapping function, so a standard implementation (e.g., Reinhard, ACES) would be required for a complete implementation.
## Subtopics
## Operating characteristics
*   **Performance:** The method serves as an upper bound for performance when compared to vanilla NeRF trained on LDR images, as it utilizes more information (HDR) [S1].
*   **Limitations:** It struggles to encode all scene areas uniformly due to the high dynamic range, leading to potential quality issues in certain regions [S1].
*   **Data Requirements:** Requires HDR training data, which is not commonly available in standard datasets, necessitating custom data collection or rendering [S1].
## Background
The NeRF-GT baseline is described in the context of "HDR-NeRF: High Dynamic Range Neural Radiance Fields" [S1]. The authors note that no existing dataset is appropriate for novel HDR view synthesis, leading them to collect a new dataset using Blender's Cycles path-tracer for 3D models and HDR captures for real-world scenes [S1]. The method is presented as a comparison point to evaluate the effectiveness of their proposed HDR-NeRF approach against a direct training baseline [S1].
## Sources
[S1] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
