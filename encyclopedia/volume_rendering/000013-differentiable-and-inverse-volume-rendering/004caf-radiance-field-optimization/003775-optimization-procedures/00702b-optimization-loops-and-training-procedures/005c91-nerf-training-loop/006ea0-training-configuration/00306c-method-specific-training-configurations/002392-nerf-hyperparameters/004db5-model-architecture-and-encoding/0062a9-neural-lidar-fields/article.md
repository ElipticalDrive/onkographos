# Neural LiDAR Fields

Also called: LiDAR4D

## Overview
Neural LiDAR Fields (LiDAR4D) reconstructs LiDAR point clouds into an implicit continuous representation by modeling geometric depth, reflection intensity, and ray-drop probability, differing from standard NeRFs which optimize for photometric RGB loss [S1]. It distinguishes itself by combining coarse-resolution multi-planar features with high-resolution hash grid representations to handle large-scale dynamic driving scenarios efficiently [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** LiDAR sensor origin $o$, laser direction $\mathbf{d}$ defined by azimuth $\theta$ and elevation $\phi$, and temporal information for 4D synthesis.
*   **Outputs:** Rendered depth $\hat{\mathcal{D}}(\mathbf{r})$, intensity $\hat{\mathcal{I}}(\mathbf{r})$, and ray-drop probability $\hat{\mathcal{P}}(\mathbf{r})$ for a ray $\mathbf{r}$.

**Data Structures**
The method uses a hybrid feature representation consisting of:
1.  **Planar Features ($\mathbf{f}_{\text{planar}}$):** Coarse-resolution multi-planar features.
2.  **Hash Grid Features ($\mathbf{f}_{\text{hash}}$):** High-resolution hash grid representation.
3.  **Positional Encoding ($\gamma(\mathbf{d})$):** Hierarchical encoding of the view direction.

**Positional Encoding**
The direction $\mathbf{d}$ is encoded using $\gamma(\cdot)$ [S1]:
$$ \gamma(x) = (\sin(2^0 x), \cos(2^0 x), \dots, \sin(2^{L-1} x), \cos(2^{L-1} x)) $$
where $L$ is the maximum frequency [S1].

**Laser Direction**
The direction $\mathbf{d}$ is determined by azimuth $\theta$ and elevation $\phi$ [S1]:
$$ \mathbf{d} = (\cos \theta \cos \phi, \ \sin \theta \sin \phi, \ \cos \phi)^T $$

**Network Architecture**
Separate Multi-Layer Perceptrons (MLPs) predict the outputs using the aggregated features and encoded direction [S1]:
*   **Intensity Prediction:** $i = \mathrm{MLP}_{\text{intensity}}(\mathbf{f}_{\text{planar}}, \mathbf{f}_{\text{hash}}, \gamma(\mathbf{d}))$ [S1].
*   **Ray-Drop Prediction:** $\rho = \mathrm{MLP}_{\text{raydrop}}(\mathbf{f}_{\text{planar}}, \mathbf{f}_{\text{hash}}, \gamma(\mathbf{d}))$ [S1].
*   **Density:** Predicted by querying 3D points along the laser into the neural fields [S1].

**Rendering Equation**
Samples are queried along the ray. Let $\sigma_i$ be density, $\delta_i$ be the distance between samples, and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ be the accumulated transmission [S1]. The opacity is $\alpha_i = 1 - e^{-\sigma_i \delta_i}$ [S1].

1.  **Depth:**
    $$ \hat{\mathcal{D}}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i z_i $$
    where $z_i$ is the depth value of the $i$-th queried point [S1].

2.  **Intensity:**
    $$ \hat{\mathcal{I}}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i i_i $$
    where $i_i$ is the predicted intensity at the $i$-th point [S1].

3.  **Ray-Drop Probability:**
    $$ \hat{\mathcal{P}}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i p_i $$
    where $p_i$ is the predicted ray-drop probability at the $i$-th point [S1].

**Ray-Drop Refinement**
To address noise in point-wise ray-drop predictions, a U-Net with residuals refines the ray-drop mask globally using the full ray-drop probability, depth, and intensity predictions as inputs [S1]. The refinement loss is binary cross-entropy:
$$ \mathcal{L}_{\text{refine}} = \mathrm{BCELoss}(\hat{\mathcal{M}}_{\text{Pred}}, \mathcal{M}_{\text{GT}}) $$
where $\hat{\mathcal{M}}_{\text{Pred}}$ is the refined mask and $\mathcal{M}_{\text{GT}}$ is the ground truth [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_lidar_ray(origin, azimuth_theta, elevation_phi, samples):
    # 1. Compute laser direction
    d = vector([
        cos(azimuth_theta) * cos(elevation_phi),
        sin(azimuth_theta) * sin(elevation_phi),
        cos(elevation_phi)
    ])
    
    # 2. Encode direction
    d_encoded = positional_encoding(d, L)
    
    # 3. Initialize accumulators
    depth_acc = 0.0
    intensity_acc = 0.0
    raydrop_acc = 0.0
    transmission = 1.0
    
    # 4. Iterate over samples along the ray
    for point, delta in samples:
        # Query features from planar and hash grids
        f_planar = query_planar_features(point)
        f_hash = query_hash_features(point)
        
        # Predict density, intensity, and ray-drop probability
        sigma = query_density(point, f_planar, f_hash, d_encoded)
        i_val = mlp_intensity(f_planar, f_hash, d_encoded)
        p_val = mlp_raydrop(f_planar, f_hash, d_encoded)
        
        # Compute opacity
        alpha = 1.0 - exp(-sigma * delta)
        
        # Accumulate contributions
        weight = transmission * alpha
        depth_acc += weight * point.depth
        intensity_acc += weight * i_val
        raydrop_acc += weight * p_val
        
        # Update transmission
        transmission *= (1.0 - alpha)
        
        # Early exit if transmission is negligible
        if transmission < epsilon:
            break
            
    # 5. Refine ray-drop probability using U-Net
    # Inputs: full ray-drop prob, depth, intensity predictions
    refined_mask = u_net_refine(raydrop_acc, depth_acc, intensity_acc)
    
    return depth_acc, intensity_acc, refined_mask
```

## Operating characteristics
*   **Complexity:** Uses hybrid multi-planar and hash grid features for efficient large-scale reconstruction [S1].
*   **Strengths:** Avoids adverse effects of hash collisions and handles dynamic driving scenarios [S1].
*   **Limitations:** Ray-drop prediction via point-wise MLP is inherently noisy without refinement [S1].
*   **Performance:** Benchmarked against LiDARsim, NKSR, and PCGen on depth, intensity, and point cloud metrics (CD, F-score, RMSE, MedAE, LPIPS, SSIM, PSNR) [S1].

## Background
LiDAR4D was introduced to redefine neural fields for LiDAR data, focusing on geometric depth, reflection intensity, and ray-drop probability [S1]. It builds on the concept of neural radiance fields but adapts them for LiDAR sensors which measure time-of-flight and intensity rather than RGB color [S1]. The method incorporates temporal information for novel space-time view synthesis [S1].

## Sources
[S1] LiDAR4D: Dynamic Neural Fields for Novel Space-Time View LiDAR Synthesis, https://doi.org/10.1109/cvpr52733.2024.00492, fetched 2026-10-30
