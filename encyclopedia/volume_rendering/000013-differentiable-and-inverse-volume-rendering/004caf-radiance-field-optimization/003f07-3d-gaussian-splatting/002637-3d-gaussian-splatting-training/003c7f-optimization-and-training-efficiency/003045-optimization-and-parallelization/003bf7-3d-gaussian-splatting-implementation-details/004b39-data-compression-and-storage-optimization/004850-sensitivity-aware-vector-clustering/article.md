# Sensitivity-Aware Vector Clustering

Also called: Color Compression, Sensitivity-Aware Scene Compression, Sensitivity-aware k-Means

## Overview
Sensitivity-Aware Vector Clustering is a compression scheme for 3D Gaussian Splatting that reduces storage requirements by clustering Spherical Harmonic (SH) coefficients and Gaussian shape features (scale and rotation) into compact codebooks using sensitivity-aware vector quantization [S1]. It distinguishes itself from standard k-Means by weighting the distance metric with a sensitivity measure derived from the gradient of the rendering error, thereby prioritizing the preservation of parameters that significantly affect image quality [S1]. Unlike post-training quantization, this method guides the clustering process during the compression derivation to minimize visual degradation [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A set of optimized 3D Gaussian kernels, each defined by position, opacity, scale, rotation, and SH coefficients for color.
- **Output:** Two codebooks (one for color/SH, one for shape/rotation-scale) and a list of Gaussians where each Gaussian is represented by two indices (one for each codebook) instead of the full parameter vectors.

**Data Structures:**
1.  **Codebook Entry (Shape):** Stores 4 scalar values for a quaternion (rotation) and 3 scalar values for scaling [S1].
2.  **Codebook Entry (Color):** Stores the compressed SH coefficient vector [S1].
3.  **Gaussian Representation:** Stores two 32-bit unsigned integer indices pointing to entries in the respective codebooks, alongside other uncompressed attributes (position, opacity) [S1].

**Sensitivity Measure:**
The sensitivity $S$ of a parameter $p$ is defined as the contribution of that parameter to the training images [S1]. It is computed using the gradient of the total image energy $E$ (sum of RGB components) with respect to $p$ [S1].

For a single parameter $p$:
$$ S(p) = \frac{\sum_{i=1}^{N} \frac{\partial E_i}{\partial p_i}}{\sum_{i=1}^{N} P_i} $$
where $N$ is the number of training images, $E_i$ is the error/image contribution for image $i$, and $P_i$ is the number of pixels in image $i$ [S1].

For a vector $x$ (e.g., a set of SH coefficients or a shape vector), the sensitivity is the maximum sensitivity over its components $d$:
$$ S(x) = \max_d S(x_d) $$
where $S(x_d)$ is the sensitivity of component $d$ [S1].

**Algorithm Steps:**

1.  **Sensitivity Calculation:** Compute the sensitivity $S(x)$ for all input vectors (SH coefficients and shape parameters) using a single backward pass over the training images to calculate gradients of the image energy [S1].

2.  **Codebook Initialization:** Initialize codebooks randomly with a uniform distribution within the minimum and maximum values of each parameter [S1]. The default codebook size $K$ is 4096 [S1].

3.  **High-Sensitivity Handling (SH):** Identify SH vectors with sensitivity higher than a threshold $\beta_c$. These vectors are excluded from clustering and added directly to the codebook [S1]. The typical value for $\beta_c$ is $6 \times 10^{-7}$ [S1].

4.  **High-Sensitivity Handling (Shape):** Identify shape vectors with sensitivity higher than a threshold $\beta_g$. These are also excluded from clustering and added directly to the codebook. The typical value for $\beta_g$ is $3 \times 10^{-6}$ [S1].

5.  **Clustering (Sensitivity-Aware k-Means):** Cluster the remaining vectors using sensitivity-aware k-Means.
    -   **Distance Metric:** The weighted distance $D$ between a vector $x$ and a centroid $c_k$ is:
        $$ D(x, c_k) = S(x) \|x - c_k\|_2^2 $$
    -   **Assignment:** Assign each vector $x$ to the centroid $c_k$ with the minimum distance [S1].
    -   **Centroid Update:** Update centroid $c_k$ using the weighted average of assigned vectors $A(k)$:
        $$ c_k = \frac{1}{\sum_{x_i \in A(k)} S(x_i)} \sum_{x_i \in A(k)} S(x_i) x_i $$
    -   **Batched Strategy:** For performance, use a batched strategy. Pick a random subset of vectors, compute the update, and update centroids using a moving average with decay factor $\lambda_d$ [S1]. Typical $\lambda_d$ is 0.8 [S1].
    -   **Batch Sizes:** Use a batch size of $2^{18}$ for color features and $2^{20}$ for Gaussian parameters [S1].
    -   **Iterations:** Perform 800 update steps for Gaussians and 100 for SH coefficients [S1].

6.  **Normalization (Shape Codebook):** Due to floating-point errors, clustering normalized covariance matrices may result in non-unit scaling vectors. Re-normalize each codebook vector after each update step by dividing it through the trace of the covariance metric [S1].

7.  **Decomposition (Shape Codebook):** Decompose each codebook entry (covariance matrix) into rotation (quaternion) and scale parameters using eigenvalue decomposition. This is required for quantization-aware training [S1].

8.  **Quantization-Aware Fine-Tuning:** Fine-tune the scene parameters at reduced bit-rates using quantization-aware training to recover information lost during clustering. Perform 5000 optimization steps [S1].

9.  **Encoding:** Store indices into the codebooks as 32-bit unsigned integers. Compress the data using DEFLATE (LZ77 + Huffman) after ordering Gaussians along a Z-order curve (Morton order) to exploit spatial coherence [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_sensitivity(optimization_state, training_images):
    """
    Computes sensitivity S(x) for a set of vectors x.
    optimization_state: Contains parameters p (SH coeffs, shape params).
    training_images: List of target images for reconstruction.
    Returns: Dictionary mapping parameter keys to sensitivity values.
    """
    sensitivities = {}
    # Compute gradients of image energy E w.r.t parameters p
    # This requires a single backward pass per image or batch
    for img in training_images:
        # Render scene with current optimization_state
        rendered_img = render(optimization_state)
        # Compute loss E (e.g., MSE between rendered_img and img)
        loss = compute_loss(rendered_img, img)
        # Backward pass to get gradients dE/dp
        gradients = backward_pass(loss, optimization_state)
        
        # Accumulate sensitivities
        # S(p) = sum(dE/dp) / sum(P_i)
        # Note: P_i is number of pixels. Assuming normalized loss or handling P_i in aggregation.
        # Here we assume gradients are already scaled or P_i is handled in the final aggregation.
        for key, grad in gradients.items():
            if key not in sensitivities:
                sensitivities[key] = np.zeros_like(grad)
            sensitivities[key] += grad
            
    # Normalize by total pixels (simplified aggregation)
    total_pixels = sum(img.num_pixels for img in training_images)
    for key in sensitivities:
        sensitivities[key] /= total_pixels
        
    # For a vector x, S(x) = max_d S(x_d)
    # This is handled per-vector during clustering assignment
    return sensitivities

def sensitivity_aware_kmeans(vectors, sensitivities, codebook_size, decay_factor, batch_size, max_iterations):
    """
    Performs sensitivity-aware k-Means clustering.
    vectors: List of input vectors (e.g., SH coeffs or shape params).
    sensitivities: Corresponding sensitivity values S(x) for each vector.
    codebook_size: K, number of centroids.
    decay_factor: lambda_d for moving average.
    batch_size: Number of vectors per update step.
    max_iterations: Number of update steps.
    Returns: Codebook (centroids).
    """
    # Initialize centroids randomly within min/max of vectors
    centroids = initialize_centroids(vectors, codebook_size)
    
    for iteration in range(max_iterations):
        # Pick random subset (batch)
        indices = np.random.choice(len(vectors), batch_size, replace=False)
        batch_vectors = vectors[indices]
        batch_sensitivities = sensitivities[indices]
        
        # Assign vectors to nearest centroid using weighted distance
        # D(x, ck) = S(x) * ||x - ck||^2
        distances = compute_weighted_distances(batch_vectors, batch_sensitivities, centroids)
        assignments = np.argmin(distances, axis=1)
        
        # Update centroids using moving average
        new_centroids = np.copy(centroids)
        for k in range(codebook_size):
            assigned_indices = np.where(assignments == k)[0]
            if len(assigned_indices) > 0:
                assigned_batch_vectors = batch_vectors[assigned_indices]
                assigned_batch_sensitivities = batch_sensitivities[assigned_indices]
                
                # Weighted average
                # ck = sum(S(xi)*xi) / sum(S(xi))
                weighted_sum = np.sum(assigned_batch_sensitivities[:, np.newaxis] * assigned_batch_vectors, axis=0)
                weight_total = np.sum(assigned_batch_sensitivities)
                
                if weight_total > 0:
                    new_centroid = weighted_sum / weight_total
                else:
                    new_centroid = centroids[k]
                    
                # Moving average update
                new_centroids[k] = decay_factor * new_centroids[k] + (1 - decay_factor) * new_centroid
                
        # Special handling for shape codebook: re-normalize
        # Divide by trace of covariance metric (simplified as norm for vector representation)
        # Note: The source mentions dividing by trace of covariance metric.
        # Assuming shape vectors are stored as covariance matrices or derived from them.
        # For vector representation, we might need to reconstruct covariance, normalize, and re-encode.
        # Here we assume a placeholder for the specific normalization step described.
        new_centroids = re_normalize_shape_centroids(new_centroids)
        
        centroids = new_centroids
        
    return centroids

def re_normalize_shape_centroids(centroids):
    """
    Re-normalizes shape centroids to handle floating point errors.
    Divides by the trace of the covariance metric.
    """
    normalized_centroids = []
    for centroid in centroids:
        # Reconstruct covariance matrix from quaternion/scale or vector representation
        # This step depends on the specific parameterization.
        # Assuming centroid is a covariance matrix or can be converted to one.
        cov_matrix = reconstruct_covariance(centroid)
        trace = np.trace(cov_matrix)
        if trace > 0:
            # Normalize
            normalized_cov = cov_matrix / trace
            # Decompose back to quaternion/scale
            normalized_centroid = decompose_covariance(normalized_cov)
        else:
            normalized_centroid = centroid
        normalized_centroids.append(normalized_centroid)
    return normalized_centroids

def decompose_covariance(cov_matrix):
    """
    Decomposes covariance matrix into rotation (quaternion) and scale.
    Uses eigenvalue decomposition.
    """
    eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
    # Extract scale and rotation from eigenvalues and eigenvectors
    # Scale is sqrt of eigenvalues (or similar, depending on parameterization)
    # Rotation is eigenvectors
    scale = np.sqrt(np.maximum(eigenvalues, 0))
    rotation = eigenvectors
    # Convert rotation matrix to quaternion
    quaternion = matrix_to_quaternion(rotation)
    return np.concatenate([quaternion, scale])

def compress_scene(gaussians, color_codebook, shape_codebook):
    """
    Compresses the scene by replacing parameters with codebook indices.
    """
    compressed_gaussians = []
    for g in gaussians:
        # Find index in color codebook
        color_idx = find_nearest_index(g.sh_coeffs, color_codebook)
        # Find index in shape codebook
        shape_idx = find_nearest_index(g.shape_params, shape_codebook)
        
        # Store indices instead of full parameters
        compressed_gaussians.append({
            'position': g.position,
            'opacity': g.opacity,
            'color_idx': color_idx,
            'shape_idx': shape_idx
        })
        
    # Order by Morton code for spatial coherence
    morton_codes = compute_morton_codes([g['position'] for g in compressed_gaussians])
    sorted_indices = np.argsort(morton_codes)
    compressed_gaussians = [compressed_gaussians[i] for i in sorted_indices]
    
    # Encode indices and other data using DEFLATE
    compressed_data = deflate_encode(compressed_gaussians)
    
    return compressed_data
```

## Operating characteristics
- **Time Complexity:** Clustering is iterative. The batched strategy reduces per-step cost. Typical settings use 800 steps for Gaussians and 100 for SH coefficients [S1].
- **Memory Complexity:** Reduces storage by up to 31× on real-world scenes [S1]. Codebook size is typically 4096 entries [S1].
- **Performance:** Enables real-time rendering on lightweight GPUs, achieving up to 4× higher framerates than uncompressed versions via optimized GPU compute pipelines [S1].
- **Strengths:** Maintains high visual quality with minimal degradation; efficient for network streaming and low-power devices [S1].
- **Limitations:** Requires a sensitivity calculation phase (backward pass) before clustering; quantization-aware fine-tuning adds training time [S1].

## Background
Sensitivity-Aware Vector Clustering was proposed in the paper "Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis" (CVPR 2024) [S1]. It builds on the observation that SH coefficients and Gaussian shape parameters in 3D Gaussian Splatting are highly redundant and consume the majority of storage space [S1]. The method is inspired by volumetric radiance field compression and deep network weight quantization [S1]. It is used to accelerate novel view synthesis on low-end devices and web browsers via WebGPU [S1].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
