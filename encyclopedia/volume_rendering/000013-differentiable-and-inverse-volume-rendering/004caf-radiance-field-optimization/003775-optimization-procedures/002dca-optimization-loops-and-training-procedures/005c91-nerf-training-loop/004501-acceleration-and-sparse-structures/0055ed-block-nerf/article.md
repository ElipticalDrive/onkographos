# Block-NeRF

Also called: Block-based NeRF, Spatially Decomposed NeRF

## Overview
Block-NeRF scales Neural Radiance Fields to city-scale environments by decomposing the scene into multiple individually trained, memory-constrained NeRFs [S1][S3, unofficial]. It decouples rendering time from the total scene size by dynamically selecting and compositing only the relevant blocks for a given camera viewpoint [S1]. Unlike a single monolithic NeRF, it allows for piecewise updates and independent optimization of spatial regions [S1].

## How it works
Block-NeRF extends the mip-NeRF architecture [S1, S2] to handle large-scale outdoor data collected over time, addressing transient objects, pose errors, and appearance variations [S1].

### Inputs and Outputs
*   **Inputs:** A set of posed camera images covering a large area (e.g., street-level imagery), semantic segmentation masks for dynamic objects, and camera exposure metadata (shutter speed, analog gain) [S1].
*   **Outputs:** A collection of independent Block-NeRF models, one per spatial block, and rendered images for novel views via compositing [S1].

### Data Structures
Each Block-NeRF consists of three Multi-Layer Perceptrons (MLPs) [S1]:
1.  **Density Network ($f_\sigma$):**
    *   Input: Positional encoded position $\mathbf{x}$.
    *   Output: Density $\sigma$ and a feature vector $\mathbf{f}$.
    *   Structure: 8 layers, width 512 (Mission Bay) or 1024 (other experiments) [S1].
2.  **Color Network ($f_c$):**
    *   Input: Concatenation of $\mathbf{f}$, viewing direction $\mathbf{d}$, exposure embedding, and appearance embedding $\mathbf{a}$ [S1].
    *   Output: Color $\mathbf{c}$.
    *   Structure: 3 layers, width 128 [S1].
3.  **Visibility Network ($f_v$):**
    *   Input: Position $\mathbf{x}$ and viewing direction $\mathbf{d}$.
    *   Output: Predicted visibility/transmittance $\hat{T}$ [S1].
    *   Structure: 4 layers, width 128 [S1].

**Appearance Embeddings:** A latent vector $\mathbf{a}$ of dimension 32, optimized per image using Generative Latent Optimization [S1, S2].

**Pose Correction:** A learned offset per driving segment consisting of a translation vector and a $3 \times 3$ residual rotation matrix added to the identity matrix and normalized [S1].

### Steps

1.  **Scene Partitioning:**
    *   Divide the environment into spatial blocks. Typically, one Block-NeRF is placed at each intersection, covering the intersection and 75% of the way to the next, resulting in 50% overlap between adjacent blocks on connecting streets [S1].
    *   Apply a geographical filter to ensure training data for each block stays within its intended bounds [S1].

2.  **Preprocessing:**
    *   Use a semantic segmentation model to mask out transient objects (cars, pedestrians) [S1, S2].
    *   Process camera exposure information as $\gamma PE(\text{shutter speed} \times \text{analog gain} / t)$, where $\gamma PE$ is a sinusoidal positional encoding with 4 levels and $t=1000$ [S1].

3.  **Training Individual Block-NeRFs:**
    *   For each block, train $f_\sigma$, $f_c$, and $f_v$ independently [S1].
    *   **Density/Color Loss:** Standard NeRF volumetric rendering loss [S1].
    *   **Visibility Loss:** Train $f_v$ to predict transmittance $T_i$ (Eq. 2 in [S1]) using Mean Squared Error (MSE) loss, scaled by $10^{-6}$, supervised by the density $\sigma$ from $f_\sigma$ [S1, S2].
    *   **Pose Refinement:** Optimize pose offsets jointly with the NeRF. Initialize offsets to 0. Regularize the element-wise L2 norm of the offsets, scaled by $10^5$ initially, linearly decaying to $10^{-1}$ after 5000 iterations [S1].
    *   **Optimizer:** Adam [S1].
    *   **Batch Size:** 16384 [S1].
    *   **Iterations:** 300,000 [S1].
    *   **Learning Rate:** Annealed logarithmically from $2 \cdot 10^{-3}$ to $2 \cdot 10^{-5}$, with warm-up for the first 1024 iterations [S1].
    *   **Sampling:** 256 samples per ray during training (coarse and fine networks) [S1].

4.  **Appearance Matching:**
    *   To align adjacent Block-NeRFs, optimize their appearance embeddings to minimize the L2 loss between renders of overlapping areas [S1].
    *   Propagate optimized appearance iteratively through the scene starting from a root Block-NeRF [S1].

5.  **Inference (Rendering):**
    *   **Selection:** Compute visibility maps for all Block-NeRFs within a given radius of the camera. Discard Block-NeRFs with low visibility [S1, S2].
    *   **Rendering:** Render color images from the filtered Block-NeRFs [S1].
    *   **Compositing:** Interpolate the rendered images using Inverse Distance Weighting (IDW) in 2D image space [S1].
        *   Weight for block $i$: $w_i \propto \text{distance}(\mathbf{c}, \mathbf{x}_i)^{-p}$, where $\mathbf{c}$ is the camera origin, $\mathbf{x}_i$ is the block center, and $p$ is a power parameter (e.g., $p=4$ for Alamo Square, $p=1$ for Mission Bay) [S1, S2].
        *   Final color: $\mathbf{c}_{out} = \frac{\sum w_i \mathbf{c}_i}{\sum w_i}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def train_block_nerf(block_data, block_origin, pose_offsets):
    """
    Trains a single Block-NeRF for a specific spatial block.
    block_data: List of images with poses, masks, and exposure metadata.
    block_origin: 3D coordinate of the block center.
    pose_offsets: Learned pose correction (translation + rotation).
    """
    # Initialize MLPs
    f_sigma = MLP(layers=8, width=1024) # Width 512 for Mission Bay
    f_c = MLP(layers=3, width=128)
    f_v = MLP(layers=4, width=128)
    
    # Appearance embeddings (32 dims) optimized per image
    appearance_embeddings = optimize_per_image_embeddings(block_data)
    
    optimizer = Adam(lr=2e-3, warmup_steps=1024)
    
    for iteration in range(300000):
        # Decay pose regularization
        reg_scale = 1e5 * max(1 - iteration/5000, 0) + 1e-1 * min(iteration/5000, 1)
        
        batch = sample_batch(block_data, batch_size=16384)
        
        for ray in batch:
            # Filter ray if it doesn't pass through the block's volume
            if not ray_intersects_block(ray, block_origin):
                continue
                
            # Apply pose correction to ray origin/direction
            corrected_ray = apply_pose_correction(ray, pose_offsets)
            
            # Sample points along ray (256 samples)
            points = sample_points(corrected_ray, num_samples=256)
            
            # Get positional encoding
            pe_points = integrated_positional_encoding(points)
            
            # Forward pass through density network
            sigma, feature_vec = f_sigma(pe_points)
            
            # Get appearance embedding for this image
            app_emb = appearance_embeddings[ray.image_id]
            
            # Exposure encoding
            exposure_val = ray.shutter_speed * ray.analog_gain / 1000
            exposure_pe = sinusoidal_pe(exposure_val, levels=4)
            
            # Forward pass through color network
            color_input = concat([feature_vec, ray.direction, exposure_pe, app_emb])
            color = f_c(color_input)
            
            # Forward pass through visibility network
            vis_input = concat([pe_points, ray.direction])
            vis_pred = f_v(vis_input)
            
            # Compute transmittance from sigma
            deltas = compute_deltas(points)
            transmittance = exp(-sum(sigma[:-1] * deltas[:-1]))
            
            # Loss calculation
            color_loss = mse(color, ray.pixel_color)
            vis_loss = mse(vis_pred, transmittance) * 1e-6
            
            # Pose regularization loss
            pose_loss = l2_norm(pose_offsets) * reg_scale
            
            total_loss = color_loss + vis_loss + pose_loss
            
            optimizer.step(total_loss)

def render_block_nerf(camera, block_nerfs):
    """
    Renders a view by compositing relevant Block-NeRFs.
    camera: Camera origin c and direction.
    block_nerfs: List of trained Block-NeRF models with their origins.
    """
    # 1. Selection: Filter blocks by visibility and distance
    active_blocks = []
    for block in block_nerfs:
        # Check if block is within rendering radius
        if distance(camera.origin, block.origin) > RENDER_RADIUS:
            continue
        
        # Check visibility using f_v
        # Sample points in block volume and average predicted visibility
        avg_vis = compute_avg_visibility(block.f_v, block.origin)
        if avg_vis < VISIBILITY_THRESHOLD:
            continue
            
        active_blocks.append(block)
        
    # 2. Rendering and Compositing
    # Render each active block
    renders = []
    for block in active_blocks:
        # Render the block using standard volumetric rendering
        # with coarse/fine sampling (512 samples for video)
        img = render_single_block(block, camera, num_samples=512)
        renders.append((img, block.origin))
        
    if not renders:
        return empty_image()
        
    # 3. Inverse Distance Weighting (IDW) in 2D image space
    # Assume all renders are aligned to the same image grid
    final_image = zeros_like(renders[0][0])
    weight_sum = 0
    
    for img, origin in renders:
        dist = distance(camera.origin, origin)
        # Power p=4 for Alamo Square, p=1 for Mission Bay
        weight = dist ** -4 
        
        final_image += img * weight
        weight_sum += weight
        
    return final_image / weight_sum
```

### Parameters and Edge Cases
*   **Overlap:** 50% overlap is typical between adjacent blocks on connecting streets [S1]. Reducing overlap improves PSNR/SSIM but increases block size constraints; overlap is necessary to avoid temporal artifacts during interpolation [S1].
*   **Transient Objects:** Masked out using semantic segmentation. If not properly masked, they cause artifacts [S1].
*   **Pose Errors:** Addressed by learned pose offsets. Regularization is critical in early training to prevent instability [S1].
*   **Exposure:** Handled via $\gamma PE$ encoding. Allows modifying exposure at inference [S1].

## Subtopics

### Inverse Distance Weighting (IDW) Interpolation
A method for blending outputs from multiple Block-NeRFs based on the distance between the camera and the Block-NeRF centers to smooth transitions [S1].
*   **Definition:** Weights are calculated as $w_i \propto \text{distance}(\mathbf{c}, \mathbf{x}_i)^{-p}$ in 2D image space [S1].
*   **Performance:** On the Mission Bay dataset, IDW 2D achieved PSNR 26.59, SSIM 0.890, LPIPS 0.278, compared to Nearest Neighbor's PSNR 26.40 [S1].
*   **Comparison:** IDW 2D is preferred for flythrough videos for temporal consistency. IDW 3D (using projected 3D points) suffers from artifacts if depth predictions are incorrect. Nearest neighbor is least compute-intensive but has harsh jumps [S1].

## Operating characteristics
*   **Time Complexity:** Rendering time is decoupled from scene size. Only a subset of blocks (typically <3) are rendered per frame [S1].
*   **Memory Complexity:** Each Block-NeRF fits into memory independently. Total memory scales with the number of blocks, but each block is small [S1].
*   **Performance:** Training takes 9-24 hours per Block-NeRF on 32 TPU v3 cores [S1]. Rendering a 1200x900px image for a single block takes ~5.9 seconds [S1].
*   **Strengths:** Scales to arbitrarily large environments (e.g., entire neighborhoods). Allows piecewise updates without retraining the whole scene [S1].
*   **Limitations:** Requires overlap for seamless compositing. Distant objects may be blurrier due to unbounded volumetric sampling [S1]. Real-time rendering is challenging due to MLP evaluation cost [S1].
*   **When to use:** Large-scale urban reconstruction, autonomous driving simulation, city-scale mapping [S1][S3, unofficial].
*   **When not to use:** Small scenes where a single NeRF suffices, or applications requiring real-time interactive frame rates without significant optimization [S1].

## Background
Block-NeRF was introduced by Matthew Tancik, Vincent Casser, Xinchen Yan, Ben Mildenhall, Pratul P. Srinivasan, Jonathan T. Barron, Sabeek Pradhan, and Henrik Kretzschmar in arXiv:2202.05263v1 on Feb 10, 2022 [S1]. It was developed to address the limitations of single NeRFs in scaling to city-sized environments, leveraging data collected from Waymo autonomous vehicles [S1]. It builds on mip-NeRF and NeRF-W [S1, S2].

## Sources
[S1] Block-NeRF: Scalable Large Scene Neural View Synthesis, https://doi.org/10.1109/cvpr52688.2022.00807, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
