# Neural Surface Reconstruction and Rendering for LiDAR-Visual Systems

Also called: M2Mapping

## Overview
M2Mapping is a method that integrates LiDAR-derived occupancy maps with neural radiance fields (NeRF) and signed distance fields (SDF) to reconstruct geometry and render appearance for LiDAR-visual systems [S1]. It distinguishes itself from standard NeRF approaches by using visible-aware occupancy classification to handle unknown spaces and structure-aware sampling to adaptively capture sharp geometric details, particularly for slim objects [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** LiDAR point clouds (used to build the occupancy map), camera images, camera intrinsics, and extrinsics [S1].
*   **Outputs:** Reconstructed 3D mesh (geometry) and rendered RGB images (appearance) [S1].

**Data Structures**
*   **Occupancy Map:** A 3D grid representing the environment with states: `free`, `occupied`, and `unknown` [S1]. The `unknown` state is further classified into `visible unknown` and `invisible unknown` based on visibility from camera viewpoints [S1].
*   **Neural Network:** A Multi-Layer Perceptron (MLP) architecture [S1].
    *   **SDF Network ($f_S$):** Takes 3D position $\mathbf{x}$ as input and outputs a signed distance value $\hat{s}$ and a scale parameter $\hat{\beta}$ [S1].
    *   **Color Network ($f_C$):** Takes encoded position $\mathbf{x}$ and view direction $\mathbf{d}$ as input and outputs view-dependent color $\hat{\mathbf{c}}$ [S1].

**Encoding**
The method uses two distinct encoding strategies for the input coordinates [S1]:
1.  **Hash Encoding:** Applied to the 3D position $\mathbf{x}$ for multi-resolution feature encoding [S1].
2.  **Spherical Harmonics Encoding:** Applied to the viewing direction $\mathbf{d}$ for directional encoding [S1].
These encodings are concatenated before being fed into the respective MLPs [S1].

**Numbered Steps**
1.  Classify `unknown` grid states in the occupancy map into `visible unknown` and `invisible unknown` via pixel ray casting from camera images [S1]. A grid is `visible unknown` if it is visible from the camera without any `occupied` grid in between [S1].
2.  Sample points along a pixel ray $r(t) = \mathbf{o} + t\mathbf{d}$ from the camera origin $\mathbf{o}$ in direction $\mathbf{d}$ using structure-aware sampling [S1].
3.  Encode sample positions and view directions using hash and spherical harmonics encodings, respectively [S1].
4.  Query the SDF network to obtain $\hat{s}_k$ and $\hat{\beta}_k$ at each sample point $\mathbf{x}_k$ [S1].
5.  Compute volume density $\hat{\sigma}_k$ from the SDF output and ray direction [S1].
6.  Query the color network with concatenated encoded features to obtain $\hat{\mathbf{c}}_k$ [S1].
7.  Render the pixel color $\hat{\mathbf{C}}$ by volume rendering blending the samples' colors and densities [S1].

**Formulas**
The volume rendering equation blending colors and densities is [S1]:
$$ \hat{\mathbf{C}} = \sum_{k} \left( \prod_{j<k} \exp(-\hat{\sigma}_j \delta_j) \right) (1 - \exp(-\hat{\sigma}_k \delta_k)) \hat{\mathbf{c}}_k $$
where $\delta_k = t_{k+1} - t_k$ is the distance between adjacent samples, and $\hat{\sigma}_k$ is the density [S1]. This abbreviates to $\hat{\mathbf{C}} = \sum_{k} T_k \alpha_k \hat{\mathbf{c}}_k$, where $\alpha_k = 1 - \exp(-\hat{\sigma}_k \delta_k)$ is the opacity and $T_k = \prod_{j<k} \exp(-\hat{\sigma}_j \delta_j)$ is the transmittance [S1].

Density is derived from the SDF output using the gradient of the signed distance value along the ray direction (slope) [S1]:
$$ \hat{\sigma}_k = \max \left( - \frac{\Phi(-\hat{s}_k, \hat{\beta}_k) \frac{\partial \hat{s}_k}{\partial \mathbf{x}_k} \cdot \mathbf{d}}{\hat{\beta}_k}, 0 \right) $$
where $\frac{\partial \hat{s}_k}{\partial \mathbf{x}_k}$ is the gradient of the SDF at $\mathbf{x}_k$, $\mathbf{d}$ is the ray direction, and $\Phi$ is the derivative of the cumulative distribution function used for SDF rendering (Gaussian PDF) [G].

**Parameters**
*   Number of samples per ray: Not given in the sources.
*   Hash encoding levels and grid resolutions: Not given in the sources.
*   Spherical harmonics degree: Not given in the sources.

**Edge Cases and Pitfalls**
*   **Precision:** The density computation involves exponentials and gradients; numerical stability requires clamping $\hat{\beta}_k$ away from zero and handling negative slopes correctly [G].
*   **Boundaries:** The last sample's delta must be large enough to ensure full ray coverage, otherwise transmittance will not decay to zero [G].
*   **Degenerate Input:** If the ray direction $\mathbf{d}$ is zero or unnormalized, the slope calculation fails; rays must be normalized before rendering [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_pdf(x):
    """Standard Gaussian probability density function [G]."""
    return np.exp(-0.5 * x**2) / np.sqrt(2 * np.pi)

def compute_density_from_sdf(sdf_value, sdf_scale, sdf_gradient, ray_direction):
    """Computes volume density from SDF output using the source formula [S1]."""
    slope = np.dot(sdf_gradient, ray_direction)
    # Phi is the derivative of the Gaussian CDF, i.e., the Gaussian PDF [G]
    phi_factor = gaussian_pdf(-sdf_value / sdf_scale) / sdf_scale
    density = max(- (phi_factor * slope) / sdf_scale, 0.0)
    return density

def compute_sdf_gradients(sdf_network, points):
    """Computes SDF gradients via finite differences for self-containment [G]."""
    eps = 1e-4
    grad = np.zeros_like(points)
    for i in range(3):
        p_plus = points.copy()
        p_plus[:, i] += eps
        p_minus = points.copy()
        p_minus[:, i] -= eps
        grad[:, i] = (sdf_network(p_plus)[:, 0] - sdf_network(p_minus)[:, 0]) / (2 * eps)
    return grad

def render_pixel(camera_origin, camera_direction, occupancy_map, sdf_network, color_network, hash_encoder, sh_encoder, num_samples, near, far):
    """Renders a single pixel using M2Mapping [S1]."""
    # 1. Structure-aware sampling (linear spacing as placeholder for occupancy-guided strategy)
    sample_distances = np.linspace(near, far, num_samples)
    
    # 2. Get sample points
    sample_points = camera_origin + sample_distances[:, np.newaxis] * camera_direction
    
    # 3. Encode inputs
    pos_features = hash_encoder(sample_points)
    dir_features = sh_encoder(camera_direction)
    
    # 4. Predict SDF and Color
    sdf_outputs = sdf_network(pos_features)
    sdf_values = sdf_outputs[:, 0]
    sdf_scales = sdf_outputs[:, 1]
    sdf_gradients = compute_sdf_gradients(sdf_network, pos_features, sample_points)
    
    color_input = np.concatenate([pos_features, dir_features], axis=-1)
    colors = color_network(color_input)
    
    # 5. Compute densities
    densities = np.zeros(num_samples)
    for k in range(num_samples):
        densities[k] = compute_density_from_sdf(sdf_values[k], sdf_scales[k], sdf_gradients[k], camera_direction)
        
    # 6. Volume Rendering
    deltas = np.diff(sample_distances)
    deltas = np.append(deltas, 1e10)  # Ensure last delta covers remaining ray [G]
    alphas = 1.0 - np.exp(-densities * deltas)
    
    transmittances = np.ones(num_samples + 1)
    for k in range(num_samples):
        transmittances[k+1] = transmittances[k] * np.exp(-densities[k] * deltas[k])
    transmittances = transmittances[:-1]
    
    weights = alphas * transmittances
    rendered_color = np.sum(weights[:, np.newaxis] * colors, axis=0)
    return rendered_color
```

## Operating characteristics
*   **Strengths:** Captures more precise geometric details in slim objects compared to standard NeRF [S1]. Retains both structure and texture in extrapolation rendering [S1].
*   **Limitations:** Relies on the quality of the LiDAR-derived occupancy map [S1]. The specific implementation details of "structure-aware sampling" and the exact SDF-to-density conversion function $\Phi$ are not fully detailed in the provided notes, requiring reference to the original paper [S1].
*   **Performance:** Not given in the sources.

## Background
M2Mapping was introduced in the paper "Neural Surface Reconstruction and Rendering for LiDAR-Visual Systems" [S1]. It is designed for generalized LiDAR-visual systems, leveraging the strong geometric priors from LiDAR point clouds (occupied, free, unknown spaces) to improve NeRF training [S1]. The method was developed by researchers associated with HKU-MARS [S1].

## Sources
[S1] Neural Surface Reconstruction and Rendering for LiDAR-Visual Systems, https://arxiv.org/abs/2409.05310v1, fetched 2026-10-26
