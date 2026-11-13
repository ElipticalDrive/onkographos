# Sensitivity-Aware Vector Clustering

Also called: Color Compression, Sensitivity-Aware Scene Compression, Sensitivity-aware k-Means

## Overview
Sensitivity-Aware Vector Clustering is a compression scheme for 3D Gaussian Splatting that reduces storage requirements by clustering spherical harmonic (SH) coefficients and Gaussian shape features (scale and rotation) into compact codebooks using sensitivity-aware vector quantization [S1]. It sets itself apart from standard k-Means by weighting the distance metric with a sensitivity measure derived from the gradient of the rendering error, thereby prioritizing the preservation of parameters that significantly affect image quality [S1]. Unlike post-training quantization, this method guides the clustering process during the compression scheme derivation to minimize visual degradation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of optimized 3D Gaussians, each defined by position, opacity, rotation, scale, and SH coefficients. A set of $N$ training images with known camera parameters.
*   **Output:** Two codebooks (one for color/SH, one for shape/rotation-scale) and a compressed representation where each Gaussian is stored with two indices (32-bit unsigned integers) pointing to entries in these codebooks, alongside other uncompressed attributes [S1].

**Data Structures**
*   **Codebook (Color):** Stores SH coefficient vectors. Size $K \times D_{SH}$, where $K$ is the codebook size and $D_{SH}$ is the dimension of the SH vector (e.g., 48 for degree 3) [S1].
*   **Codebook (Shape):** Stores shape parameters. Each entry consists of 4 scalar values for a quaternion (rotation) and 3 scalar values for scaling [S1].
*   **Gaussian Index:** A 32-bit unsigned integer index into the respective codebook [S1].

**Sensitivity Calculation**
The sensitivity $S(p)$ of a parameter $p$ is calculated based on its contribution to the training images. The total image energy $E$ is the sum of RGB components over all pixels. The sensitivity is computed via the gradient of $E$ with respect to $p$ using a single backward pass over each training image [S1].

For a parameter $p$, the sensitivity is defined as:
$$ S(p) = \frac{\sum_{i=1}^{N} \frac{\partial E_i}{\partial p_i}}{\sum_{i=1}^{N} P_i} $$
where $N$ is the number of training images, $E_i$ is the error/image contribution for image $i$, and $P_i$ is the number of pixels in image $i$ [S1].

For a vector $x$ (representing SH coefficients or shape parameters), the sensitivity $S(x)$ is the maximum sensitivity over its components $d$:
$$ S(x) = \max_d S(x_d) $$
where $S(x_d)$ is the sensitivity of component $d$ [S1].

**Clustering Algorithm**
The method uses two separate codebooks: one for SH coefficients (Color Compression) and one for Gaussian shape features (Gaussian Shape Compression) [S1].

1.  **Initialization:** Codebooks are initialized randomly with a uniform distribution within the minimum and maximum values of each parameter [S1].
2.  **Thresholding (Outliers):**
    *   For SH coefficients: Identify vectors with sensitivity higher than a threshold $\beta_c$. These vectors are not clustered but are added directly to the codebook [S1].
    *   For Shape parameters: Identify Gaussians with maximum sensitivity over a threshold $\beta_g$. These are also excluded from clustering and added directly to the codebook [S1].
3.  **Weighted Distance:** For the remaining vectors, the distance between a vector $x$ and a centroid $c_k$ is weighted by the vector's sensitivity:
    $$ D(x, c_k) = S(x) \|x - c_k\|_2^2 $$
    where $\| \cdot \|_2^2$ is the squared Euclidean distance [S1].
4.  **Assignment:** Each vector $x$ is assigned to the centroid $c_k$ with the minimum weighted distance [S1].
5.  **Centroid Update:** The centroid $c_k$ is updated using the weighted average of all vectors $x_i$ assigned to it (set $A(k)$):
    $$ c_k = \frac{1}{\sum_{x_i \in A(k)} S(x_i)} \sum_{x_i \in A(k)} S(x_i) x_i $$
    [S1].
6.  **Batched Clustering:** To improve performance, a batched strategy is used. In each update step, a random subset of vectors is picked. The assignment and update steps are computed for this subset. The centroids are then updated using a moving average with a decay factor $\lambda_d$ [S1].
7.  **Normalization (Shape Codebook):** For the shape codebook, clustering normalized covariance matrices can lead to non-unit scaling vectors due to floating-point errors. After each update step, each codebook vector is re-normalized by dividing it through the trace of the covariance metric [S1].
8.  **Decomposition:** After clustering, each codebook entry in the shape codebook (covariance matrix) is decomposed into rotation and scale parameters using eigenvalue decomposition. This is required for quantization-aware training, as direct optimization of the matrix is not possible [S1]. The final codebook entry is encoded as 4 quaternion values and 3 scaling values [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sensitivity_aware_clustering(gaussians, training_images, codebook_size_K, beta_c, beta_g, lambda_d, batch_size_color, batch_size_shape, num_steps_color, num_steps_shape):
    """
    Performs sensitivity-aware vector clustering for 3D Gaussian Splatting.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - sh_coeffs: Vector of SH coefficients
                   - rotation: Rotation matrix or quaternion
                   - scale: Scaling vector
                   - position: 3D position
        training_images: List of training images with known camera parameters.
        codebook_size_K: Number of centroids in the codebook.
        beta_c: Sensitivity threshold for SH coefficients.
        beta_g: Sensitivity threshold for shape parameters.
        lambda_d: Decay factor for moving average in batched clustering.
        batch_size_color: Batch size for color features.
        batch_size_shape: Batch size for Gaussian parameters.
        num_steps_color: Number of update steps for SH clustering.
        num_steps_shape: Number of update steps for shape clustering.
        
    Returns:
        codebook_color: List of SH coefficient vectors (centroids).
        codebook_shape: List of shape parameters (quaternions and scales).
        indices_color: List of indices into codebook_color for each Gaussian.
        indices_shape: List of indices into codebook_shape for each Gaussian.
    """
    
    # 1. Calculate Sensitivity
    # Compute sensitivity for each parameter/component using gradients of image energy
    # S(p) = sum(dE_i/dp_i) / sum(P_i)
    # S(x) = max_d(S(x_d))
    sensitivities_sh = calculate_sensitivity_vector(gaussians, training_images, type='sh')
    sensitivities_shape = calculate_sensitivity_vector(gaussians, training_images, type='shape')
    
    # 2. Initialize Codebooks
    # Random uniform distribution within min/max of parameters
    codebook_color = initialize_codebook_random(codebook_size_K, dimension=sh_dimension)
    codebook_shape = initialize_codebook_random(codebook_size_K, dimension=shape_dimension)
    
    # 3. Handle Outliers (High Sensitivity)
    # Gaussians with sensitivity > beta are added directly to codebook
    outlier_indices_sh = [i for i, s in enumerate(sensitivities_sh) if s > beta_c]
    outlier_indices_shape = [i for i, s in enumerate(sensitivities_shape) if s > beta_g]
    
    # Add outliers to codebooks (Note: Source implies they are added to the codebook,
    # effectively increasing the effective codebook size or occupying specific slots.
    # For implementation, we reserve slots or append. Here we assume they occupy
    # the first len(outliers) slots or are managed separately. 
    # The source says "added directly to the codebook", implying they are part of the final set.
    # We will assume the codebook size K includes these or K is the size for the clustered part.
    # Given "Codebook size K: 4096" and "percentage < 5%", we assume K is the total capacity.
    # Implementation detail: We fill the codebook with outliers first, then cluster the rest.
    
    codebook_color[:len(outlier_indices_sh)] = [gaussians[i].sh_coeffs for i in outlier_indices_sh]
    codebook_shape[:len(outlier_indices_shape)] = [gaussians[i].shape_params for i in outlier_indices_shape]
    
    # Remaining vectors for clustering
    remaining_indices_sh = [i for i in range(len(gaussians)) if i not in outlier_indices_sh]
    remaining_indices_shape = [i for i in range(len(gaussians)) if i not in outlier_indices_shape]
    
    # Initialize centroids for the remaining vectors (random uniform)
    # Note: Source says "initialized randomly... within min and max".
    # We initialize the rest of the codebook slots.
    start_idx_sh = len(outlier_indices_sh)
    start_idx_shape = len(outlier_indices_shape)
    
    codebook_color[start_idx_sh:] = initialize_codebook_random(codebook_size_K - start_idx_sh, dimension=sh_dimension)
    codebook_shape[start_idx_shape:] = initialize_codebook_random(codebook_size_K - start_idx_shape, dimension=shape_dimension)
    
    # 4. Clustering Loop for Color (SH)
    centroids_sh = codebook_color[start_idx_sh:]
    for step in range(num_steps_color):
        # Pick random subset
        batch_indices = random_sample(remaining_indices_sh, batch_size_color)
        batch_vectors = [gaussians[i].sh_coeffs for i in batch_indices]
        batch_sensitivities = [sensitivities_sh[i] for i in batch_indices]
        
        # Assignment: Find closest centroid using weighted distance
        # D(x, ck) = S(x) * ||x - ck||^2
        assignments = []
        for x, s in zip(batch_vectors, batch_sensitivities):
            min_dist = infinity
            min_idx = -1
            for k, ck in enumerate(centroids_sh):
                dist = s * squared_euclidean_distance(x, ck)
                if dist < min_dist:
                    min_dist = dist
                    min_idx = k
            assignments.append(min_idx)
        
        # Update: Weighted average
        new_centroids = copy(centroids_sh)
        for k in range(len(centroids_sh)):
            assigned_vectors = [batch_vectors[j] for j, a in enumerate(assignments) if a == k]
            assigned_sensitivities = [batch_sensitivities[j] for j, a in enumerate(assignments) if a == k]
            
            if assigned_vectors:
                sum_s = sum(assigned_sensitivities)
                weighted_sum = sum(s * v for s, v in zip(assigned_sensitivities, assigned_vectors))
                new_centroid = weighted_sum / sum_s
                # Moving average update
                new_centroids[k] = (1 - lambda_d) * centroids_sh[k] + lambda_d * new_centroid
        
        centroids_sh = new_centroids

    # 5. Clustering Loop for Shape
    centroids_shape = codebook_shape[start_idx_shape:]
    for step in range(num_steps_shape):
        batch_indices = random_sample(remaining_indices_shape, batch_size_shape)
        batch_vectors = [gaussians[i].shape_params for i in batch_indices]
        batch_sensitivities = [sensitivities_shape[i] for i in batch_indices]
        
        assignments = []
        for x, s in zip(batch_vectors, batch_sensitivities):
            min_dist = infinity
            min_idx = -1
            for k, ck in enumerate(centroids_shape):
                dist = s * squared_euclidean_distance(x, ck)
                if dist < min_dist:
                    min_dist = dist
                    min_idx = k
            assignments.append(min_idx)
        
        new_centroids = copy(centroids_shape)
        for k in range(len(centroids_shape)):
            assigned_vectors = [batch_vectors[j] for j, a in enumerate(assignments) if a == k]
            assigned_sensitivities = [batch_sensitivities[j] for j, a in enumerate(assignments) if a == k]
            
            if assigned_vectors:
                sum_s = sum(assigned_sensitivities)
                weighted_sum = sum(s * v for s, v in zip(assigned_sensitivities, assigned_vectors))
                new_centroid = weighted_sum / sum_s
                new_centroids[k] = (1 - lambda_d) * centroids_shape[k] + lambda_d * new_centroid
        
        centroids_shape = new_centroids
        
        # Renormalization for Shape Codebook
        # Divide through the trace of the covariance metric to counteract floating point errors
        for k in range(len(centroids_shape)):
            centroids_shape[k] = renormalize_covariance_vector(centroids_shape[k])

    # 6. Decomposition and Final Codebook Assembly
    # Decompose shape codebook entries into quaternion and scale
    final_codebook_shape = []
    for entry in centroids_shape:
        q, s = eigen_decompose_shape(entry)
        final_codebook_shape.append((q, s))
    
    # Combine outliers and clustered centroids
    final_codebook_color = codebook_color[:start_idx_sh] + centroids_sh
    final_codebook_shape = codebook_shape[:start_idx_shape] + final_codebook_shape
    
    # 7. Generate Indices
    # Assign each Gaussian to its closest centroid in the final codebook
    indices_color = []
    indices_shape = []
    
    for i, g in enumerate(gaussians):
        # Find index in final_codebook_color
        min_dist = infinity
        min_idx = -1
        for k, ck in enumerate(final_codebook_color):
            dist = squared_euclidean_distance(g.sh_coeffs, ck)
            if dist < min_dist:
                min_dist = dist
                min_idx = k
        indices_color.append(min_idx)
        
        # Find index in final_codebook_shape
        min_dist = infinity
        min_idx = -1
        for k, (q, s) in enumerate(final_codebook_shape):
            # Compare with decomposed parameters
            dist = squared_euclidean_distance(g.shape_params, (q, s))
            if dist < min_dist:
                min_dist = dist
                min_idx = k
        indices_shape.append(min_idx)
        
    return final_codebook_color, final_codebook_shape, indices_color, indices_shape

def calculate_sensitivity_vector(gaussians, training_images, type):
    """
    Calculates sensitivity S(x) = max_d(S(x_d)) for each Gaussian.
    S(p) = sum(dE_i/dp_i) / sum(P_i)
    """
    sensitivities = []
    for g in gaussians:
        if type == 'sh':
            params = g.sh_coeffs
        else:
            params = g.shape_params
            
        # Compute gradient of E w.r.t params for each training image
        # This requires a backward pass
        grad_sum = 0
        pixel_sum = 0
        for img in training_images:
            # Compute rendering error E_i
            # Compute gradient dE_i/dp
            grad = compute_gradient(img, g, params)
            grad_sum += grad
            pixel_sum += img.num_pixels
            
        # S(p) = grad_sum / pixel_sum
        # S(x) = max_d(S(x_d))
        s_p = grad_sum / pixel_sum
        s_x = max(abs(s_p)) # Assuming scalar sensitivity per component, take max
        sensitivities.append(s_x)
    return sensitivities

def renormalize_covariance_vector(v):
    """
    Re-normalizes a shape vector (covariance matrix representation) by dividing
    through the trace of the covariance metric.
    """
    # Convert vector to covariance matrix
    cov_matrix = vector_to_covariance(v)
    trace = sum(diagonal(cov_matrix))
    # Divide vector representation by trace? 
    # Source says "dividing it through the trace of the covariance metric".
    # This likely means scaling the covariance matrix by 1/tr(trace) and converting back.
    scaled_cov = cov_matrix / trace
    return covariance_to_vector(scaled_cov)
```

**Parameters**
*   $\lambda_d$: Decay factor for the moving average in batched clustering. Typical value: 0.8 [S1].
*   Batch size for color features: $2^{18}$ [S1].
*   Batch size for Gaussian parameters: $2^{20}$ [S1].
*   Codebook size $K$: 4096 [S1].
*   $\beta_c$: Sensitivity threshold for SH coefficients. Gaussians with sensitivity $> \beta_c$ are excluded from clustering. Typical value: $6 \times 10^{-7}$ [S1].
*   $\beta_g$: Sensitivity threshold for shape parameters. Typical value: $3 \times 10^{-6}$ [S1].
*   Optimization steps: 5000 steps of quantization-aware fine-tuning after clustering [S1].

**Edge Cases and Pitfalls**
*   **Floating Point Errors:** Clustering normalized covariance matrices can lead to non-unit scaling vectors due to floating-point errors. This is handled by re-normalizing each codebook vector after each update step by dividing it through the trace of the covariance metric [S1].
*   **High Sensitivity Outliers:** A small percentage of SH coefficients (< 5%) have high sensitivity. These must be identified and excluded from clustering to keep rendering error low [S1].

## Operating characteristics
*   **Compression Rate:** Achieves up to 31× compression on real-world scenes with minimal visual degradation [S1].
*   **Rendering Performance:** Enables real-time rendering on lightweight GPUs, up to 4× higher framerates than optimized GPU compute pipelines [S1].
*   **Storage:** Replaces high-precision per-Gaussian attributes with 32-bit indices into compact codebooks [S1].
*   **Strengths:** Reduces memory consumption significantly, suitable for network streaming and low-power devices [S1].
*   **Limitations:** Requires a quantization-aware fine-tuning step to recover information lost during clustering [S1].

## Background
*   **Origin:** Proposed in "Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis" (CVPR 2024) [S1].
*   **History:** Inspired by volumetric radiance field compression and deep network weight quantization [S1]. It addresses the redundancy in SH coefficients and Gaussian shape parameters which take up the majority of storage space in 3D Gaussian Splatting [S1].
*   **Usage:** Used for compressed 3D Gaussian splat representation to enable efficient rendering on lightweight GPUs and web browsers via WebGPU [S1].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
