# Radiometric Self-Calibration

Also called: Color Calibration, Exposure and Sensor Bias Estimation, Gain and Bias Calibration, Response Curve Optimization

## Overview
Radiometric Self-Calibration optimizes radiometric parameters, specifically exposure and sensor bias, during volume reconstruction to correct for photometric errors arising from uncalibrated camera sensors or inconsistent lighting [S1, S2]. It differs from geometric self-calibration by focusing on intensity mapping rather than spatial pose, and from simple color correction by learning these parameters jointly with the volume representation [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images $\{I_{\text{rgb}}^{(k)}\}$, camera poses, and an initial volume representation (e.g., neural radiance field or voxel grid).
*   **Outputs:** Optimized volume parameters, per-camera gain $g$, per-camera bias $b$, and optionally per-camera background images $I_{\text{rgb}}^{(\text{bg})}$ [S2].

**Data Structures:**
*   **Per-camera parameters:** For each camera $k$, store a gain scalar $g_k$ and a bias scalar $b_k$ (or vectors per color channel) [S2].
*   **Background images:** Optional static per-camera RGB images $I_{\text{rgb}}^{(\text{bg})}$ [S2].

**Steps:**
1.  **Initialize:** Set initial values for gain $g=1$ and bias $b=0$ for all cameras [S2]. Initialize background images if used [S2].
2.  **Ray Marching:** For each training view, ray march through the volume to compute the accumulated color $I_{\text{rgb}}(\mathbf{p})$ and opacity $I_\alpha(\mathbf{p})$ for each pixel $\mathbf{p}$ [S2].
3.  **Radiometric Correction:** Apply the per-camera gain and bias to the accumulated volume color:
    $$ I_{\text{corrected}}(\mathbf{p}) = g I_{\text{rgb}}(\mathbf{p}) + b $$
    [S2].
4.  **Background Merging (Optional):** If background estimation is enabled, blend the corrected volume color with the static background image based on the exiting opacity:
    $$ \widehat{I}_{\text{rgb}}(\mathbf{p}) = (1 - I_\alpha(\mathbf{p})) I_{\text{rgb}}^{(\text{bg})}(\mathbf{p}) + I_{\text{corrected}}(\mathbf{p}) $$
    [S2].
5.  **Loss Computation:** Compute the photometric loss between the final rendered image $\widehat{I}_{\text{rgb}}(\mathbf{p})$ and the ground truth training image [S2].
6.  **Optimization:** Update volume parameters, gain $g$, bias $b$, and background images using an optimizer (e.g., Adam) [S2].

**Parameters:**
*   **Learning Rate for $g, b$:** $10^{-3}$ [S2].
*   **Learning Rate for Background Images:** $10^{-1}$ [S2].
*   **Response Curve Optimization:** Can be enabled to further improve quality, though specific functional form is not detailed in the provided notes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def radiometric_self_calibration(volume_model, training_data, num_iterations=500000):
    # Initialize per-camera gain and bias
    gains = {cam_id: 1.0 for cam_id in training_data.cameras}
    biases = {cam_id: 0.0 for cam_id in training_data.cameras}
    
    # Optional: Initialize background images
    backgrounds = {cam_id: None for cam_id in training_data.cameras}
    if use_background_estimation:
        for cam_id in training_data.cameras:
            backgrounds[cam_id] = torch.zeros_like(training_data.images[cam_id])
            
    optimizer = Adam([
        {'params': volume_model.parameters(), 'lr': 1e-4},
        {'params': list(gains.values()), 'lr': 1e-3},
        {'params': list(biases.values()), 'lr': 1e-3},
        {'params': list(backgrounds.values()), 'lr': 1e-1}
    ])
    
    for iteration in range(num_iterations):
        # Sample a batch of views
        batch = sample_batch(training_data, batch_size=16)
        
        total_loss = 0
        for cam_id, gt_image in batch:
            # Ray march to get volume color and opacity
            volume_color, opacity = volume_model.ray_march(cam_id)
            
            # Apply gain and bias
            corrected_color = gains[cam_id] * volume_color + biases[cam_id]
            
            # Merge with background if enabled
            if use_background_estimation and backgrounds[cam_id] is not None:
                bg_image = backgrounds[cam_id]
                rendered_image = (1 - opacity) * bg_image + corrected_color
            else:
                rendered_image = corrected_color
                
            # Compute loss
            loss = mse_loss(rendered_image, gt_image)
            total_loss += loss
            
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return volume_model, gains, biases, backgrounds
```

## Subtopics

### Geometric Self-Calibration
Optimizes geometric parameters, such as detector orientation and source position, during reconstruction to correct for calibration errors [S1]. It recovers from significant initial errors (e.g., 15 dB degradation from Gaussian noise in position/rotation) and outperforms baselines with bad initial calibration [S1]. Choose this when geometric poses are inaccurate or unknown [S1].

### Background Image Merging
Estimates a static per-camera background image $I_{\text{rgb}}^{(\text{bg})}$ to capture stationary objects outside the reconstruction volume, reducing artifacts caused by calibration errors or view-dependent effects [S2]. It merges the volume output with the background using opacity weighting [S2]. Choose this when static backgrounds are present and cause reconstruction artifacts [S2].

### Exposure Correction Strategy
Corrects inconsistent exposure across training views caused by auto-exposure behavior by mimicking the exposure setting of the closest source view at novel views [S3]. Unlike prior work that only corrects training views, this strategy generalizes to images rendered at any viewpoint [S3]. Choose this when dealing with auto-exposure cameras in novel view synthesis [S3].

## Operating characteristics
*   **Performance:** Enabling exposure and sensor bias estimation improves reconstruction quality by approximately 1 dB [S1].
*   **Complexity:** Adds minimal computational overhead per iteration as it involves simple per-pixel affine transformations [S2].
*   **Memory:** Requires storing per-camera gain/bias (negligible) and optionally per-camera background images (significant if high resolution) [S2].
*   **Strengths:** Robust to photometric inconsistencies and uncalibrated sensors [S1, S2].
*   **Limitations:** Assumes static backgrounds for background merging; may not handle dynamic lighting changes well [S2].

## Background
Radiometric self-calibration originated in the context of neural volume rendering and tomography to address photometric inconsistencies in real-world datasets [S1, S2]. It was introduced as a component of systems like NeAT (Neural Adaptive Tomography) and Neural Volumes to improve reconstruction quality by jointly optimizing radiometric parameters with the volume representation [S1, S2].

## Sources
[S1] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
[S2] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S3] IBGS: Image-Based Gaussian Splatting, https://arxiv.org/abs/2511.14357v1, fetched 2026-10-26
