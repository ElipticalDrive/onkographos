# Radar Fields

Also called: SAR Radiance Fields

## Overview
Radar Fields is a differentiable rendering framework that extends the radiance field concept to Synthetic Aperture Radar (SAR) imagery, enabling the learning of volumetric surface models from multiple SAR measurements [S1]. Unlike optical radiance fields that integrate emission along individual rays, Radar Fields integrates radar signal contributions across all rays within a given azimuth plane that share the same range distance [S1]. This architectural difference allows for the reconstruction of 3D geometry from SAR data, which offers day-night and cloud-penetrating imaging capabilities [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A collection of SAR images (complex-valued or intensity) acquired from different azimuth positions; camera poses (satellite positions and viewing directions); SAR range sampling parameters.
*   **Outputs:** A Digital Surface Model (DSM) representing the scene geometry, and optionally a specularity parameter map $\theta$ [S1].

**Data Structures:**
*   **Surface Model:** The scene surface is defined by a Digital Surface Model function $dsm: [0, 1]^2 \rightarrow [0, 1]$, where the output represents normalized height [S1].
*   **Implicit Function:** The surface is represented implicitly by $F(x) = z - dsm(x, y)$ for $x=(x,y,z) \in [0, 1]^3$ [S1].
*   **Implementation:** The DSM is rasterized into an image where each pixel stores the height value. These pixel values are directly optimized during training [S1]. Normals are computed via bilinear interpolation of the nearest height values [S1].

**Numbered Steps:**

1.  **Ray and Azimuth Plane Definition:**
    For a given azimuth plane (spanning the satellite flight direction and range direction), define a set of rays $\{r_j\}$ with origins $o_j$ and directions $v_j$ [S1]. The distance sampling $\{d_i\}$ is fixed and defined by the SAR range sampling resolution [S1].

2.  **Ray Origin Perturbation:**
    To improve optimization performance (analogous to noise injection in NeRF), perturb the ray origin. Since fixed uniform sampling along rays is not possible due to fixed SAR range sampling, noise is added to the origin [S1].
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
    *   $T_{i,j}$ is the transmittance (typically 1 in SAR as the medium is non-absorbing, or handled via the specific radar formulation) [S1].
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
    where $n \sim \Gamma(1, 1)$ (Gamma distribution with shape 1, scale 1, i.e., exponential distribution) [S1].

6.  **Optimization:**
    Minimize the difference between the rendered signal $s(d_i)$ and the observed SAR intensity $I$. The DSM height values and specularity parameter $\theta$ are optimized [S1]. Surface regularization can be applied using terms like:
    $$
    L_{d} = \frac{1}{|\mathcal{C}|} \sum_{C \in \mathcal{C}} \exp \left( -\frac{L}{d} \text{Tr}(\Sigma^{-1} C) \right)
    $$
    where $\Sigma$ is the covariance matrix of the surface [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class RadarField:
    def __init__(self, dsm_image, azimuth_planes, rays_per_plane=256):
        """
        Initialize Radar Field.
        
        Args:
            dsm_image: Tensor of shape (H, W) representing the rasterized DSM.
                       Values are normalized heights in [0, 1].
            azimuth_planes: List of azimuth plane configurations.
                           Each plane contains a set of ray origins and directions.
            rays_per_plane: Number of rays to sample per azimuth plane.
        """
        self.dsm = dsm_image.requires_grad_(True)
        self.theta = nn.Parameter(torch.ones_like(dsm_image) * 1.0) # Specularity param
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
        z = torch.nn.functional.grid_sample(
            self.dsm.unsqueeze(0).unsqueeze(0), 
            torch.stack([x, y], dim=-1).unsqueeze(0).unsqueeze(0), 
            mode='bilinear', align_corners=True
        ).squeeze()
        
        # Compute normal via finite differences or gradient of DSM
        # Assuming dsm is differentiable, compute gradients wrt x and y
        dz_dx, dz_dy = torch.autograd.grad(
            z.sum(), 
            [x, y], 
            create_graph=True
        )
        
        # Normal vector n = (-dz/dx, -dz/dy, 1) normalized
        n = torch.stack([-dz_dx, -dz_dy, torch.ones_like(z)], dim=-1)
        n = n / torch.norm(n, dim=-1, keepdim=True)
        
        return z, n

    def render_azimuth_plane(self, azimuth_plane_config):
        """
        Render a single azimuth plane.
        
        Args:
            azimuth_plane_config: Dictionary containing:
                - origins: Tensor (N, 3) ray origins o_j
                - directions: Tensor (N, 3) ray directions v_j
                - distances: Tensor (M,) fixed range samples d_i
                
        Returns:
            rendered_signal: Tensor (M,) signal for each range bin.
        """
        origins = azimuth_plane_config['origins']
        directions = azimuth_plane_config['directions']
        distances = azimuth_plane_config['distances']
        
        N = origins.shape[0]
        M = distances.shape[0]
        
        # 1. Perturb ray origins
        # w_j is orthogonal to v_j in the azimuth plane
        # For simplicity, assume w_j is precomputed or derived from plane geometry
        # n_noise ~ N(0, 1)
        noise = torch.randn(N, 1)
        w = azimuth_plane_config['orthogonal_vectors'] # (N, 3)
        perturbed_origins = origins + noise * w
        
        # 2. Compute intersections and normals for all rays and distances
        # In a real implementation, this involves marching along rays or solving
        # for intersection with the implicit surface F(x) = z - dsm(x,y) = 0.
        # Here we assume a helper function that returns sigma, normal, transmittance
        # for each (ray, distance) pair.
        
        # Initialize signal accumulator
        total_signal = torch.zeros(M, device=origins.device)
        
        # Batch processing: Iterate over rays
        for j in range(N):
            o_j = perturbed_origins[j]
            v_j = directions[j]
            
            # For each distance sample d_i, find intersection
            # Note: In SAR, distance is fixed by range sampling.
            # We assume the intersection point p_ij is at distance d_i from o_j along v_j
            # p_ij = o_j + d_i * v_j
            
            # However, the surface is defined by DSM. We must check if the surface
            # exists at the projected (x,y) of the ray at distance d_i.
            # Or, more accurately, the ray intersects the surface at some depth.
            # The paper states: "reflectors that are at the same distance are projected into the same pixel."
            # This implies we evaluate the surface at the point defined by the ray and the fixed range.
            
            # Calculate 3D point for this ray and all distances
            # p_ij = o_j + d_i * v_j
            # d_i is (M,), v_j is (3,), o_j is (3,)
            points = o_j.unsqueeze(0) + distances.unsqueeze(1) * v_j.unsqueeze(0) # (M, 3)
            
            # Get x, y coordinates
            x_coords = points[:, 0]
            y_coords = points[:, 1]
            
            # Get height and normal at these (x,y)
            z_vals, normals = self.get_normal_and_height(x_coords, y_coords)
            
            # Check if intersection is valid (e.g., z_val matches the z-coordinate of the point?)
            # The implicit function is F(x) = z - dsm(x,y).
            # The point on the ray is (x, y, z_ray). The surface height is dsm(x,y).
            # Intersection occurs if z_ray == dsm(x,y).
            # In the rendering equation, sigma and T handle the visibility.
            # For a DSM-based field, we often assume the surface is the top layer.
            
            # Compute dot product <v_j | n>
            # v_j is (3,), normals is (M, 3)
            dot_product = torch.sum(v_j.unsqueeze(0) * normals, dim=-1) # (M,)
            
            # Specular model: s = - <v|n> ^ theta
            # Note: The paper uses s = - <v|n>^theta. 
            # If dot_product is negative (back-facing), signal might be zero or handled.
            # Assuming front-facing for simplicity in this pseudocode structure.
            theta_val = self.theta[0, 0] # Scalar theta for now, or per-pixel
            
            # Signal contribution
            # T_ij and sigma_ij are assumed to be 1 or handled by the surface presence
            # The paper formula: s = - T * sigma * <v|n>^theta
            # We assume T=1, sigma=1 for the surface layer in DSM representation
            signal_j = - torch.pow(dot_product, theta_val)
            
            # Accumulate signal for this ray into the total signal for each distance bin
            # Since each ray contributes to a specific distance bin? 
            # No, the formula sums over j for a FIXED i.
            # So for each distance i, we sum the signal from all rays j.
            # But here we are iterating j. We need to add signal_j to total_signal.
            # Wait, the formula is s(d_i) = sum_j s_{r_j}(d_i).
            # So for each ray j, it contributes to ALL distance bins i?
            # Yes, because the ray passes through all range bins.
            # But does it intersect the surface at all bins?
            # The DSM defines the surface height. The ray intersects the surface ONCE.
            # The paper implies that for a given azimuth plane, we integrate across rays
            # at the SAME distance. This suggests that the "pixel" in SAR corresponds
            # to a specific range bin and azimuth angle.
            # The signal at range bin i is the sum of contributions from all rays
            # that pass through the volume at range bin i?
            # Actually, the paper says: "reflectors that are at the same distance are projected into the same pixel."
            # This means the SAR pixel (azimuth, range) collects energy from all scatterers
            # at that range distance within the azimuth swath.
            
            # Therefore, for each ray j, we calculate the signal it would produce
            # if it were the only ray, but we must determine IF it intersects the surface
            # at distance d_i.
            # In a DSM, the surface is at z = dsm(x,y).
            # The ray point at distance d_i is z_ray = o_j.z + d_i * v_j.z.
            # Intersection if z_ray == dsm(x,y).
            # This is a strict equality check, which is numerically unstable.
            # Usually, ray marching is used. But SAR range is fixed.
            # The paper likely assumes that the ray intersects the surface at the
            # distance d_i where the ray's z-coordinate matches the DSM height.
            # If not, sigma=0.
            
            # Let's refine the intersection logic:
            # For each ray j, find the distance d_k such that the ray intersects the DSM.
            # Then add the signal to total_signal[k].
            
            # Find intersection distance for ray j
            # Solve o_j.z + d * v_j.z = dsm(o_j.x + d*v_j.x, o_j.y + d*v_j.y)
            # This requires root finding or binary search over distances.
            
            # Simplified pseudocode for intersection:
            # dists = distances
            # zs_ray = o_j.z + dists * v_j.z
            # zs_surf = dsm evaluated at (o_j.x + dists * v_j.x, o_j.y + dists * v_j.y)
            # diff = zs_ray - zs_surf
            # Find index k where diff is closest to 0 (or crosses zero)
            
            # For the sake of the algorithm description, we assume a function
            # find_intersection(ray, distances) returns the index k and the signal.
            
            k, signal_at_intersection = self.find_intersection_and_signal(
                o_j, v_j, distances, self.dsm, self.theta
            )
            
            if k is not None:
                total_signal[k] += signal_at_intersection
                
        return total_signal

    def find_intersection_and_signal(self, origin, direction, distances, dsm, theta):
        """
        Find the intersection of a ray with the DSM and compute signal.
        """
        # Calculate z of ray at all distances
        z_ray = origin[2] + distances * direction[2]
        
        # Calculate x, y of ray at all distances
        x_ray = origin[0] + distances * direction[0]
        y_ray = origin[1] + distances * direction[1]
        
        # Get DSM height at these x, y
        z_surf = torch.nn.functional.grid_sample(
            dsm.unsqueeze(0).unsqueeze(0),
            torch.stack([x_ray, y_ray], dim=-1).unsqueeze(0).unsqueeze(0),
            mode='bilinear', align_corners=True
        ).squeeze()
        
        # Difference
        diff = z_ray - z_surf
        
        # Find minimum absolute difference (closest intersection)
        min_diff, idx = torch.min(torch.abs(diff), dim=0)
        
        # Threshold for intersection
        if min_diff > 0.01: # Arbitrary threshold
            return None, None
            
        # Compute normal at intersection
        # Re-evaluate normal at the specific intersection point
        x_int = x_ray[idx]
        y_int = y_ray[idx]
        _, normal = self.get_normal_and_height(x_int, y_int)
        
        # Dot product
        dot = torch.dot(direction, normal)
        
        # Signal
        signal = - torch.pow(dot, theta)
        
        return idx.item(), signal

    def train_step(self, target_signal, azimuth_plane_config):
        """
        Perform one optimization step.
        """
        rendered_signal = self.render_azimuth_plane(azimuth_plane_config)
        
        # Loss: MSE between rendered and target (accounting for speckle noise)
        # For speckle noise, a multiplicative loss or Gamma likelihood is better.
        # Simplified to MSE for pseudocode.
        loss = torch.nn.functional.mse_loss(rendered_signal, target_signal)
        
        loss.backward()
        
        # Update parameters
        with torch.no_grad():
            self.dsm -= 0.1 * self.dsm.grad
            self.dsm.grad.zero_()
            self.theta -= 0.1 * self.theta.grad
            self.theta.grad.zero_()
            
        return loss.item()
```

**Parameters:**
*   **Specularity $\theta$:** Initial value 1.0 (Lambertian). Can be learned [S1].
*   **Learning Rate:** Initial 1.0, reduced to 0.1 at step 5,000, and to $10^{-2}$ at step 8,000 [S1].
*   **Training Steps:** 10,000 steps [S1].
*   **Batching:** Azimuth planes are sampled uniformly and grouped into batches of size 64 [S1].
*   **Rays per Plane:** 256 rays sampled per azimuth plane [S1].

**Edge Cases and Pitfalls:**
*   **Multi-bounce Effects:** Signals from double or multiple bounces (e.g., A->B->receiver) appear at incorrect positions (e.g., at C). This violates the zero-doppler assumption, requiring batching multiple azimuth planes, which is highly unfeasible without strong priors [S1].
*   **Fixed Sampling:** Fixed uniform sampling along rays is not possible because the sampling step is defined by the SAR range sampling. Noise must be added to the ray origin instead [S1].
*   **Complex Data:** SAR raw data stores complex numbers. The method typically uses intensity images, losing phase information which contains path delay data [S1].

## Operating characteristics
*   **Time Complexity:** On average, the computations necessary to learn a radar field remain the same as those for a radiance field, with the primary difference being the batching strategy [S1].
*   **Memory Complexity:** Requires batching all rays of a given azimuth plane together during computation, as computing a single pixel requires computing all rays in that azimuth plane [S1].
*   **Performance:** Reconstruction errors are in the order of $10^{-2}$ pixels relative to ground pixel resolution [S1].
*   **Strengths:** Enables 3D reconstruction from SAR data, leveraging its day-night and cloud-penetrating capabilities. Does not prescribe geometric configuration of acquisitions, allowing opportunistic use [S1].
*   **Limitations:** Struggles with multi-bounce effects. Requires significant batching for azimuth planes. Assumes non-absorbing medium [S1].

## Background
Radar Fields was introduced to extend radiance fields to Synthetic Aperture Radar (SAR) data. SAR systems emit electromagnetic pulses and receive backscattered echoes. Unlike optical sensors, SAR wavelengths experience minimal interaction with atmospheric gases, allowing for imaging in all weather conditions and at night [S1]. The method adapts the differentiable rendering framework to account for the unique SAR image formation model, where integration occurs across rays in an azimuth plane rather than along individual rays [S1].

## Sources
[S1] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
