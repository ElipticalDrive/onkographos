# Random Patch Sampling

Also called: RP+RRot+Hflip, Rand-Patch, Random Patch Sampling with Augmentations

## Overview
Random patch sampling is a training strategy for Neural Radiance Field (NeRF) models, specifically those employing Super-Resolution (SR) modules or handling defocus blur, where image patches are extracted from random positions in the ray space rather than a fixed grid [S1, S2]. It solves the problem of rigid boundary artifacts in grid-based sampling, which prevents convolutional kernels from seeing variations between grid lines, and reduces training time by allowing neighboring pixels to share ray computations in blur modeling [S1, S2]. Unlike grid-based patch sampling, it avoids cutting ray space strictly, ensuring the model sees diverse patterns without bias [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images (Low-Resolution ray space $R_{LR}$ and High-Resolution ray space $R_{HR}$), a patch size $P'$, and optional augmentation parameters (rotation angle, flip probability).
*   **Outputs:** Transformed patches $R_{LR}^{P'}$ and $R_{HR}^{P'}$ used for gradient updates.

**Data Structures:**
*   **Ray Space:** Represented as 2D image grids corresponding to camera views.
*   **Patch:** A sub-region of the ray space defined by width and height $P'$.

**Steps:**
1.  **Random Region Selection:** Randomly sample a region in the ray space. The region is defined by a top-left coordinate and dimensions $P' \times P'$ [S1, S2].
    *   *Note:* In deblurring contexts, the patch size $P'$ determines the number of "interesting" pixels. The total rendered pixels include neighboring rays for blur convolution: $(P' + K + 1)^2$, where $K$ is the blur kernel size [S2].
2.  **Transformation (Augmentation):** Apply a transformation $\mathcal{A}$ to the selected LR and HR patches to yield transformed patches [S1]:
    $$ R_{LR}^{P'} = \mathcal{A}(R_{LR}^{P}) $$
    $$ R_{HR}^{P'} = \mathcal{A}(R_{HR}^{P}) $$
    Common transformations include random rotation and horizontal flip [S1].
3.  **Training Integration:** Integrate the transformed patches into the loss function (e.g., Equation 3 in [S1]) to train the pipeline [S1].
4.  **Iteration:** Repeat steps 1-3 for a fixed number of iterations or epochs. The patch size and number of iterations are fixed to ensure the total number of patches seen by the model is identical to grid-based baselines [S1].

**Parameters:**
*   **Patch Size ($P'$):** The width and height of the region to compute intensity. In experiments, $P' = 22$ [S2].
*   **Blur Kernel Size ($K$):** Size of the blur kernel ($K \times K$). In experiments, $K = 11$ [S2].
*   **Rotation Angle:** Maximum rotation angle. Set to 10 degrees for synthetic datasets and 5 degrees for LLFF [S1].
*   **Horizontal Flip Probability:** Set to 10% for synthetic datasets. Omitted for LLFF as it is inappropriate for forward-facing scenes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def random_patch_sampling_training(nerf_model, sr_model, dataset, patch_size, num_epochs, augmentation_config):
    """
    Trains a NeRF+SR pipeline using random patch sampling.
    
    Args:
        nerf_model: The NeRF backbone model.
        sr_model: The Super-Resolution model (e.g., EDSR).
        dataset: Iterable of training views (images).
        patch_size: Integer, width/height of the patch (P').
        num_epochs: Integer, number of training epochs.
        augmentation_config: Dict with 'max_rotation' and 'flip_prob'.
    """
    for epoch in range(num_epochs):
        for view in dataset:
            # 1. Randomly sample a region in the ray space
            # Determine bounds based on image size and patch_size
            h, w = view.lr_image.shape[:2]
            y_start = random.randint(0, h - patch_size)
            x_start = random.randint(0, w - patch_size)
            
            # Extract LR and HR patches
            lr_patch = view.lr_image[y_start:y_start+patch_size, x_start:x_start+patch_size]
            hr_patch = view.hr_image[y_start:y_start+patch_size, x_start:x_start+patch_size]
            
            # 2. Apply transformation A (Augmentation)
            # Random Rotation
            angle = random.uniform(-augmentation_config['max_rotation'], augmentation_config['max_rotation'])
            lr_patch = rotate(lr_patch, angle)
            hr_patch = rotate(hr_patch, angle)
            
            # Random Horizontal Flip
            if random.random() < augmentation_config['flip_prob']:
                lr_patch = flip_horizontal(lr_patch)
                hr_patch = flip_horizontal(hr_patch)
            
            # 3. Forward Pass
            # Render rays corresponding to the patch using NeRF
            # In deblurring context, render (P' + K + 1)^2 pixels to account for neighbors
            rendered_lr_patch = nerf_model.render_patch(lr_patch)
            
            # Apply SR model to upsample the rendered patch
            sr_output = sr_model(rendered_lr_patch)
            
            # 4. Compute Loss and Backpropagate
            # Compare sr_output with hr_patch
            loss = compute_loss(sr_output, hr_patch)
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Random sampling near image edges requires careful coordinate clamping or padding to avoid out-of-bounds errors [S1].
*   **Forward-Facing Scenes:** Horizontal flips must be omitted for forward-facing datasets (e.g., LLFF) as they create geometrically invalid views [S1].
*   **Training Stability:** Grid-based sampling requires randomizing the order of patches to ensure stability; random patch sampling inherently provides this diversity [S1].

## Subtopics

### Finetuning Pretrained SR Model with Random Patch Sampling
A training strategy where a pretrained Super-Resolution model (e.g., EDSR) is finetuned on the NeRF dataset using random patch sampling. It offers the best trade-off between training time and PSNR, achieving 31.64 PSNR (2×) and 32.53 PSNR (4×) on NeRF-Synthetic with training times of 51 minutes (2×) and 19 minutes (4×). It outperforms bilinear interpolation and pretrained EDSR without finetuning, while being faster than distillation methods [S1]. Training duration is 150 epochs [S1].

## Operating characteristics
*   **Time Complexity:** Reduces training time compared to rendering all rays. In deblurring, it reduces the number of required neighboring rays per ray from $K^2$ to $\frac{K^2}{(P' + K + 1)^2 / P'^2}$ [S2].
*   **Memory Complexity:** Depends on patch size $P'$. Smaller patches reduce memory footprint per iteration [S2].
*   **Performance:** Improves PSNR by up to 0.89 dB for 2× upscaling and 1.44 dB for 4× upscaling compared to grid-based sampling [S1].
    *   NeRF-Synthetic SR-2×: 32.53 dB (Random) vs 31.84 dB (Grid) [S1].
    *   NeRF-Synthetic SR-4×: 35.39 dB (Random) vs 34.34 dB (Grid) [S1].
    *   LLFF SR-4×: 30.47 dB (Random) vs 29.28 dB (Grid) [S1].
*   **Strengths:** Lightweight augmentation; improves generalization by exposing convolutional kernels to diverse patterns; reduces training overhead compared to distillation [S1].
*   **Limitations:** Adding rotation and horizontal flip augmentations causes marginal PSNR degradation compared to random patch sampling alone [S1]. Slight PSNR decrease (0.16 dB) at 2× on LLFF [S1].

## Background
Originated in the context of efficient NeRF training, specifically for Super-Resolution pipelines (FastSR-NeRF) and deblurring (Sharp-NeRF). Proposed to mitigate the overfitting issues of CNN-based SR modules on small NeRF datasets and to model defocus blur interference [S1, S2]. The technique was introduced in 2024 [S1, S2].

## Sources
[S1] FastSR-NeRF: Improving NeRF Efficiency on Consumer Devices with A Simple Super-Resolution Pipeline, https://doi.org/10.1109/wacv57701.2024.00593, fetched 2026-10-26
[S2] Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior, https://doi.org/10.1109/wacv57701.2024.00367, fetched 2026-10-27
