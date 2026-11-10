# Radar Fields

Also called: SAR Radiance Fields

## Overview
Radar Fields is a differentiable rendering framework that extends the radiance field concept to Synthetic Aperture Radar (SAR) imagery, enabling the learning of volumetric surface models from multiple SAR measurements [S1]. Unlike optical radiance fields that integrate emission along individual rays, Radar Fields integrates radar signal contributions across all rays within a given azimuth plane that share the same range distance [S1]. This architectural difference allows for the reconstruction of 3D geometry from SAR data, which offers day-night and cloud-penetrating imaging capabilities [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A collection of SAR images (complex-valued or intensity) acquired from different azimuth positions; camera poses (satellite positions and viewing directions); SAR range sampling parameters [S1].
*   **Outputs:** A Digital Surface Model (DSM) representing the scene geometry, and optionally a specularity parameter map $\theta$ [S1].

**Data Structures:**
*   **Surface Model:** The scene surface is defined by a Digital Surface Model function $dsm: [0, 1]^2 \rightarrow [0, 1]$, where the output represents normalized height [S1].
*   **Implicit Function:** The surface is represented implicitly by $F(x) = z - dsm(x, y)$ for $x=(x,y,z) \in [0, 1]^3$ [S1].
*   **Implementation:** The DSM is rasterized into an image where each pixel stores the height value. These pixel values are directly optimized during training [S1]. Normals are computed via bilinear interpolation of the nearest height values [S1].

**Numbered Steps:**

1.  **Ray and Azimuth Plane Definition:**
    For a given azimuth plane (spanning the satellite flight direction and range direction), define a set of rays $\{r_j\}$ with origins $o_j$ and directions $v_j$ [S1]. The distance sampling $\{d_i\}$ is fixed and defined by the SAR range sampling resolution [S1].

2.  **Ray Origin Perturbation:**
    To improve optimization performance (analogous to noise injection in NeRF), perturb the ray origin [S1]. Since fixed uniform sampling along rays is not possible due to fixed SAR range sampling, noise is added to the origin [S1].
    $$
    \tilde{o}_j = o_j + n w_j
    $$
    where $w_j$ is a vector orthogonal to $v_j$ within the azimuth plane, and $n \sim \mathcal{N}(0, 1)$ [S1].

3.  **Signal Computation per Ray:**
    For each ray $r_j$ and distance sample $d_i$, compute the sensed signal contribution $s_{r_j}(d_i)$ [S1]. This involves evaluating the surface implicit function $F$ to determine intersection and normal $n_{i,j}$ [S1]. The signal is modeled using a specular reflection model:
    $$
    s_{r_j}(d_i) = -T_{i,j} \sigma_{i,j} \langle v_j | n_{i,j} \rangle^\theta
    $$
    where:
    *   $T_{i,j}$ is the transmittance [S1].
    *   $\sigma_{i,j}$ is the density/scattering coefficient [S1].
    *   $\langle v_j | n_{i,j} \rangle$ is the dot product of the incidence direction and surface normal [S1].
    *   $\theta$ is the specularity parameter ($\theta=1$ is Lambertian; $\theta > 1$ is specular) [S1].

4.  **Cross-Ray Integration (Radar Rendering):**
    Unlike optical rendering which sums along a ray, Radar Fields sums contributions from all rays $j$ in the azimuth plane that correspond to the same range distance $d_i$ [S1]. The total signal $s(d_i)$ for a pixel at distance $d_i$ is:
    $$
    s(d_i) = \sum_j s_{r_j}(d_i) = \sum_j -T_{i,j} \sigma_{i,j} \langle v_j | n_{i,j} \rangle^\theta
    $$
    Note that the index $i$ (distance) is the primary key for the pixel, while $j$ (ray) is summed over [S1].

5.  **Noise Modeling:**
    For single-channel single-look intensity images, the noise-free reflectance $R$ is related to the observed intensity $I$ by multiplicative speckle noise:
    $$
    I = n \times R
    $$
    where $n \sim \Gamma(1, 1)$ [S1]. This $n$ is distinct from the perturbation noise in Step 2.

6.  **Optimization:**
    Minimize the difference between the rendered signal $s(d_i)$ and the observed SAR intensity $I$ [S1]. The DSM height values and specularity parameter $\theta$ are optimized [S1]. Surface regularization can be applied using terms like:
    $$
    L_{d} = \frac{1}{|\mathcal{C}|} \sum_{C \in \mathcal{C}} \exp \left( -\frac{L}{d} \text{Tr}(\Sigma^{-1} C) \right)
    $$
    with $\Gamma_d(L) = \pi^{d(d-1)/2} \prod_{k=1}^{d} \Gamma(L - k + 1)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class RadarField:
    def __init__(self, dsm_grid, azimuth_planes, rays_per_plane=256):
        """
        Initialize Radar Field.
        
        Args:
            dsm_grid: 2D array of shape (H, W) representing the rasterized DSM.
                      Values are normalized heights in [0, 1].
            azimuth_planes: List of azimuth plane configurations.
                           Each plane contains ray origins, directions, and range samples.
            rays_per_plane: Number of rays to sample per azimuth plane.
        """
        self.dsm = dsm_grid.copy()
        self.dsm.requires_grad = True
        self.theta = np.ones_like(dsm_grid) * 1.0
        self.rays_per_plane = rays_per_plane
        
    def get_normal_and_height(self, x, y):
        """
        Compute height and normal from rasterized DSM using bilinear interpolation.
        
        Args:
            x, y: Coordinates in [0, 1] range.
            
        Returns:
            z: Height value.
            n: Surface normal vector.
        """
        # Bilinear interpolation to get height
        z = self._bilinear_interp(self.dsm, x, y)
        
        # Compute normal via finite differences or gradient of DSM
        dz_dx, dz_dy = self._compute_gradients(self.dsm, x, y)
        
        # Normal vector n = (-dz/dx, -dz/dy, 1) normalized
        n = np.stack([-dz_dx, -dz_dy, np.ones_like(z)], axis=-1)
        n = n / np.linalg.norm(n, axis=-1, keepdims=True)
        
        return z, n

    def _bilinear_interp(self, grid, x, y):
        # Standard bilinear interpolation implementation
        # Returns interpolated height values
        pass

    def _compute_gradients(self, grid, x, y):
        # Standard gradient computation implementation
        # Returns dz/dx and dz/dy
        pass

    def render_azimuth_plane(self, azimuth_plane_config):
        """
        Render a single azimuth plane following the cross-ray integration formula.
        
        Args:
            azimuth_plane_config: Dictionary containing:
                - origins: Array (N, 3) ray origins o_j
                - directions: Array (N, 3) ray directions v_j
                - distances: Array (M,) fixed range samples d_i
                
        Returns:
            rendered_signal: Array (M,) signal for each range bin.
        """
        origins = azimuth_plane_config['origins']
        directions = azimuth_plane_config['directions']
        distances = azimuth_plane_config['distances']
        
        N = origins.shape[0]
        M = distances.shape[0]
        
        # 1. Perturb ray origins
        # w_j is orthogonal to v_j in the azimuth plane
        noise = np.random.randn(N, 1)
        w = azimuth_plane_config['orthogonal_vectors'] # (N, 3)
        perturbed_origins = origins + noise * w
        
        # Initialize signal accumulator for each distance bin
        total_signal = np.zeros(M)
        
        # 2. Cross-Ray Integration: s(d_i) = sum_j s_{r_j}(d_i)
        for i in range(M):
            d_i = distances[i]
            ray_contribution_sum = 0.0
            
            for j in range(N):
                o_j = perturbed_origins[j]
                v_j = directions[j]
                
                # 3. Compute intersection and signal for ray j at distance d_i
                # Intersection logic not provided in sources [G].
                # Typically involves solving F(x) = 0 along the ray or checking
                # if the ray intersects the DSM at the specified range bin.
                # For this pseudocode, we assume a helper returns sigma, normal, transmittance.
                sigma, normal, transmittance = self._compute_ray_intersection(
                    o_j, v_j, d_i, self.dsm
                )
                
                if sigma is not None:
                    dot_product = np.dot(v_j, normal)
                    # Specular model: s = - <v|n> ^ theta
                    signal_j = -transmittance * sigma * (dot_product ** self.theta)
                    ray_contribution_sum += signal_j
                    
            total_signal[i] = ray_contribution_sum
            
        return total_signal

    def _compute_ray_intersection(self, origin, direction, distance, dsm):
        # Computes intersection point, normal, sigma, and transmittance.
        # Returns None if no valid intersection is found.
        # Implementation details omitted as they are not provided in sources [G].
        pass

    def train_step(self, target_signal, azimuth_plane_config, learning_rate):
        """
        Perform one optimization step.
        """
        rendered_signal = self.render_azimuth_plane(azimuth_plane_config)
        
        # Loss: MSE between rendered and target (accounting for speckle noise)
        loss = np.mean((rendered_signal - target_signal) ** 2)
        
        # Backpropagation and parameter update would occur here
        # self.dsm -= learning_rate * grad(loss, self.dsm)
        # self.theta -= learning_rate * grad(loss, self.theta)
        
        return loss
```

**Parameters:**
*   **Specularity $\theta$:** Initial value 1.0 (Lambertian). Can be learned [S1].
*   **Learning Rate:** Initial 1.0, reduced to 0.1 at step 5,000, and to $10^{-2}$ at step 8,000 [S1].
*   **Training Steps:** 10,000 steps [S1].
*   **Batching:** Azimuth planes are sampled uniformly and grouped into batches of size 64 [S1].
*   **Rays per Plane:** 256 rays sampled per azimuth plane [S1].

**Edge Cases and Pitfalls:**
*   **Multi-bounce Effects:** Signals from double or multiple bounces (e.g., A->B->receiver) appear at incorrect positions (e.g., at C) [S1]. This violates the zero-doppler assumption, requiring batching multiple azimuth planes, which is highly unfeasible without strong priors [S1].
*   **Fixed Sampling:** Fixed uniform sampling along rays is not possible because the sampling step is defined by the SAR range sampling [S1]. Noise must be added to the ray origin instead [S1].
*   **Complex Data:** SAR raw data stores complex numbers [S1]. The method typically uses intensity images, losing phase information which contains path delay data [S1].
*   **Batching Requirement:** Requires significant batching for azimuth planes because computing a single pixel requires computing all rays in that azimuth plane [S1].

## Operating characteristics
*   **Time Complexity:** On average, the computations necessary to learn a radar field remain the same as those for a radiance field, with the primary difference being the batching strategy [S1].
*   **Memory Complexity:** Requires batching all rays of a given azimuth plane together during computation, as computing a single pixel requires computing all rays in that azimuth plane [S1].
*   **Performance:** Reconstruction errors are in the order of $10^{-2}$ pixels relative to ground pixel resolution [S1].
*   **Strengths:** Enables 3D reconstruction from SAR data, leveraging its day-night and cloud-penetrating capabilities. Does not prescribe geometric configuration of acquisitions, allowing opportunistic use [S1].
*   **Limitations:** Struggles with multi-bounce effects. Requires significant batching for azimuth planes. Assumes non-absorbing medium [S1].

## Background
Radar Fields was introduced to extend radiance fields to Synthetic Aperture Radar (SAR) data [S1]. SAR systems emit electromagnetic pulses and receive backscattered echoes [S1]. Unlike optical sensors, SAR wavelengths experience minimal interaction with atmospheric gases, allowing for imaging in all weather conditions and at night [S1]. The method adapts the differentiable rendering framework to account for the unique SAR image formation model, where integration occurs across rays in an azimuth plane rather than along individual rays [S1].

## Sources
[S1] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
