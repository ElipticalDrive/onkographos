# Adaptive sampling and reconstruction
Also called: Weiss et al

## Overview
Adaptive sampling and reconstruction is an end-to-end neural pipeline that predicts high-density sampling locations using an importance network, converts these predictions into sparse sample positions via a differentiable sampler, and reconstructs full-resolution images from the sparse data using a reconstruction network [S1, unofficial]. It addresses the inefficiency of uniform sampling by allocating computational resources only to visually important regions, achieving high fidelity with significantly fewer volume queries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Low-resolution view (typically 128×128) or view parameters [S1, unofficial].
*   **Output:** Full-resolution rendered image [S1, unofficial].
*   **Intermediate:** Sparse set of sample positions and their corresponding volume values [S1, unofficial].

**Data Structures:**
*   **Importance Map:** Scalar field predicted by the importance network representing sampling probability density [S1, unofficial].
*   **Sample Positions:** Set of 2D coordinates $(u, v)$ and associated 3D ray parameters derived from the importance map [G].
*   **Sparse Samples:** List of tuples $(ray\_id, t, value)$, where $ray\_id$ is the pixel index, $t$ is the sample position along the ray, and $value$ is the accumulated color/opacity or scalar value [G].

**Numbered Steps:**
1.  **Importance Prediction:** The importance network takes a low-resolution input (e.g., 128×128) and outputs an importance map $I(x, y)$ for each pixel. This network learns to identify silhouettes and high-curvature regions without explicit feature descriptors [S1, unofficial].
2.  **Differentiable Sampling:** The importance map $I(x, y)$ is converted into a probability distribution. Sample positions are drawn from this distribution using a soft-max approximation with parameter $\alpha$ [S1, unofficial].
3.  **Volume Query:** For each selected sample position, a ray is cast into the volume, and the value is queried. This results in a sparse set of samples, significantly reducing the number of volume queries compared to uniform sampling [S1, unofficial].
4.  **Reconstruction:** The reconstruction network takes the sparse samples and the original low-resolution view as input. It combines pull-push inpainting with a residual CNN to predict the missing pixel values and reconstruct the full-resolution image [S1, unofficial].
5.  **Training:** The entire pipeline is trained end-to-end using image losses (PSNR, SSIM, LPIPS) computed on the final reconstructed image. The goal is to minimize the difference between the reconstructed image and the ground truth rendered at high sample density [S1, unofficial].

**Formulas:**
*   Soft-max approximation parameter $\alpha \approx 50$ controls the sharpness of the sampling distribution [G]. Explicit sampling distribution formulas are not given in the sources [S1, unofficial].

**Parameters:**
*   **Soft-max approximation parameter ($\alpha$):** Controls the sharpness of the sampling distribution [G]. Typical value: $\alpha \approx 50$ [S1, unofficial].
*   **Sample Density:** Fraction of total possible samples used [G]. Typical values: 5–10% for isosurface rendering, 10% for Direct Volume Rendering (DVR) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Reconstruction Overhead:** The reconstruction network has a fixed overhead of approximately 90ms, regardless of volume size. This makes the method slower than optimized ray casting for volumes smaller than 1024^3 or images smaller than 1024^2 [S1, unofficial].
*   **Transfer Function Mismatch:** Quality degrades significantly when test transfer functions use color ranges that were absent during training. The network struggles to reconstruct textures if the color distribution is out-of-distribution [S1, unofficial].
*   **Complex Transfer Functions:** Networks struggle with small-scale color variations from complex transfer functions, often defaulting to uniform sampling and resulting in poor reconstruction in textured regions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sampling_and_reconstruction(volume, view_params, importance_net, reconstruction_net, alpha=50, density=0.1):
    # Step 1: Generate low-resolution view for importance prediction
    # Not given in the sources. Standard coarse render or learned feature map [G].
    low_res_view = coarse_render(volume, view_params, resolution=(128, 128))
    
    # Step 2: Predict importance map
    importance_map = importance_net(low_res_view)
    
    # Step 3: Convert importance map to sample positions
    # Soft-max approximation parameter alpha controls distribution sharpness [G].
    num_pixels = 128 * 128
    num_samples = int(num_pixels * density)
    
    # Differentiable sampler draws samples based on importance distribution [S1, unofficial].
    # Implementation uses soft-max approximation with alpha [G].
    # Explicit sampling distribution formulas are not given in the sources [S1, unofficial].
    sample_positions = soft_max_sample(importance_map, num_samples, alpha)
    
    # Step 4: Query volume at sample positions
    sparse_samples = []
    for pos in sample_positions:
        ray = cast_ray_from_pixel(view_params, pos)
        value = trilinear_interpolate(volume, ray.origin, ray.direction)
        sparse_samples.append((pos, value))
        
    # Step 5: Reconstruct full-resolution image
    full_res_image = reconstruction_net(low_res_view, sparse_samples)
    
    return full_res_image
```

## Operating characteristics
*   **Performance:** Achieves PSNR > 30 dB and SSIM > 0.95 with only 5–10% of the samples used in uniform sampling [S1, unofficial].
*   **Speedup:** Demonstrates a 1.5–2× speedup for large volumes (2048^3) at high resolutions (≥1536^2) [S1, unofficial].
*   **Strengths:** Reduces computational cost by focusing samples on important regions; learns sampling strategy without explicit feature engineering; high visual fidelity with sparse samples [S1, unofficial].
*   **Limitations:** High reconstruction overhead for small volumes/images; sensitive to transfer function changes; struggles with complex, small-scale textures; requires training data and joint optimization [S1, unofficial].
*   **When to Use:** Large volumes (≥1024^3) and high resolutions (≥1024^2) where the reconstruction overhead is amortized; scenarios where transfer functions are stable and known during training [S1, unofficial].
*   **When Not to Use:** Small volumes or low resolutions; dynamic or unknown transfer functions; scenes with highly complex, small-scale textures that were not present in training data [S1, unofficial].

## Background
The method was proposed by Weiss et al. [156] as a learned optimization for volume visualization, addressing the inefficiency of uniform sampling [S1, unofficial]. The approach is part of a broader trend in scientific visualization and computer graphics to leverage deep learning for rendering acceleration and quality improvement [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
