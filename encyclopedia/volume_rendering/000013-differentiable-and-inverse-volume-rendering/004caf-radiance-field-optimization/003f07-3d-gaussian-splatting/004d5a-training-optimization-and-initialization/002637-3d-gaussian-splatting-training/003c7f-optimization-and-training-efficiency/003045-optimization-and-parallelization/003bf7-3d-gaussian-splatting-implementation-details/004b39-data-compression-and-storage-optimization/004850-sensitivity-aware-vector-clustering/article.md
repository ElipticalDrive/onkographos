# Sensitivity-Aware Vector Clustering

Also called: Color Compression, Sensitivity-Aware Scene Compression, Sensitivity-aware k-Means

## Overview
Sensitivity-Aware Vector Clustering is a compression scheme for 3D Gaussian Splatting that reduces storage requirements by clustering spherical harmonic (SH) coefficients and Gaussian shape features (scale and rotation) into compact codebooks using sensitivity-aware vector quantization [S1]. It distinguishes itself from standard k-Means by weighting the distance between vectors and centroids based on a sensitivity measure, prioritizing the preservation of parameters that significantly affect image quality [S1]. Unlike post-training quantization, this method derives a sensitivity measure to guide the clustering process, ensuring that high-impact parameters are either preserved or handled with higher fidelity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of optimized 3D Gaussian kernels $\{G_i\}$, each defined by position, opacity, rotation, scale, and SH coefficients. A set of $N$ training images with known camera parameters.
*   **Output:** Two codebooks: one for SH coefficients (color) and one for Gaussian shape parameters (rotation and scale). Each Gaussian is represented by two indices (32-bit unsigned integers) pointing to entries in these codebooks, alongside its position, opacity, and other non-clustered parameters.

**Data Structures:**
1.  **Codebook (Color):** Stores SH coefficient vectors. Size $K \times D_{SH}$, where $K$ is the codebook size and $D_{SH}$ is the dimension of the SH vector (e.g., 48 for degree 3).
2.  **Codebook (Shape):** Stores shape parameters. Each entry consists of 4 scalar values for a quaternion (rotation) and 3 scalar values for scaling (normalized scale vector and scalar magnitude) [S1].
3.  **Gaussian Representation:** For each Gaussian $i$, store:
    *   Position (3 floats)
    *   Opacity (1 float)
    *   Index into Color Codebook (1 uint32)
    *   Index into Shape Codebook (1 uint32)
    *   [G] Other parameters like view-dependent SH base or position derivatives if applicable, though the core compression targets SH and Shape.

**Sensitivity Calculation:**
The sensitivity $S(p)$ of a parameter $p$ is computed to measure its contribution to the training images [S1].
1.  Compute the total image energy $E$, defined as the sum of RGB components over all pixels in all training images [S1].
2.  Compute the gradient of $E$ with respect to parameter $p$ using a single backward pass over each of the $N$ training images [S1].
3.  The sensitivity $S(p)$ is defined as:
    $$ S(p) = \frac{\sum_{i=1}^{N} \frac{\partial E_i}{\partial p_i}}{\sum_{i=1}^{N} P_i} $$
    where $E_i$ is the error/image contribution for image $i$, and $P_i$ is the number of pixels in image $i$ [S1].
4.  For a vector $x$ (e.g., a set of SH coefficients or a shape vector), the sensitivity $S(x)$ is the maximum sensitivity over its components $d$:
    $$ S(x) = \max_d S(x_d) $$
    where $S(x_d)$ is the sensitivity of component $d$ [S1].

**Clustering Algorithm:**
The method uses two separate codebooks: one for color (SH coefficients) and one for shape (rotation and scale) [S1].

1.  **Initialization:** Initialize codebook centroids randomly with a uniform distribution within the minimum and maximum values of each parameter [S1].
2.  **High-Sensitivity Handling:** Identify vectors (SH coefficients or shape parameters) with sensitivity higher than a threshold $\beta_c$ (for color) or $\beta_g$ (for shape). These vectors are not clustered but are added directly to the codebook to preserve high-impact features [S1].
3.  **Batched Clustering:** For the remaining vectors, use a batched k-Means strategy [S1]:
    *   Pick a random subset (batch) of vectors.
    *   Compute pairwise weighted distances between vectors $x$ and codebook vectors $c_k$:
        $$ D(x, c_k) = S(x) \|x - c_k\|_2^2 $$
        where $S(x)$ is the sensitivity of vector $x$ [S1].
    *   Assign each vector to the centroid $c_k$ with the minimum distance.
    *   Update centroids using the weighted average:
        $$ c_k = \frac{1}{\sum_{x_i \in A(k)} S(x_i)} \sum_{x_i \in A(k)} S(x_i) x_i $$
        where $A(k)$ is the set of vectors assigned to centroid $c_k$ [S1].
    *   Update centroids using a moving average with decay factor $\lambda_d$:
        $$ c_k^{new} = \lambda_d c_k^{old} + (1 - \lambda_d) c_k^{updated} $$
        [S1].
4.  **Normalization (Shape Codebook):** For the shape codebook, clustering normalized covariance matrices can lead to non-unit scaling vectors due to floating-point errors. After each update step, re-normalize each codebook vector by dividing it through the trace of the covariance metric [S1].
5.  **Decomposition:** After clustering, decompose each codebook entry (covariance matrix) into rotation (quaternion) and scale parameters using eigenvalue decomposition. This is required for quantization-aware training [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_sensitivity(optimization_state, training_images):
    """
    Computes sensitivity S(p) for each parameter p.
    optimization_state: Contains current Gaussian parameters.
    training_images: List of training images and camera poses.
    Returns: Dictionary mapping parameter names to sensitivity values.
    """
    sensitivities = {}
    # Compute total image energy E
    E = 0
    for img in training_images:
        # Render image with current state
        rendered_img = render_gaussians(optimization_state, img.camera)
        E += np.sum(rendered_img)
    
    # Compute gradients dE/dp for each parameter
    # This requires a single backward pass per image or batched
    gradients = compute_gradients(optimization_state, training_images, E)
    
    for p in optimization_state.parameters:
        # S(p) = sum(dE_i/dp_i) / sum(P_i)
        # P_i is number of pixels in image i
        sum_grad = np.sum(gradients[p])
        sum_pixels = sum(img.num_pixels for img in training_images)
        sensitivities[p] = sum_grad / sum_pixels
        
    return sensitivities

def sensitivity_aware_kmeans(vectors, sensitivities, codebook_size, decay_factor, max_iterations):
    """
    Performs sensitivity-aware k-Means clustering.
    vectors: Array of shape (N, D)
    sensitivities: Array of shape (N,) containing S(x) for each vector
    codebook_size: K
    decay_factor: lambda_d
    max_iterations: Number of update steps
    Returns: Codebook centroids of shape (K, D)
    """
    # Initialize centroids randomly within min/max of vectors
    centroids = np.random.uniform(vectors.min(axis=0), vectors.max(axis=0), (codebook_size, vectors.shape[1]))
    
    for step in range(max_iterations):
        # Pick random subset (batch)
        batch_indices = np.random.choice(vectors.shape[0], size=batch_size, replace=False)
        batch_vectors = vectors[batch_indices]
        batch_sensitivities = sensitivities[batch_indices]
        
        # Compute weighted distances
        # D(x, ck) = S(x) * ||x - ck||^2
        # Expand dimensions for broadcasting
        # vectors: (B, D), centroids: (K, D) -> dists: (B, K)
        diff = batch_vectors[:, np.newaxis, :] - centroids[np.newaxis, :, :]
        sq_dist = np.sum(diff**2, axis=2)
        weighted_dist = batch_sensitivities[:, np.newaxis] * sq_dist
        
        # Assign to nearest centroid
        assignments = np.argmin(weighted_dist, axis=1)
        
        # Update centroids using weighted average
        new_centroids = np.zeros_like(centroids)
        for k in range(codebook_size):
            mask = assignments == k
            if np.any(mask):
                assigned_vectors = batch_vectors[mask]
                assigned_sens = batch_sensitivities[mask]
                weight_sum = np.sum(assigned_sens)
                if weight_sum > 0:
                    new_centroids[k] = np.sum(assigned_vectors * assigned_sens[:, np.newaxis], axis=0) / weight_sum
                else:
                    new_centroids[k] = centroids[k] # Keep old if no assignments
            else:
                new_centroids[k] = centroids[k]
        
        # Moving average update
        centroids = decay_factor * centroids + (1 - decay_factor) * new_centroids
        
        # Re-normalize shape codebook if applicable
        if is_shape_codebook:
            centroids = re_normalize_shape_codebook(centroids)
            
    return centroids

def re_normalize_shape_codebook(centroids):
    """
    Re-normalizes shape codebook entries by dividing through the trace of the covariance metric.
    centroids: Array of shape (K, 7) [4 quaternion + 3 scale]
    Returns: Normalized centroids
    """
    # Convert quaternion and scale to covariance matrix
    covariances = quaternion_scale_to_covariance(centroids)
    # Compute trace
    traces = np.trace(covariances, axis1=1, axis2=2)
    # Divide covariance by trace
    normalized_covariances = covariances / traces[:, np.newaxis, np.newaxis]
    # Convert back to quaternion and scale
    normalized_centroids = covariance_to_quaternion_scale(normalized_covariances)
    return normalized_centroids
```

**Parameters:**
*   $\lambda_d$: Decay factor for the moving average in batched clustering. Typical value: 0.8 [S1].
*   Batch size for color features: $2^{18}$ [S1].
*   Batch size for Gaussian parameters: $2^{20}$ [S1].
*   Codebook size $K$: 4096 [S1].
*   $\beta_c$: Sensitivity threshold for SH coefficients. Gaussians with sensitivity $> \beta_c$ are excluded from clustering and added directly to the codebook. Typical value: $6 \times 10^{-7}$ [S1].
*   $\beta_g$: Sensitivity threshold for Gaussian shape parameters. Typical value: $3 \times 10^{-6}$ [S1].

**Edge Cases and Pitfalls:**
*   **Floating Point Errors:** Clustering normalized covariance matrices can lead to non-unit scaling vectors due to floating-point errors. This is handled by re-normalizing each codebook vector after each update step by dividing it through the trace of the covariance metric [S1].
*   **High Sensitivity Vectors:** A small percentage of SH coefficients (< 5%) may have very high sensitivity. These must be identified and added directly to the codebook to prevent rendering error [S1].

## Operating characteristics
*   **Time Complexity:** The clustering step is iterative. The sensitivity calculation requires a backward pass over all training images. The clustering itself uses a batched strategy, making it scalable to large numbers of Gaussians.
*   **Memory Complexity:** Storage is reduced by a factor of up to 31× on real-world scenes [S1]. Each Gaussian stores two 32-bit indices instead of full SH coefficients and shape parameters.
*   **Performance:** The compressed representation can be rendered with hardware rasterization on lightweight GPUs at up to 4× higher framerates than the uncompressed baseline [S1].
*   **Strengths:** Significant storage reduction with minimal visual degradation. Efficient rendering on low-power devices.
*   **Limitations:** Requires an initial optimization pass to compute sensitivities. The clustering process adds overhead to the compression pipeline.

## Background
Sensitivity-Aware Vector Clustering was proposed in the paper "Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis" [S1]. It builds on the observation that SH coefficients and Gaussian shape parameters are highly redundant and that sensitivity to image quality is not uniform across parameters. The method is inspired by volumetric NeRF compression and deep network weight quantization [S1].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
