# Superpixel-guide Efficient Sampling (SES)


## Overview
Superpixel-guide Efficient Sampling (SES) is a training strategy for neural fields (such as NeRF) that uses SLIC superpixels as the smallest processing units to guide ray sampling, addressing slow convergence and scattered sampling in high-resolution training by focusing computation on complex regions [S1]. Unlike sibling methods like [Context-Based Adaptive Ray Sampling with Quadtree Subdivision](8164-context-based-adaptive-ray-sampling-with-quadtree-subdivisio.md) which use fixed geometric partitions that cause redundant computations and fragmented information, SES resolves inadequate partitioning issues by adapting to image content and rendering error, thereby reducing computational overhead per instance [S1].

## How it works
**Inputs:** Input RGB images $I$, ground truth images, neural field parameters (e.g., Instant-NGP or DVGO), and hyperparameters $\beta_0$, $\lambda_{sp}$, $\lambda_{mse}$, $t$, $\lambda_{error}$, $\lambda_{std}$.
**Outputs:** Trained neural field model, rendered images.

**Data Structures:**
*   **Superpixel Map:** An array of integers where each element corresponds to a pixel in the input image, storing the superpixel ID it belongs to.
*   **Error Map:** A 2D array storing the rendering error $r_j$ for each pixel $j$.
*   **Sampling Probability Map:** A 2D array storing the probability $\gamma_K(x)$ for each pixel $x$ to be sampled.

**Numbered Steps:**

1.  **Initialization:**
    *   Apply the SLIC algorithm to partition the input RGB image $I$ into superpixels. Let $\ell_i$ be the $i$-th superpixel and $L$ be the collection of all superpixels [S1].
    *   Initialize the neural field with random weights.
    *   Set initial sampling probabilities uniformly or based on initial random sampling.

2.  **Initial Training and Error Recording:**
    *   Perform random ray sampling for training.
    *   After each training iteration/session, record the rendering error $r_j$ for each sampled pixel $j$ [S1].
    *   Calculate the average error for each superpixel $\ell_{error}^i$:
        $$ \ell_{error}^i = \frac{1}{N} \sum_{j \in N} r_j $$
        where $N$ is the set of pixels in superpixel $\ell_i$ [S1].

3.  **Secondary Segmentation (Re-segmentation):**
    *   Identify areas with the largest rendering errors and their adjacent regions for re-segmentation. This preserves the original segmentation structure in well-learned regions [S1].
    *   Use a masked SLIC algorithm. The combined feature vector $C_k$ for distance calculation in SLIC considers both RGB color and rendering error:
        $$ C_k = \beta_0 \cdot T_{CIE}(I) + (1 - \beta_0) \cdot T_{CIE}(R) $$
        where $R$ is the portion of render error to be recalculated, replicated three times to match RGB dimensions, and $T_{CIE}$ transforms from RGB to CIE color space [S1]. $\beta_0$ is the weight between RGB image and render error (0.8 for synthetic, 0.65 for real datasets) [S1].
    *   Perform SLIC segmentation on these masked regions to obtain updated superpixels.

4.  **Guided Sampling Probability Calculation:**
    *   Calculate sampling probabilities based on superpixel errors. The probability component for error $\ell_{p_i}^{error}$ and standard deviation $\ell_{p_i}^{std}$ are calculated as [S1]:
        $$ \ell_{p_i}^{error} = \lambda_{error} \cdot \ell_{error}^i $$
        $$ \ell_{p_i}^{std} = \lambda_{std} \cdot \ell_{std\_p}^i $$
        where $t$ is a minimum error threshold used in a clamp function to exclude negligible errors (blank/well-learned regions) [S1].
    *   Normalize the sampling probability of the rendering error component to lie between 0 and 1. Let $m$ be the number of pixels in the current superpixel and $\bar{c}$ denote the average color within the superpixel [S1].
    *   Update the global sampling probability map $\gamma_K(x)$ for position encoding $g_i(x)$:
        $$ \gamma_K'(x) = \gamma_K(x) * M(i) $$
        where $M(i)$ is the mask or weight derived from the superpixel probabilities [S1].

5.  **Ray Generation and Training:**
    *   Regenerate sampled rays according to the updated probability map [S1].
    *   Train the neural field using the sampled rays.
    *   Accumulate rendering errors during this phase.
    *   Upon completion of training for all rays generated in a single sampling step, update the superpixel errors and regenerate sampled rays for the next iteration [S1].

6.  **Loss Function:**
    *   Optimize neural field parameters using pixel-level Mean Squared Error (MSE) loss $L_{mse}$ and superpixel loss $L_{sp}$ [S1].
    *   Total loss:
        $$ L = \lambda_{mse} \cdot L_{mse} + \lambda_{sp} \cdot L_{sp} $$
    *   Superpixel loss $L_{sp}$ is defined as [S1]:
        $$ L_{sp} = \ell_{error}^i + 0.001 \cdot \ell_{std\_p}^i $$
    *   $\lambda_{sp}$ is typically three to four orders of magnitude smaller than $\lambda_{mse}$ due to the strong bias of $L_{sp}$ [S1].

**Parameters:**
*   $\beta_0$: Weight between RGB image and render error in secondary segmentation. Synthetic datasets: 0.8. Real datasets: 0.65 [S1].
*   $\lambda_{sp}$: Coefficient for superpixel loss. Synthetic datasets: $10^{-6}$. Real datasets: $10^{-4}$ [S1].
*   $\lambda_{mse}$: Coefficient for MSE loss. Typically 1 [S1].
*   $t$: Minimum error threshold to exclude negligible errors. Value not explicitly given, but used in clamp function [S1].
*   $\lambda_{error}, \lambda_{std}$: Coefficients for sampling probabilities. Adjusted to balance weights [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ses_training(input_images, ground_truth, neural_field, hyperparams):
    """
    SES Training Loop
    """
    # Hyperparameters
    beta_0 = hyperparams['beta_0']
    lambda_sp = hyperparams['lambda_sp']
    lambda_mse = hyperparams['lambda_mse']
    t = hyperparams['t'] # Minimum error threshold
    lambda_error = hyperparams['lambda_error']
    lambda_std = hyperparams['lambda_std']
    
    # Initialization
    # Apply SLIC to partition input images into superpixels
    superpixel_map = slic_segmentation(input_images, beta_0=beta_0)
    
    # Initialize sampling probabilities uniformly
    sampling_prob = initialize_uniform_probability(input_images.shape)
    
    # Initial random sampling
    rays = generate_random_rays(input_images.shape, sampling_prob)
    
    for epoch in range(num_epochs):
        # 1. Train with current rays
        rendered_images = neural_field.render(rays)
        loss_mse = compute_mse_loss(rendered_images, ground_truth)
        
        # Record rendering error for each pixel
        pixel_errors = compute_pixel_errors(rendered_images, ground_truth)
        
        # Update superpixel errors
        superpixel_errors = compute_superpixel_errors(pixel_errors, superpixel_map)
        
        # 2. Secondary Segmentation (every N epochs or iterations)
        # Identify regions with largest errors
        regions_to_resegment = identify_high_error_regions(superpixel_errors, threshold=t)
        
        if regions_to_resegment:
            # Re-segment specific regions using masked SLIC
            # Combine RGB and Render Error in CIE space
            error_map_replicated = replicate_error_to_rgb(pixel_errors)
            combined_features = beta_0 * rgb_to_cie(input_images) + (1 - beta_0) * rgb_to_cie(error_map_replicated)
            
            # Apply masked SLIC
            superpixel_map = masked_slic_segmentation(input_images, combined_features, regions_to_resegment)
            
            # Recalculate superpixel errors after re-segmentation
            superpixel_errors = compute_superpixel_errors(pixel_errors, superpixel_map)
        
        # 3. Calculate Sampling Probabilities
        # Calculate error and std components
        error_component = lambda_error * superpixel_errors
        std_component = lambda_std * compute_superpixel_std(pixel_errors, superpixel_map)
        
        # Normalize and clamp
        # Note: Exact normalization formula from source involves m and c_bar, 
        # simplified here as probability map update based on errors
        normalized_prob = normalize_probabilities(error_component, std_component, t)
        
        # Update global sampling probability map
        sampling_prob = update_sampling_probability(sampling_prob, normalized_prob, superpixel_map)
        
        # 4. Generate new rays based on probabilities
        rays = generate_rays_from_probability(input_images.shape, sampling_prob)
        
        # 5. Compute Total Loss
        # L_sp = l_error^i + 0.001 * l_std_p^i
        loss_sp = compute_superpixel_loss(superpixel_errors, std_component)
        total_loss = lambda_mse * loss_mse + lambda_sp * loss_sp
        
        # Backpropagate
        neural_field.backward(total_loss)
        
    return neural_field
```

## Operating characteristics
*   **Time Complexity:** The method introduces overhead for SLIC segmentation and probability map updates, but reduces the number of rays needed for high-quality rendering by focusing on complex regions. The sources state that superpixels resolve inadequate partitioning issues and reduce computational overhead per instance compared to quadtrees [S1].
*   **Memory Complexity:** Requires storing superpixel maps, error maps, and probability maps. The sources do not specify exact memory overhead in bytes, but note that the method is implemented on CUDA to maximize hardware performance [S1].
*   **Performance:** Integrated with Instant-NGP and DVGO baselines, SES improves rendering quality for high-resolution inputs. Experiments were conducted on a single NVIDIA RTX 4090 GPU [S1].
*   **Strengths:** Addresses slow convergence and scattered sampling in high-resolution NeRF training. Reduces computational overhead compared to fixed-shape partitions like quadtrees.
*   **Limitations:** Requires careful tuning of hyperparameters ($\beta_0$, $\lambda_{sp}$). The method is specific to neural field training and may not directly apply to inference-only scenarios without modification.
*   **When to use:** High-resolution neural field training where standard random sampling is inefficient.
*   **When not to use:** Low-resolution scenes where random sampling is sufficient, or when computational resources for SLIC segmentation are limited.

## Background
SES was introduced by Zhongwei Xuan et al. in 2024 at the ACM Multimedia Conference (MM '24) [S1]. The method was developed to address the limitations of existing adaptive sampling methods, such as quadtrees, which suffer from redundant computations and fragmented information due to fixed shapes, and methods based on color gradients that struggle with large-scale image blocks [S1]. The authors integrated SES into popular neural field methods like Instant-NGP and DVGO to demonstrate its effectiveness in improving rendering quality for large input images [S1].

## Sources
[S1] Superpixel-based Efficient Sampling for Learning Neural Fields from Large Input, https://doi.org/10.1145/3664647.3681299, fetched 2026-11-08
