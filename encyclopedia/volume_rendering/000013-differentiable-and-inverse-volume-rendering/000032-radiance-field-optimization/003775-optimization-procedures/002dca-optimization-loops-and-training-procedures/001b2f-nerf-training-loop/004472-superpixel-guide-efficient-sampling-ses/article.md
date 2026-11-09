# Superpixel-guide Efficient Sampling (SES)


## Overview
Superpixel-guide Efficient Sampling (SES) is a training strategy for neural radiance fields that uses SLIC superpixels as the smallest processing units to guide ray sampling, addressing slow convergence and scattered sampling in high-resolution NeRF training by focusing computation on complex regions [S1]. Unlike quadtrees which cause redundant computations and fragmented information due to fixed shapes, superpixels resolve inadequate partitioning issues and reduce computational overhead per instance [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Input RGB images $I$, ground truth images, neural field parameters (e.g., Instant-NGP or DVGO), and hyperparameters $\beta_0, \lambda_{sp}, \lambda_{mse}, \lambda_{error}, \lambda_{std}$.
*   **Outputs:** Updated neural field parameters, rendered images, and sampling probability maps.

**Data Structures:**
*   **Superpixel Map:** A 2D array where each element corresponds to a pixel in the input image, storing the superpixel ID.
*   **Error Map:** A 2D array storing the rendering error $r_j$ for each pixel $j$.
*   **Sampling Probability Map:** A 2D array storing the probability of sampling a ray for each pixel.

**Numbered Steps:**

1.  **Initialization:**
    *   Apply the SLIC algorithm to partition the input RGB image $I$ into superpixels [S1].
    *   Let $\ell_i$ represent the $i$-th superpixel in the image, and $L$ denote the collection of all superpixels [S1].
    *   Initialize random ray sampling for training [S1].

2.  **Training and Error Recording:**
    *   Perform training iterations using the current sampling strategy [S1].
    *   After each training session/iteration, record the rendering error $r_j$ for each sampled pixel $j$ within a superpixel [S1].
    *   Calculate the average error within each superpixel $\ell_{error}^i$ as follows [S1]:
        $$ \ell_{error}^i = \frac{1}{N} \sum_{j \in N} r_j $$
        where $N$ is the set of pixels in superpixel $\ell_i$ [S1].

3.  **Secondary Segmentation (Re-segmentation):**
    *   After a training period, identify areas with the largest rendering errors and their adjacent regions for re-segmentation [S1].
    *   Use a masked SLIC algorithm to re-segment these specific regions, considering both RGB color and rendering error simultaneously [S1].
    *   Map the rendering error to the RGB color space by replicating it three times to meet dimensional requirements for CIE space transformation [S1].
    *   Calculate the combined feature vector $C_k$ for the masked region $R$ (portion of render error to be recalculated) [S1]:
        $$ C_k = \beta_0 \cdot T_{CIE}(I) + (1 - \beta_0) \cdot T_{CIE}(R) $$
        where $T_{CIE}$ transforms from RGB to CIE color space, and $\beta_0$ is the weight between RGB image and render error [S1].
    *   Calculate the distance in SLIC using both RGB and CIE spaces based on $C_k$ [S1].

4.  **Guided Sampling:**
    *   Calculate the sampling probability for each pixel. The probability component for error $\ell_{p_i}^{error}$ and standard deviation $\ell_{p_i}^{std}$ is calculated as [S1]:
        $$ \ell_{p_i} = \lambda_{error} \cdot \ell_{std\_p} + \lambda_{std} \cdot \ell_{error}^i $$
        where $t$ is the minimum error threshold to exclude negligible errors (used in a clamp function), $m$ is the number of pixels in the current superpixel, and $\bar{c}$ is the average color within the superpixel [S1].
    *   Normalize the sampling probability of the rendering error component to lie between 0 and 1 [S1].
    *   Regenerate sampled rays according to the probability map [S1].
    *   Accumulate rendering errors during this phase [S1].
    *   Upon completion of training for all rays generated in a single sampling, update the superpixel errors and regenerate sampled rays [S1].

5.  **Loss Function:**
    *   Optimize neural field parameters using pixel-level Mean Squared Error (MSE) loss $L_{mse}$ and superpixel loss $L_{sp}$ [S1].
    *   The superpixel loss $L_{sp}$ is calculated as [S1]:
        $$ L_{sp} = \ell_{error}^i + 0.001 \cdot \ell_{std\_p} $$
    *   The total loss is computed as [S1]:
        $$ L = \lambda_{mse} \cdot L_{mse} + \lambda_{sp} \cdot L_{sp} $$
    *   Due to the strong bias of $L_{sp}$, $\lambda_{sp}$ is typically set to be three to four orders of magnitude smaller than $\lambda_{mse}$ [S1].

**Parameters:**
*   $\beta_0$: Weight between RGB image and render error in secondary segmentation. Synthetic datasets: 0.8. Real datasets: 0.65 [S1].
*   $\lambda_{sp}$: Coefficient for superpixel loss. Synthetic datasets: $1e-6$. Real datasets: $1e-4$ [S1].
*   $\lambda_{mse}$: Coefficient for MSE loss. Typically 1 [S1].
*   $t$: Minimum error threshold to exclude negligible errors. Value not explicitly given, but used in clamp function [S1].
*   $\lambda_{error}, \lambda_{std}$: Coefficients for sampling probabilities. Adjusted to balance weights [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from skimage.segmentation import slic
from skimage.color import rgb2lab

def ses_training_loop(images, gt_images, model, num_iterations, beta_0, lambda_sp, lambda_mse, lambda_error, lambda_std, t):
    """
    Main training loop for SES.
    
    Args:
        images: List of input RGB images.
        gt_images: List of ground truth images.
        model: Neural field model (e.g., Instant-NGP).
        num_iterations: Total number of training iterations.
        beta_0: Weight for secondary segmentation.
        lambda_sp: Coefficient for superpixel loss.
        lambda_mse: Coefficient for MSE loss.
        lambda_error: Coefficient for error sampling probability.
        lambda_std: Coefficient for std sampling probability.
        t: Minimum error threshold.
        
    Returns:
        Trained model.
    """
    # Step 1: Initialization
    # Apply SLIC to segment input images into superpixels
    superpixel_maps = []
    for img in images:
        # SLIC parameters need to be chosen appropriately for the image size
        segments = slic(img, n_segments=100, compactness=10, sigma=1)
        superpixel_maps.append(segments)
        
    # Initialize random sampling
    sampling_probabilities = [np.ones_like(img[..., 0]) for img in images]
    
    for iteration in range(num_iterations):
        # Step 2: Training and Error Recording
        errors = []
        for i, (img, gt) in enumerate(zip(images, gt_images)):
            # Sample rays based on current sampling probabilities
            rays = sample_rays(img.shape, sampling_probabilities[i])
            
            # Render using the model
            rendered_images = model.render(rays)
            
            # Calculate pixel-level MSE
            mse_loss = np.mean((rendered_images - gt) ** 2)
            
            # Record rendering error for each pixel
            pixel_errors = (rendered_images - gt) ** 2
            errors.append(pixel_errors)
            
            # Update model parameters using gradient descent
            # (Implementation details depend on the specific model framework)
            model.update_gradients(mse_loss)
            
        # Step 3: Secondary Segmentation (Re-segmentation)
        # Identify areas with largest rendering errors
        avg_errors = [np.mean(err, axis=(0, 1)) for err in errors] # Average error per superpixel? Or per pixel?
        # The source says "average sampling error within each superpixel"
        # Let's assume we calculate error per superpixel first
        superpixel_errors = []
        for i, (sp_map, err) in enumerate(zip(superpixel_maps, errors)):
            unique_sp_ids = np.unique(sp_map)
            sp_err = np.zeros(len(unique_sp_ids))
            for j, sp_id in enumerate(unique_sp_ids):
                mask = (sp_map == sp_id)
                sp_err[j] = np.mean(err[mask])
            superpixel_errors.append(sp_err)
            
        # Identify regions with largest errors and adjacent regions
        # This requires identifying which superpixels have high error and their neighbors
        # For simplicity, we assume we re-segment the whole image but with masked SLIC
        # The source mentions "masked slic algorithm"
        
        # Calculate combined feature vector for re-segmentation
        # Map rendering error to RGB space (replicate 3 times)
        # Transform to CIE space
        # Note: The source formula C_k = beta_0 * T_CIE(I) + (1 - beta_0) * T_CIE(R)
        # R is the portion of render error to be recalculated.
        # This implies we create a "error image" by replicating error values across RGB channels
        # and then transform.
        
        new_superpixel_maps = []
        for i, (img, sp_map, err) in enumerate(zip(images, superpixel_maps, errors)):
            # Create error image
            error_img = np.stack([err, err, err], axis=-1)
            
            # Transform original image to CIE
            img_cie = rgb2lab(img)
            
            # Transform error image to CIE (error is in grayscale, so replicate for 3 channels)
            # Note: rgb2lab expects RGB input. If error_img is already in a format suitable for lab,
            # we might need to adjust. Assuming error_img is treated as RGB-like for transformation.
            # However, error is a scalar per pixel. Replicating 3 times makes it 3-channel.
            # We need to ensure the values are in the correct range for rgb2lab (0-1 or 0-255).
            # Let's assume normalized 0-1.
            error_img_cie = rgb2lab(error_img)
            
            # Combine features
            # C_k = beta_0 * T_CIE(I) + (1 - beta_0) * T_CIE(R)
            combined_features = beta_0 * img_cie + (1 - beta_0) * error_img_cie
            
            # Apply SLIC on combined features
            # Masked SLIC implies we only re-segment certain regions.
            # For simplicity, we apply SLIC on the whole image with combined features.
            new_segments = slic(combined_features, n_segments=100, compactness=10, sigma=1)
            new_superpixel_maps.append(new_segments)
            
        superpixel_maps = new_superpixel_maps
        
        # Step 4: Guided Sampling
        # Calculate sampling probabilities
        new_sampling_probabilities = []
        for i, (sp_map, err) in enumerate(zip(superpixel_maps, errors)):
            unique_sp_ids = np.unique(sp_map)
            sp_probs = np.zeros_like(sp_map, dtype=float)
            
            for sp_id in unique_sp_ids:
                mask = (sp_map == sp_id)
                sp_err = np.mean(err[mask])
                
                # Calculate probability components
                # The source formula: l_p_i = lambda_error * l_std_p + lambda_std * l_error^i
                # This seems to be for the superpixel level probability.
                # We need to map this back to pixel level.
                
                # Normalize error component to [0, 1]
                # The source mentions normalizing the sampling probability of the rendering error component.
                # Let's assume a simple normalization across all superpixels or within the image.
                # For now, we'll use a sigmoid or min-max normalization.
                # The source doesn't specify the exact normalization method, just that it's normalized.
                # We'll use a simple clamp and scale for demonstration.
                normalized_err = np.clip(sp_err, t, None) # Exclude negligible errors
                # Normalize to [0, 1] relative to max error in the image? Or superpixel?
                # Let's assume global normalization for simplicity.
                # In practice, this would be done across all superpixels.
                
                # Calculate std component (not fully detailed in source, assuming std of errors in superpixel)
                sp_std = np.std(err[mask])
                
                # Calculate probability for this superpixel
                # l_p_i = lambda_error * l_std_p + lambda_std * l_error^i
                # Note: l_std_p and l_error^i are likely normalized values.
                # We'll assume they are already normalized or we normalize them here.
                # Let's assume l_error^i is the normalized error.
                # And l_std_p is the normalized std.
                
                # For demonstration, we'll just use the raw values scaled by lambdas.
                prob = lambda_error * sp_std + lambda_std * normalized_err
                
                # Assign this probability to all pixels in the superpixel
                sp_probs[mask] = prob
                
            # Normalize probabilities to sum to 1 or scale appropriately for sampling
            # The source says "normalize the sampling probability... to lie between 0 and 1"
            # This might mean per-pixel or per-superpixel.
            # We'll normalize per-superpixel to [0, 1] by dividing by max prob in that superpixel?
            # Or globally? The source is ambiguous.
            # Let's assume global normalization for the probability map.
            max_prob = np.max(sp_probs)
            if max_prob > 0:
                sp_probs = sp_probs / max_prob
            new_sampling_probabilities.append(sp_probs)
            
        sampling_probabilities = new_sampling_probabilities
        
        # Regenerate sampled rays according to the probability map
        # This is handled in the next iteration's sampling step.
        
    return model
```

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. However, the method involves additional SLIC segmentation steps, which adds computational overhead compared to random sampling. The source notes that superpixels reduce computational overhead per instance compared to quadtrees [S1].
*   **Memory Complexity:** Not explicitly given. Requires storing superpixel maps and error maps.
*   **Measured Performance:** The method was tested on synthetic and real datasets using Instant-NGP and DVGO baselines on a single NVIDIA RTX 4090 GPU [S1]. It improves rendering quality of neural field baselines for high-resolution inputs [S1].
*   **Scaling:** The method is designed to handle large-scale image blocks, addressing challenges faced by methods that guide computation through color gradients [S1].
*   **Strengths:** Resolves inadequate partitioning issues caused by structures like quadtrees, reduces computational overhead per instance, and focuses computation on complex regions [S1].
*   **Limitations:** Requires additional segmentation steps, which may increase training time. The effectiveness depends on the choice of hyperparameters like $\beta_0$ and $\lambda_{sp}$ [S1].
*   **When to use:** Suitable for high-resolution NeRF training where slow convergence and scattered sampling are issues [S1].
*   **When not to use:** Not suitable for low-resolution inputs or when computational resources are extremely limited due to the overhead of segmentation [S1].

## Background
*   **Origin:** Proposed by Zhongwei Xuan et al. in 2024 [S1].
*   **History and Development:** The method addresses limitations of existing sampling strategies like quadtrees and color gradient-based methods, which struggle with large-scale images and fixed shapes [S1]. It integrates with popular neural field methods like Instant-NGP and DVGO [S1].
*   **Where it is used:** Integrated with Instant-NGP (using nerf-template/torch-ngp implementation) and DVGO baselines [S1].

## Sources
[S1] Superpixel-based Efficient Sampling for Learning Neural Fields from Large Input, https://doi.org/10.1145/3664647.3681299, fetched 2026-11-08
