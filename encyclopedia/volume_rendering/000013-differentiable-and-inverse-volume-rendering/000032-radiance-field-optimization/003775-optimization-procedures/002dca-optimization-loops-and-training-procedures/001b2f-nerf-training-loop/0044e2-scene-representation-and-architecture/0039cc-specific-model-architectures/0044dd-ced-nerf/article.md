# Ced-NeRF


## Overview
Ced-NeRF is a compact and efficient method for modeling dynamic neural radiance fields that utilizes a grid-based hybrid representation and a canonical space to model motion [S1]. It specifically addresses the problem of overfitting artifacts, such as floaters and noisy geometry, which commonly occur in hybrid representations of dynamic scenes due to fast convergence before accurate motion prediction is achieved [S1]. It distinguishes itself from siblings like [AdaNeRF](6195-adanerf.md) and [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md) by employing a high-resolution grid for rapid convergence while avoiding separated fields or models that increase training difficulty and slow rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses and timestamps $t$. For rendering, a camera ray $\mathbf{r} = \mathbf{o} + s\mathbf{d}$ defined by origin $\mathbf{o}$, direction $\mathbf{d}$, and a specific timestamp $t$.
*   **Outputs:** A predicted pixel color $\bar{C}(\mathbf{r})$.

**Data Structures:**
*   **Grid $G$:** A 3D grid of learnable feature vectors. The specific layout is not detailed beyond the interpolation mechanism [S1].
*   **Deformation Network $\Phi_d$:** A neural network mapping encoded coordinates and timestamp to a deformation vector [S1].
*   **Radiance Field Network $\Phi_r$:** A neural network predicting density $\sigma$ and color $\mathbf{c}$ from grid features and direction [S1].

**Position Encoding:**
Ced-NeRF uses two types of encoding [S1]:
1.  **Frequency Encoding $\gamma(\cdot)$:** Applied to spatial coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ to capture high-frequency details and mitigate spectral bias [S1, Parent].
    $$
    \gamma(\mathbf{p}) = (\sin(2^0 \pi \mathbf{p}), \cos(2^0 \pi \mathbf{p}), \dots, \sin(2^{L-1} \pi \mathbf{p}), \cos(2^{L-1} \pi \mathbf{p}))
    $$
    where $L$ is the number of frequency bands [S1, Parent].
2.  **Grid Encoding $\gamma_{grid}(\cdot)$:** Applied to spatial coordinates $\mathbf{x}'$ to retrieve features from the grid $G$ [S1]. This involves determining the closest grid cells and interpolating their features. For example, Instant-NGP selects eight grid cells for linear interpolation [S1].

**Algorithm Steps:**
1.  **Sample Points:** For a camera ray $\mathbf{r} = \mathbf{o} + s\mathbf{d}$, sample $N$ points $\mathbf{r}_i = \mathbf{o} + s_i \mathbf{d}$ along the ray [S1].
2.  **Encode Input:** Encode the spatial coordinate $\mathbf{x}_i$ and timestamp $t$ using frequency encoding $\gamma(\cdot)$ [S1].
3.  **Deform Coordinate:** Pass the encoded $\mathbf{x}_i$ and $t$ into the deformation network $\Phi_d$ to predict a deformation vector. Add this to the original coordinate to get the canonical space coordinate $\mathbf{x}'_i$:
    $$
    \mathbf{x}'_i = \Phi_d(\gamma(\mathbf{x}_i), \gamma(t)) + \mathbf{x}_i
    $$
    [S1].
4.  **Grid Feature Retrieval:** Encode the deformed coordinate $\mathbf{x}'_i$ using grid encoding $\gamma_{grid}(\cdot)$ to retrieve feature vectors from the grid $G$ [S1].
5.  **Encode Direction:** Encode the viewing direction $\mathbf{d}$ using frequency encoding $\gamma(\cdot)$ [S1].
6.  **Predict Radiance:** Pass the grid features (from $\mathbf{x}'_i$) and direction features (from $\mathbf{d}$) into the radiance field network $\Phi_r$ to predict density $\sigma_i$ and color $\mathbf{c}_i$ [S1].
7.  **Volume Rendering:** Compute the final color $\bar{C}(\mathbf{r})$ using volumetric rendering [S1]:
    $$
    \bar{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $\delta_i$ is the distance between samples $s_i$ and $s_{i+1}$, and $T_i$ is the accumulated transmittance:
    $$
    T_i = \prod_{j=1}^{i-1} \exp(-\sigma_j \delta_j)
    $$
    [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ced_nerf_render(ray_origin, ray_direction, timestamp, grid, deformation_net, radiance_net, num_samples):
    """
    Renders a single ray using the Ced-NeRF method.
    
    Args:
        ray_origin: 3D vector (o)
        ray_direction: 3D vector (d)
        timestamp: float (t)
        grid: 3D grid of features
        deformation_net: Neural network Phi_d
        radiance_net: Neural network Phi_r
        num_samples: int N
        
    Returns:
        predicted_color: 3D vector
    """
    # 1. Sample points along the ray
    # Assuming uniform sampling for simplicity; specific s_i distribution not detailed in sources
    # s_i ranges from near_clip to far_clip
    samples = sample_along_ray(ray_origin, ray_direction, num_samples)
    
    predicted_color = 0.0
    accumulated_transmittance = 1.0
    
    for i, s_i in enumerate(samples):
        # 2. Get spatial coordinate x_i
        x_i = ray_origin + s_i * ray_direction
        
        # 3. Encode x_i and t
        enc_x = frequency_encode(x_i)
        enc_t = frequency_encode(timestamp)
        
        # 4. Deform coordinate to canonical space x'_i
        # Phi_d takes encoded inputs and outputs a 3D vector
        deformation = deformation_net(enc_x, enc_t)
        x_prime_i = x_i + deformation
        
        # 5. Retrieve grid features for x'_i
        # Grid encoding involves finding closest cells and interpolating
        grid_features = grid_encode(x_prime_i, grid)
        
        # 6. Encode viewing direction d
        enc_d = frequency_encode(ray_direction)
        
        # 7. Predict density and color
        sigma_i, color_i = radiance_net(grid_features, enc_d)
        
        # 8. Compute alpha and contribute to color
        # delta_i is distance to next sample. For last sample, assume delta=0 or handle boundary.
        # Note: Source defines delta_i as distance between s_i and s_{i+1}.
        # We need delta_i for the current sample's alpha calculation.
        if i < len(samples) - 1:
            delta_i = samples[i+1] - samples[i]
        else:
            delta_i = 0.0 # Or a small epsilon to avoid division by zero if needed, though alpha becomes 0
            
        alpha_i = 1.0 - exp(-sigma_i * delta_i)
        
        # Contribution: T_i * alpha_i * c_i
        # T_i is accumulated_transmittance before this sample
        predicted_color += accumulated_transmittance * alpha_i * color_i
        
        # Update transmittance for next sample
        accumulated_transmittance *= (1.0 - alpha_i)
        
        # Early termination optimization (common in NeRF, not explicitly in Ced-NeRF text but standard)
        if accumulated_transmittance < 1e-5:
            break
            
    return predicted_color

def frequency_encode(p, num_bands=10):
    """
    Standard positional encoding for NeRF.
    """
    encoded = [p]
    for i in range(num_bands):
        freq = 2**i * pi
        encoded.append(sin(freq * p))
        encoded.append(cos(freq * p))
    return encoded

def grid_encode(x_prime, grid):
    """
    Retrieves features from grid G at x_prime via interpolation.
    Implementation detail: Find 8 closest grid cells and interpolate.
    """
    # Not fully specified in source, but implies standard tri-linear interpolation
    # on the 8 vertices of the voxel containing x_prime.
    return interpolate_grid_features(x_prime, grid)
```

**Parameters:**
*   **Grid Cells:** The grid resolution is not explicitly given, but the method relies on a "high-resolution grid" for fast convergence [S1].
*   **Interpolation:** Uses linear interpolation from the 8 closest grid cells to the coordinate [S1].

**Edge Cases and Pitfalls:**
*   **Overfitting:** Hybrid representations for dynamic scenes are prone to overfitting due to fast convergence before accurate motion prediction, leading to artifacts like floaters and noisy geometry on novel views [S1]. Ced-NeRF aims to mitigate this through its compact design.
*   **Deformation Accuracy:** The quality of the canonical space mapping $\mathbf{x}'$ is critical. Errors in $\Phi_d$ lead to incorrect grid feature retrieval.

## Operating characteristics
*   **Time Complexity:** The method achieves fast rendering speeds and rapid convergence (e.g., training in 4 minutes on HyperNeRF scenes) compared to other dynamic NeRF methods [S1]. This is attributed to the use of a high-resolution grid which allows for fast convergence, unlike low-resolution grids or sparse voxel grids [S1].
*   **Memory Complexity:** It is described as "compact," requiring only a small number of additional parameters to construct the hybrid representation [S1].
*   **Strengths:** Fast training and rendering speeds; high-quality rendering results; outperforms state-of-the-art methods in quality, training, and rendering speed on dynamic scene datasets [S1].
*   **Limitations:** The source notes that separated fields or models (which Ced-NeRF avoids) increase training difficulty and slow rendering, implying that Ced-NeRF's unified approach is a strength, but the specific limitations of the grid-based approach (e.g., memory usage for very high-resolution grids) are not quantified [S1].
*   **When to use:** Suitable for dynamic scene reconstruction where fast training and rendering are prioritized, and where artifacts from overfitting in hybrid representations are a concern [S1].
*   **When not to use:** Not specified, but generally, high-resolution grids may consume significant memory for very large scenes.

## Background
Ced-NeRF was proposed to address the issue of overfitting in hybrid representations for dynamic scenes [S1]. It builds upon the concept of using a deformation network $\Phi_d$ to map spatial coordinates and timestamps to a canonical space, a technique previously used in methods like DNeRF [S1]. It incorporates explicit-based methods, specifically voxel grids, to accelerate training and rendering, building on the Instant-NGP approach for grid encoding [S1]. The method was presented at the AAAI Conference on Artificial Intelligence in 2024 [S1].

## Sources
[S1] Ced-NeRF: A Compact and Efficient Method for Dynamic Neural Radiance Fields, https://doi.org/10.1609/aaai.v38i4.28138, fetched 2026-10-26
