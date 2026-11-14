# NeRF-GT Baseline Training Setup
Also called: NeRF-GT
## Overview
The NeRF-GT baseline is a training configuration for Neural Radiance Fields that utilizes High Dynamic Range (HDR) ground truth images as input data [S1]. It serves as an upper-bound reference for methods that synthesize HDR views from Low Dynamic Range (LDR) inputs [S1]. Unlike standard NeRF training which often struggles with the high dynamic range of scenes even when using tone-mapped losses, this setup directly trains on HDR views to establish a performance ceiling [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A dataset of HDR views (or consistently exposed LDR views) and corresponding camera poses.
*   **Output:** A trained NeRF model capable of rendering HDR radiance fields.

**Data Structures:**
*   **Training Data:** Pairs of $(I_{hdr}, \mathbf{P})$, where $I_{hdr}$ is the HDR image and $\mathbf{P}$ is the camera pose.
*   **NeRF Model:** Standard MLP-based radiance field representation.

**Steps:**
1.  **Train the NeRF model** using the HDR views as the target supervision signal [S1].
2.  **Render** the scene to obtain predicted HDR pixel values, denoted as $C_{pred}$ [S1].
3.  **Tone-map** the predicted HDR pixel values $C_{pred}$ into LDR pixel values, denoted as $C_{pred}^{LDR}$ [S1].
4.  **Tone-map** the ground truth HDR views $I_{hdr}$ into LDR pixel values, denoted as $I_{gt}^{LDR}$ [S1].
5.  **Compute the loss** by comparing the tone-mapped predictions $C_{pred}^{LDR}$ to the tone-mapped ground truth $I_{gt}^{LDR}$ [S1].

**Pitfalls:**
*   Training directly on HDR views can fail to encode all areas of a scene well due to the high dynamic range, even when tone-mapped colors are used for loss calculation [S1]. This makes it difficult to ensure uniform quality across the entire scene [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_gt_baseline(nerf_model, hdr_dataset, optimizer, num_epochs):
    """
    Trains a NeRF model using HDR ground truth views.
    """
    for epoch in range(num_epochs):
        for batch in hdr_dataset:
            # Get ground truth HDR views and camera poses
            hdr_views = batch.hdr_images
            poses = batch.poses
            
            # Sample rays and render predicted HDR colors
            # ray_batch contains ray origins and directions
            ray_batch = sample_rays(poses, batch_size=256)
            predicted_hdr = nerf_model(ray_batch)
            
            # Tone-map predicted HDR values to LDR
            # Assuming a standard tone-mapping operator (e.g., Reinhard or ACES)
            predicted_ldr = tone_map(predicted_hdr)
            
            # Tone-map ground truth HDR views to LDR
            gt_ldr = tone_map(hdr_views)
            
            # Compute loss between tone-mapped predictions and ground truth
            loss = compute_loss(predicted_ldr, gt_ldr)
            
            # Backpropagate and update
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```
## Subtopics
## Operating characteristics
*   **Strengths:** Provides an upper-bound performance metric for HDR synthesis methods [S1].
*   **Limitations:** Difficult to ensure all areas of a scene are encoded well due to the high dynamic range [S1].
*   **Usage:** Used as a baseline comparison for novel HDR view synthesis methods [S1].
## Background
The NeRF-GT baseline is described in the context of "HDR-NeRF: High Dynamic Range Neural Radiance Fields" [S1]. The authors note that no existing dataset was appropriate for novel HDR view synthesis, leading them to collect a new dataset using Blender's Cycles path-tracer for 3D models and real-world captures [S1].
## Sources
[S1] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
