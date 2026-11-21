# Instant-NGP with Scale-Space Augmentation

Also called: Instant-NGP adapted NeRF, NeRF with Instant-NGP extensions and scale-space augmentation, NeRF2Real NeRF

## Overview
This method adapts the Instant-NGP architecture [S1] for robust reconstruction from short mobile videos, specifically targeting sim2real transfer scenarios where compact geometry and anti-aliased rendering are critical [S1]. It differs from standard NeRF by replacing dense grids with multi-scale spatial hash grids for $O(1)$ lookup, adding a distortion loss for compactness, decomposing color into specular/diffuse components, and injecting scale-space variance as an MLP input for anti-aliasing [S1]. It builds on [Hash-grid anti-aliasing and regularization](15594-hash-grid-anti-aliasing-and-regularization.md) by explicitly providing the integrated scale variance $\Sigma$ to the decoder, allowing the network to interpolate samples in scale-space [S1].

## How it works
**Inputs:**
*   A short video ($\sim 5-6$ minutes) captured with a mobile camera (e.g., Google Pixel 6 rear camera) [S1].
*   Camera intrinsics and keyframe extrinsics estimated via COLMAP [S1].
*   $N \sim 1000$ keyframes extracted from the video [S1].
*   Video resolution $\ge 1080p$ [S1].

**Outputs:**
*   A trained NeRF model capable of novel view synthesis and collision geometry extraction [S1].
*   Rendering speed of $\sim 6$ms per frame on a V100 GPU [S1].

**Data Structures:**
*   **Multi-scale Spatial Hash Grid:** A set of hash tables, one per resolution level $\ell$. Each table maps hashed 3D coordinates to feature vectors. The layout follows the standard Instant-NGP structure: a global hash function maps a 3D point to an index in a flat feature array, with trilinear interpolation used for feature retrieval [S1].
*   **MLP Decoder:** A multi-layer perceptron that takes concatenated features from the hash grids and an additional scale variance input $\Sigma$ to predict volume density $\sigma$ and diffuse/specular color components [S1].

**Numbered Steps:**

1.  **Data Acquisition and Preprocessing:**
    *   Capture a short video of the static scene using a mobile camera. Ensure slow, even camera movement to reduce motion blur and minimize drastic viewpoint changes [S1].
    *   Fix white balance and brightness to ensure consistent lighting [S1].
    *   Extract $N \sim 1000$ keyframes from the video [S1].
    *   Use COLMAP to estimate camera intrinsics and keyframe extrinsics [S1].

2.  **Network Architecture Setup:**
    *   Initialize multi-scale spatial hash grids for feature encoding [S1].
    *   Initialize the MLP decoder with Swish activations instead of ReLU [S1].
    *   Add Layer Normalization before the final MLP layer [S1].
    *   Configure the output to predict separate specular and diffuse color components to improve reconstructed geometry [S1].

3.  **Training Loop (Per Ray):**
    *   For each training ray, sample points along the ray using a proposal network [S1].
    *   **Scale-Space Augmentation:**
        *   Sample a random blur variance $\sigma_{\text{blur}}$ from the range $[\sigma_{\min}, \sigma_{\max}]$ [S1].
        *   Compute the base sample variance $\Sigma_{\text{sample}}$ associated with the current sampling distribution [S1].
        *   Calculate the injected scale-space variance $\Sigma$ using the formula:
            $$ \Sigma = \Sigma_{\text{sample}} \times (1 + (\sigma_{\text{blur}} - \sigma_{\min})) $$
            where $\Sigma$ is the scale-space variance injected as an extra MLP input, $\Sigma_{\text{sample}}$ is the base sample variance, $\sigma_{\text{blur}}$ is the random Gaussian blur variance, and $\sigma_{\min}$ is the minimum blur variance [S1].
        *   Provide $\Sigma$ as an additional input to the final MLP layer [S1].
    *   Query the hash grids for the 3D sample points to obtain feature vectors [S1].
    *   Pass the concatenated features and $\Sigma$ through the MLP to predict density and color [S1].
    *   Compute the loss, which includes:
        *   Image reconstruction loss [S1].
        *   Distortion loss to encourage compact representations [S1].
        *   Loss on specular/diffuse decomposition [S1].
    *   Update network parameters via gradient descent [S1].

4.  **Rendering:**
    *   Use the trained model to render novel views or extract collision geometry (e.g., by coarsening the mesh and replacing the floor with a flat primitive) [S1].

**Parameters:**
*   $\sigma_{\text{blur}} \in [\sigma_{\min}, \sigma_{\max}]$: Random variance for Gaussian blur augmentation applied to training samples [S1].
*   $N \sim 1000$: Number of keyframes extracted from the captured video [S1].
*   Video resolution $\ge 1080p$: Minimum resolution required for better localization and improved NeRF reconstruction results [S1].

**Edge Cases and Pitfalls:**
*   **Motion Blur:** Rapid camera movement during capture introduces motion blur, which degrades reconstruction quality. Slow, even movement is required [S1].
*   **Lighting Consistency:** Variable lighting (e.g., auto white balance) can confuse the model. Fixed white balance and brightness are necessary [S1].
*   **Resolution:** Low-resolution videos lead to poorer localization and reconstruction. Resolution $\ge 1080p$ is recommended [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_scale_variance(sigma_sample, sigma_min, sigma_blur):
    """
    Compute the scale-space variance Sigma to be injected into the MLP.
    
    Args:
        sigma_sample (float): Base sample variance.
        sigma_min (float): Minimum blur variance.
        sigma_blur (float): Random Gaussian blur variance sampled from [sigma_min, sigma_max].
        
    Returns:
        float: The computed scale-space variance Sigma.
    """
    return sigma_sample * (1.0 + (sigma_blur - sigma_min))

def train_step(ray, hash_grids, mlp, sigma_min, sigma_max, sigma_sample):
    """
    Perform one training step for a single ray.
    
    Args:
        ray (Ray): The ray being processed.
        hash_grids (list): List of hash grids for different resolution levels.
        mlp (MLP): The decoder MLP with Swish activations and Layer Normalization.
        sigma_min (float): Minimum blur variance.
        sigma_max (float): Maximum blur variance.
        sigma_sample (float): Base sample variance for the current ray samples.
        
    Returns:
        float: The computed loss for this ray.
    """
    # 1. Sample points along the ray using the proposal network
    # (Assuming proposal network sampling is handled externally or via a helper)
    points = sample_points_along_ray(ray)
    
    # 2. Scale-Space Augmentation
    # Sample random blur variance from [sigma_min, sigma_max]
    sigma_blur = np.random.uniform(sigma_min, sigma_max)
    
    # Compute the injected scale-space variance
    sigma_injected = compute_scale_variance(sigma_sample, sigma_min, sigma_blur)
    
    # 3. Query Hash Grids
    # Get features for each point from the multi-scale hash grids
    features = []
    for grid in hash_grids:
        # Lookup and interpolate features for points in the grid
        grid_features = query_hash_grid(grid, points)
        features.append(grid_features)
    
    # Concatenate features from all levels
    combined_features = np.concatenate(features, axis=-1)
    
    # 4. MLP Forward Pass
    # Add sigma_injected as an extra input to the final MLP layer
    # (Implementation detail: sigma_injected is typically concatenated or added 
    # to the feature vector before the final layer, or passed as a separate scalar input)
    # Assuming the MLP accepts (features, sigma_injected)
    density, diffuse_color, specular_color = mlp(combined_features, sigma_injected)
    
    # 5. Volume Rendering
    # Render the ray using the predicted density and colors
    # (Standard volume rendering integration)
    rendered_color, alpha = volume_render_ray(points, density, diffuse_color, specular_color)
    
    # 6. Loss Computation
    # Compare rendered_color with the ground truth image patch for this ray
    # Include distortion loss and specular/diffuse decomposition loss
    loss = compute_loss(rendered_color, ground_truth_color, density, diffuse_color, specular_color)
    
    return loss

def query_hash_grid(grid, points):
    """
    Query a single hash grid for features at given 3D points.
    
    Args:
        grid (HashGrid): The hash grid data structure.
        points (np.ndarray): Array of 3D points with shape (N, 3).
        
    Returns:
        np.ndarray: Interpolated feature vectors with shape (N, feature_dim).
    """
    # Hash the 3D coordinates to get indices
    indices = hash_coordinates(points)
    
    # Retrieve raw features from the hash table
    raw_features = grid.lookup(indices)
    
    # Perform trilinear interpolation based on the fractional part of the coordinates
    interpolated_features = trilinear_interpolate(points, raw_features, grid.resolution)
    
    return interpolated_features

def hash_coordinates(points):
    """
    Hash 3D coordinates to integer indices.
    
    Args:
        points (np.ndarray): Array of 3D points with shape (N, 3).
        
    Returns:
        np.ndarray: Array of integer indices with shape (N,).
    """
    # Implementation of the spatial hashing function
    # (Specific hash function details are not provided in the source, 
    # but it maps 3D points to table indices)
    return np.floor(points * grid_scale).astype(int) % table_size

def trilinear_interpolate(points, raw_features, resolution):
    """
    Perform trilinear interpolation of features.
    
    Args:
        points (np.ndarray): Array of 3D points with shape (N, 3).
        raw_features (np.ndarray): Array of raw features with shape (Table_Size, Feature_Dim).
        resolution (int): Resolution of the hash grid level.
        
    Returns:
        np.ndarray: Interpolated feature vectors with shape (N, Feature_Dim).
    """
    # Get fractional coordinates
    frac = points - np.floor(points)
    
    # Get integer coordinates
    int_coords = np.floor(points).astype(int)
    
    # Compute indices for the 8 surrounding voxels
    # (Implementation details for trilinear interpolation are standard)
    # ...
    
    # Return interpolated features
    return interpolated_features
```

## Operating characteristics
*   **Time Complexity:** The multi-scale spatial hash grid approach provides $O(1)$ lookup time for feature retrieval, significantly speeding up rendering compared to dense grids [S1].
*   **Memory Complexity:** The hash grid approach is more memory-efficient than dense grids, allowing for compact representations [S1].
*   **Measured Performance:** Rendering speed is $\sim 6$ms per frame on a V100 GPU [S1].
*   **Quality:** Scale-space augmentation improves reconstruction quality, increasing average PSNR on held-out images from $\sim 31.5$ to $\sim 35.4$ [S1].
*   **Strengths:** Fast rendering, compact geometry, robust reconstruction from short mobile videos, anti-aliased rendering via scale-space augmentation [S1].
*   **Limitations:** Requires careful data capture (slow movement, fixed lighting, high resolution) [S1]. Performance depends on the quality of COLMAP localization [S1].
*   **When to Use:** When fast, high-fidelity reconstruction from short mobile videos is needed, particularly for sim2real transfer tasks [S1].
*   **When Not to Use:** When high-quality, controlled studio scans are available and speed is not a constraint, or when the scene is highly dynamic [S1].

## Background
*   **Origin:** The method is part of the NeRF2Real pipeline, presented in "NeRF2Real: Sim2real Transfer of Vision-guided Bipedal Motion Skills using Neural Radiance Fields" [S1].
*   **History:** It builds upon the Instant-NGP architecture [S1] and adapts it for "in the wild" scenes captured with generic mobile cameras [S1].
*   **Usage:** Used to generate high-fidelity, photorealistic scene renderings and collision geometry for sim2real transfer of vision-guided bipedal locomotion policies [S1].

## Sources
[S1] NeRF2Real: Sim2real Transfer of Vision-guided Bipedal Motion Skills using Neural Radiance Fields, https://doi.org/10.1109/icra48891.2023.10161544, fetched 2026-10-29
