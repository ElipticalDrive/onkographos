# Radar Fields

Also called: SAR Radiance Fields

## Overview
Radar Fields is a differentiable rendering framework that extends radiance fields to Synthetic Aperture Radar (SAR) imagery, enabling the learning of volumetric surface models from multiple SAR measurements [S1]. Unlike optical radiance fields that integrate radiance along individual rays, Radar Fields integrates radar signal contributions across all rays within a given azimuth plane that share the same range distance [S1]. This method solves the problem of 3D reconstruction from SAR data by adapting the volume rendering equation to the specific physics of SAR image formation, where the integration dimension is swapped from the ray direction to the azimuth direction [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A collection of SAR images (complex-valued or intensity) of a scene from different viewpoints; camera poses (satellite positions and orientations).
*   **Outputs:** A learned surface model of the scene, typically represented as a Digital Surface Model (DSM) or implicit function, and optionally surface properties like specularity.

**Data Structures:**
*   **Surface Model:** The surface is defined by a Digital Surface Model (DSM) function $dsm: [0, 1]^2 \rightarrow [0, 1]$, mapping 2D coordinates to height [S1]. The implicit function is defined as $F(x) = z - dsm(x, y)$ for $x=(x,y,z) \in [0,1]^3$ [S1].
*   **Implementation:** The DSM is rasterized into an image where each pixel represents the height value. These pixel values are directly optimized during training [S1]. Subpixel heights are estimated using bilinear interpolation of the nearest values, and surface normals are computed accordingly [S1].
*   **Ray Data:** For each azimuth plane, a set of rays $r_j$ is defined by origin $o_j$ and direction $v_j$ [S1]. Distance samples $d_i$ are fixed by the SAR range sampling [S1].

**Parameters:**
*   **Specularity Parameter ($\theta$):** Used in the reflection model $s = -\langle v|n \rangle^\theta$ [S1]. $\theta=1$ corresponds to a Lambertian surface; larger values represent specular surfaces. This parameter can be learned alongside the surface location [S1].
*   **Training Hyperparameters:** 10,000 optimization steps; initial learning rate 1.0, reduced to 0.1 at step 5,000, and to $10^{-2}$ at step 8,000 [S1]. Azimuth planes are sampled uniformly and batched in groups of 64, with 256 rays sampled per azimuth plane [S1].

**Algorithm Steps:**
1.  **Ray Sampling:** For a given azimuth plane, sample a set of rays $r_j$ with origins $o_j$ and directions $v_j$. The distance sampling $d_i$ is fixed by the SAR data's range resolution [S1].
2.  **Ray Origin Perturbation:** To improve optimization performance (analogous to noise injection in NeRF), perturb the ray origin. Let $w_j$ be the vector orthogonal to $v_j$ inside the azimuth plane. The perturbed origin is $\tilde{o}_j = o_j + n w_j$, where $n \sim \mathcal{N}(0, 1)$ [S1].
3.  **Signal Computation:** For each ray $r_j$ and distance sample $d_i$, compute the sensed signal $s_{r_j}(d_i)$. This involves evaluating the density $\sigma_{i,j}$ and normal $n_{i,j}$ at the intersection point, and applying the reflection model [S1]:
    $$ s_{r_j}(d_i) = -T_{i,j} \sigma_{i,j} \langle v_j | n_{i,j} \rangle^\theta $$
    where $T_{i,j}$ is the transmittance along ray $j$ up to sample $i$ [S1].
4.  **Azimuth Integration:** Batch all rays belonging to the same azimuth plane. Compute the total signal for a pixel at distance $d_i$ by summing the contributions from all rays $j$ in that plane [S1]:
    $$ s(d_i) = \sum_j s_{r_j}(d_i) = \sum_j -T_{i,j} \sigma_{i,j} \langle v_j | n_{i,j} \rangle^\theta $$
    Note that the index $j$ (ray) is summed over, while $i$ (distance) defines the pixel [S1].
5.  **Loss Computation:** Compare the rendered signal $s(d_i)$ with the observed SAR intensity $I$. For single-channel single-look intensity images, model the speckle noise as multiplicative: $I = n \times R$, where $R$ is the noise-free reflectance and $n \sim \Gamma(1, 1)$ [S1]. Minimize the difference between the rendered signal and the observed intensity, potentially using a Noise2Noise approach or explicit noise modeling [S1].
6.  **Optimization:** Back-propagate the loss to update the DSM pixel values and the specularity parameter $\theta$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class RadarField:
    def __init__(self, dsm_image, camera_poses, azimuth_planes):
        """
        Initialize Radar Field.
        :param dsm_image: Tensor of shape (H, W) representing the rasterized DSM.
        :param camera_poses: List of camera poses for each SAR acquisition.
        :param azimuth_planes: List of azimuth plane definitions.
        """
        self.dsm = nn.Parameter(dsm_image.clone().float())
        self.theta = nn.Parameter(torch.tensor(1.0)) # Specularity parameter
        self.camera_poses = camera_poses
        self.azimuth_planes = azimuth_planes

    def get_surface_normal(self, x, y):
        """
        Compute surface normal from DSM using finite differences.
        """
        # Bilinear interpolation to get height at subpixel coordinates
        h = self.dsm[y, x]
        # Compute gradients for normal
        # [G] Standard finite difference or Sobel filter implementation
        dz_dx, dz_dy = compute_gradient(self.dsm, y, x)
        normal = torch.stack([-dz_dx, -dz_dy, torch.ones_like(h)], dim=-1)
        return torch.nn.functional.normalize(normal, dim=-1)

    def render_azimuth_plane(self, azimuth_plane, rays, distance_samples):
        """
        Render signal for a single azimuth plane.
        :param azimuth_plane: Index of the azimuth plane.
        :param rays: List of ray origins and directions for the plane.
        :param distance_samples: Fixed distance samples from SAR data.
        :return: Rendered signal tensor.
        """
        rendered_signal = torch.zeros_like(distance_samples)
        
        # Batch all rays in the azimuth plane
        for j, (origin, direction) in enumerate(rays):
            # Perturb ray origin
            w = get_orthogonal_vector(direction, azimuth_plane)
            n_noise = torch.randn(1)
            perturbed_origin = origin + n_noise * w
            
            # Compute transmittance and signal for each distance sample
            # [G] Standard volume rendering integration along the ray
            # However, signal is accumulated per distance sample across rays
            for i, d_i in enumerate(distance_samples):
                # Find intersection or sample point at distance d_i
                point = perturbed_origin + d_i * direction
                x, y = project_to_dsm(point)
                
                # Get density and normal
                # [G] Density is derived from DSM gradient or implicit function
                sigma = compute_density(self.dsm, x, y)
                normal = get_surface_normal(x, y)
                
                # Compute transmittance T_{i,j}
                # [G] Accumulate opacity along the ray up to i
                T = compute_transmittance(perturbed_origin, direction, d_i, sigma)
                
                # Compute reflection signal
                # s_{r_j}(d_i) = -T_{i,j} * sigma * <v_j | n_{i,j}>^theta
                cos_angle = torch.dot(direction, normal)
                signal = -T * sigma * torch.pow(cos_angle, self.theta)
                
                # Accumulate across rays for the same distance sample
                rendered_signal[i] += signal
                
        return rendered_signal

    def forward(self, azimuth_plane_idx):
        """
        Forward pass for optimization.
        """
        azimuth_plane = self.azimuth_planes[azimuth_plane_idx]
        rays = sample_rays(azimuth_plane, num_rays=256)
        distance_samples = get_sar_range_samples(azimuth_plane)
        
        rendered = self.render_azimuth_plane(azimuth_plane, rays, distance_samples)
        observed = get_observed_sar_intensity(azimuth_plane)
        
        # Loss computation with speckle noise model
        # [G] Implement negative log-likelihood for Gamma distribution or MSE
        loss = compute_loss(rendered, observed)
        return loss

def compute_loss(rendered, observed):
    """
    Compute loss based on multiplicative speckle noise model.
    I = n * R, n ~ Gamma(1, 1)
    """
    # [G] Simple MSE as a proxy for the source's description of handling noisy data
    # More rigorous implementation would use Gamma NLL
    return torch.mean((rendered - observed) ** 2)
```

**Edge Cases and Pitfalls:**
*   **Fixed Sampling:** Unlike NeRF, fixed uniform sampling along rays is not possible because the sampling step is defined by the SAR range sampling [S1]. Noise must be added to the ray origin instead [S1].
*   **Multi-bounce Effects:** Signals from double or multiple bounces appear at incorrect positions (e.g., signal from A->B->receiver appears at C) [S1]. The generic case violates the zero-doppler assumption, requiring batching multiple azimuth planes, which is highly unfeasible without strong priors [S1].
*   **Complex Data:** SAR raw data stores complex numbers separating amplitude and phase. The method primarily focuses on intensity images, but phase contains satellite-target path delay modulo the wavelength [S1].

## Operating characteristics
*   **Time Complexity:** The computational complexity remains similar to standard radiance fields on average, with the primary difference being the batching requirement [S1]. All rays of a given azimuth plane must be batched together during computation [S1].
*   **Memory Complexity:** Not explicitly given, but batching all rays in an azimuth plane may increase memory usage compared to per-ray processing in optical NeRF [S1].
*   **Performance:** Reconstruction errors are in the order of $10^{-2}$ pixels relative to ground pixel resolution [S1].
*   **Strengths:** Enables 3D reconstruction from SAR data, which has day-night and cloud-penetrating capabilities [S1]. Does not prescribe geometric configuration, allowing opportunistic acquisitions [S1].
*   **Limitations:** Multi-bounce effects are not fully handled in the generic case [S1]. Requires batching rays by azimuth plane, which may limit scalability [S1].

## Background
Radar Fields was introduced in the paper "Radar Fields: An Extension of Radiance Fields to SAR" [S1]. The method extends the concept of radiance fields to Synthetic Aperture Radar (SAR) imagery. SAR systems use side-looking geometry and emit electromagnetic pulses, receiving echoes backscattered by the ground surface. The resulting images are complex-valued, with amplitude related to ground reflectivity and angle, and phase containing path delay information [S1]. The method was developed to leverage the widespread availability of SAR data for global 3D reconstruction [S1].

## Sources
[S1] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
