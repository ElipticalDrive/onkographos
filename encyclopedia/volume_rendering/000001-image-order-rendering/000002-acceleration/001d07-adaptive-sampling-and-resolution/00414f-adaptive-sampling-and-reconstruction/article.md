# Adaptive sampling and reconstruction
Also called: Weiss et al

## Overview
Adaptive sampling and reconstruction is an end-to-end learning-based pipeline that uses neural networks to predict regions of high visual importance, convert these predictions into sparse sample positions, and reconstruct full-resolution images from those sparse samples [S1, unofficial]. It addresses the inefficiency of uniform sampling by allocating computational resources only where they are visually critical, achieving high fidelity with significantly fewer volume queries than traditional ray casting [S1, unofficial]. It differs from sibling methods like [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md) by decoupling sample placement from the rendering loop via a learned importance network and a separate reconstruction network, rather than adapting resolution or sample count dynamically during standard rasterization or ray marching.

## How it works
The method consists of three jointly trained components: an importance network, a differentiable sampler, and a reconstruction network.

**Inputs and Outputs:**
*   **Input:** A low-resolution input image (typically $128 \times 128$) or feature map derived from the volume/transfer function context.
*   **Output:** A full-resolution reconstructed image.
*   **Intermediate:** Sparse set of sample positions and values.

**Data Structures:**
*   **Importance Map:** A 2D tensor of size $128 \times 128$ containing scalar importance values for each pixel region.
*   **Sample Positions:** A list of $(u, v)$ coordinates and associated volume values, sparsely distributed based on the importance map.
*   **Reconstructed Image:** A full-resolution image tensor.

**Parameters:**
*   **Soft-max approximation parameter ($\alpha$):** Approximately 50. This controls the sharpness of the sampling distribution [S1, unofficial].
*   **Sample Density:** 5–10% for isosurface rendering; 10% for Direct Volume Rendering (DVR) [S1, unofficial].

**Steps:**
1.  **Importance Prediction:** The importance network takes the $128 \times 128$ input and predicts an importance map indicating where high sample density is required. The network learns to place samples on silhouettes and high-curvature regions without explicit feature descriptors [S1, unofficial].
2.  **Differentiable Sampling:** The importance map is converted into adaptive sample positions using a differentiable sampler. This process uses a soft-max approximation with parameter $\alpha \approx 50$ to select a sparse subset of locations proportional to their importance [S1, unofficial].
3.  **Volume Query:** Volume values are queried only at the selected sparse sample positions.
4.  **Reconstruction:** A reconstruction network combines pull-push inpainting with a residual Convolutional Neural Network (CNN) to reconstruct the full-resolution image from the sparse samples and the original low-resolution input [S1, unofficial].
5.  **Training:** The entire pipeline is trained jointly using image loss functions, specifically PSNR, SSIM, and LPIPS, to optimize final reconstruction quality [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sampling_and_reconstruction(volume, transfer_function, camera):
    """
    End-to-end adaptive sampling and reconstruction pipeline.
    """
    # 1. Generate low-resolution input (e.g., coarse render or feature map)
    # In practice, this might be a very low-res ray cast or pre-computed features.
    low_res_input = generate_low_res_input(volume, transfer_function, camera, resolution=(128, 128))
    
    # 2. Predict importance map
    # ImportanceNet is a neural network trained to output importance values
    importance_map = ImportanceNet(low_res_input)
    
    # 3. Convert importance to sample positions
    # DifferentiableSampler uses soft-max approximation with alpha ~ 50
    # Returns sparse coordinates and values
    sample_positions, sample_values = DifferentiableSampler(
        importance_map, 
        volume, 
        transfer_function, 
        camera, 
        alpha=50,
        target_density=0.10  # 10% for DVR, 0.05-0.10 for isosurfaces
    )
    
    # 4. Reconstruct full-resolution image
    # ReconstructionNet combines pull-push inpainting and residual CNN
    full_res_image = ReconstructionNet(
        low_res_input, 
        sample_positions, 
        sample_values
    )
    
    return full_res_image
```

**Edge Cases and Pitfalls:**
*   **Overhead Threshold:** The reconstruction network has a fixed overhead of approximately 90ms, regardless of volume size. This makes the method slower than optimized ray casting for volumes smaller than $1024^3$ or images smaller than $1024^2$ [S1, unofficial].
*   **Generalization:** Quality degrades significantly when the test transfer function uses color ranges that were absent during training [S1, unofficial].
*   **Complex Textures:** Networks struggle with small-scale color variations from complex transfer functions, often defaulting to uniform sampling and resulting in poor reconstruction in textured regions [S1, unofficial].

## Operating characteristics
*   **Performance:** Achieves PSNR > 30 dB and SSIM > 0.95 using only 5–10% of the samples required by uniform sampling [S1, unofficial].
*   **Speedup:** Demonstrates a 1.5–2× speedup for large volumes ($2048^3$) at high resolutions ($\ge 1536^2$) [S1, unofficial].
*   **Memory:** Requires memory for three neural networks (importance, sampler, reconstruction) plus the volume data. The specific memory footprint of the networks is not given in the sources [S1, unofficial].
*   **Strengths:** Eliminates the need for explicit feature descriptors; automatically learns to focus on visually important regions (silhouettes, high curvature); high fidelity with sparse samples.
*   **Limitations:** High fixed latency (90ms) makes it unsuitable for small volumes or real-time interactive rendering on modest hardware; poor generalization to unseen transfer functions; struggles with fine textural details.
*   **When to use:** Large-scale volume visualization ($\ge 2048^3$) where offline or near-real-time rendering is acceptable and transfer functions are known and stable.
*   **When not to use:** Small volumes ($< 1024^3$), dynamic transfer functions with unseen color ranges, or scenarios requiring very high-frequency detail reconstruction.

## Background
The method was introduced by Weiss et al. [156] as a solution to the inefficiency of uniform sampling in volume visualization. It represents a shift from traditional acceleration techniques (like empty-space skipping) to learned, data-driven approaches that optimize sample placement based on visual importance. The approach is part of a broader trend in scientific visualization leveraging deep learning for rendering optimization.

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
