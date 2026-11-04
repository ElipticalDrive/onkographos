# Light Space Rasterization for T_light

Also called: Front Tau Rasterization, Light Space Shadow Rasterization, T_light Rasterization (Light Space)

## Overview
Light Space Rasterization computes $T_{light}$, the front-facing occlusion transmittance for each 3D Gaussian along a directional sun direction, enabling physically plausible self-shadowing in volume cloud rendering. It differs from voxel-based fallbacks by providing full continuous gradients and zero parameter expansion, allowing for differentiable relighting without requiring specific training hyperparameters.

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Gaussian splats (position, covariance/scale, rotation, opacity $\alpha$, color/SH coefficients), sun direction vector $\mathbf{d}_{sun}$.
*   **Output:** Per-Gaussian scalar $T_{light} \in [0, 1]$, representing the accumulated transmittance from the sun direction to the Gaussian center.

**Data Structures:**
*   **Gaussian Attributes:** Standard 3DGS attributes. The method relies on the projected position and scale in light space.
*   **Accumulator:** A per-pixel or per-Gaussian accumulation buffer for optical depth $\tau$. In the CUDA kernel, this is managed via the `record_front_tau` channel.

**Algorithm Steps:**

1.  **Light Camera Setup:**
    Construct a virtual perspective camera to simulate the directional sun light.
    *   **Position:** Placed at a large distance $D$ along the sun direction $\mathbf{d}_{sun}$.
    *   **Orientation:** Look-at direction is $-\mathbf{d}_{sun}$.
    *   **Field of View (FOV):** Narrow FOV. The distance $D$ and FOV are chosen such that parallax error is $<2\%$, ensuring the rays are effectively parallel for the purpose of the EWA Jacobian approximation [S1, unofficial].
    *   **Projection:** Standard perspective projection matrix.

2.  **Light-Space Projection:**
    Project all 3D Gaussians into the light camera's clip space.
    *   Transform Gaussian centers $\mathbf{p}_i$ using the light camera's view-projection matrix $M_{light}$.
    *   Compute the projected 2D covariance matrix $\Sigma_{2D}$ for each Gaussian in screen space, similar to standard 3DGS rasterization, but using the light camera's intrinsics and extrinsics.

3.  **Depth Sorting:**
    Sort the Gaussians by their depth $z$ in light space, from front (closest to light source) to back. This ensures correct front-to-back accumulation for transmittance.

4.  **Rasterization Pass (CUDA Kernel):**
    Iterate over screen pixels covered by the sorted Gaussians. For each Gaussian $i$ covering pixel $(u, v)$:
    *   **Coverage:** Determine if the Gaussian contributes to the pixel based on its projected 2D Gaussian distribution.
    *   **Accumulation:** Update the accumulated optical depth $\tau_{acc}$ for the pixel.
    *   **Recording:** Each Gaussian records the accumulated front $\tau$ weighted by $\alpha \cdot T$. Specifically, the method computes an energy-weighted mean over the entire sun-facing footprint, rather than a simple center-point sample [S1, unofficial].
    *   **Transmittance Calculation:** The transmittance $T_{light}$ for a Gaussian is derived from the accumulated optical depth of all Gaussians in front of it along the sun direction.
        $$ T_{light, i} = \exp\left(-\sum_{j \in \text{front}(i)} \sigma_j \cdot \Delta s_j \right) $$
        Where $\sigma_j$ is the density and $\Delta s_j$ is the path length contribution. In practice, this is approximated via the rasterization accumulation buffer.

5.  **Fallback and Edge Cases:**
    *   **Deeply Buried Gaussians:** If a Gaussian is deeply buried such that the accumulated weight sum $w_{sum} \approx 0$ (early termination), it is explicitly mapped to full shadow ($T_{light} = 0$) to prevent inversion artifacts [S1, unofficial].
    *   **Gradient Handling:** Full continuous gradients are enabled by default for $\tau$, weights, normalization, weak density backup transmittance, and projected position/scale/rotation. The sun camera framing, sorting, coverage, and threshold branches are treated as constants (no gradients through them) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_tlight_rasterization(gaussians, sun_direction):
    """
    Compute T_light for all Gaussians using light-space rasterization.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - center: (3,) float32
            - scale: (3,) float32
            - rotation: (4,) float32 (quaternion)
            - alpha: float32
            - density: float32 (or derived from scale/alpha)
        sun_direction: (3,) float32, normalized sun vector.
        
    Returns:
        t_light: (N,) float32 array, transmittance for each Gaussian.
    """
    # 1. Setup Light Camera
    # Place camera far away along sun_direction
    cam_pos = sun_direction * LARGE_DISTANCE_CONSTANT
    cam_lookat = -sun_direction
    cam_up = (0, 1, 0) if abs(dot(sun_direction, (0,1,0))) < 0.9 else (1, 0, 0)
    cam_right = normalize(cross(cam_lookat, cam_up))
    cam_up = cross(cam_right, cam_lookat)
    
    # Construct View Matrix
    view_matrix = look_at(cam_pos, cam_pos + cam_lookat, cam_up)
    
    # Construct Projection Matrix (Narrow FOV)
    fov = NARROW_FOV_RADIANS
    aspect = 1.0 # Assume square or handle aspect ratio
    proj_matrix = perspective_projection(fov, aspect, NEAR_PLANE, FAR_PLANE)
    
    # 2. Project Gaussians to Light Space
    projected_gaussians = []
    for i, g in enumerate(gaussians):
        # Transform center
        center_h = (g.center[0], g.center[1], g.center[2], 1.0)
        center_cam = mat_mul(view_matrix, center_h)
        center_clip = mat_mul(proj_matrix, center_cam)
        
        # Perspective divide
        if center_clip[3] <= 0:
            continue # Behind camera
            
        center_ndc = (center_clip[0]/center_clip[3], 
                      center_clip[1]/center_clip[3], 
                      center_clip[2]/center_clip[3])
        
        # Project covariance (simplified EWA approximation)
        # J is the Jacobian of the projection
        # Sigma_2D = J * R * S * R^T * J^T
        # This step is complex; in practice, use the standard 3DGS projection kernel
        # but with light camera matrices.
        cov_2d = project_covariance(g, view_matrix, proj_matrix)
        
        projected_gaussians.append({
            'index': i,
            'center_ndc': center_ndc,
            'cov_2d': cov_2d,
            'alpha': g.alpha,
            'density': g.density, # Or derived
            'depth': center_ndc[2]
        })
        
    # 3. Sort by Depth (Front to Back)
    projected_gaussians.sort(key=lambda x: x['depth'])
    
    # 4. Rasterization Pass
    # Initialize accumulation buffer for optical depth
    # Size: Screen Width x Screen Height
    # Using a hash map or sparse structure for efficiency if screen is large
    # For simplicity, assume a dense buffer or use the CUDA kernel logic
    
    # The CUDA kernel 'record_front_tau' handles the accumulation.
    # Here we simulate the logic:
    
    # We need to store the accumulated tau at the center of each Gaussian
    # to compute its T_light.
    
    # Since rasterization is pixel-based, we need to query the accumulated
    # tau at the Gaussian's center position in light space.
    
    # However, the method states: "each Gaussian records the accumulated 
    # front tau weighted by alpha*T". This implies the accumulation is 
    # done during the rasterization of the Gaussian itself or its predecessors.
    
    # Correct interpretation:
    # As we rasterize Gaussians from front to back, we maintain an 
    # accumulated optical depth buffer `tau_buffer` for each pixel.
    # When a Gaussian is rasterized, we sample `tau_buffer` at its center 
    # (or integrate over its footprint) to get the tau *before* it.
    # Then we update `tau_buffer` with this Gaussian's contribution.
    
    tau_buffer = {} # Map (u,v) -> accumulated_tau
    
    t_light_values = [0.0] * len(gaussians)
    
    for g in projected_gaussians:
        u, v, z = g['center_ndc']
        
        # Sample accumulated tau at this Gaussian's center
        # In a real rasterizer, this is done via texture lookup or atomic add
        tau_before = tau_buffer.get((u, v), 0.0)
        
        # Compute T_light for this Gaussian
        # T_light = exp(-tau_before)
        # But the source mentions "energy-weighted mean over the entire sun-facing footprint"
        # This suggests integrating tau over the Gaussian's 2D footprint.
        # For implementation simplicity, we use the center sample as a proxy,
        # or perform a small integration if high precision is needed.
        
        t_light = exp(-tau_before)
        
        # Handle deeply buried Gaussians
        if t_light < THRESHOLD_MIN_TRANSMITTANCE:
            t_light = 0.0
            
        t_light_values[g['index']] = t_light
        
        # Update tau_buffer with this Gaussian's contribution
        # Contribution = sigma * delta_s
        # In 3DGS, sigma is related to alpha and scale.
        # Approximate contribution as alpha * density_factor
        contribution = g['alpha'] * g['density'] * SCALE_FACTOR
        
        # Add to all pixels covered by this Gaussian
        # This is the expensive part; usually handled by the rasterizer's 
        # fragment shader/kernel.
        cover_pixels(g['cov_2d'], g['center_ndc'], contribution, tau_buffer)
        
    return t_light_values

def cover_pixels(cov_2d, center_ndc, contribution, tau_buffer):
    """
    Add contribution to tau_buffer for all pixels covered by the Gaussian.
    This is a simplified simulation of the rasterization step.
    """
    # Determine bounding box of the 2D Gaussian
    # sqrt(eigenvalues) * 3 sigma
    eigenvalues = eigenvalues_2x2(cov_2d)
    sigma_x = sqrt(eigenvalues[0]) * 3.0
    sigma_y = sqrt(eigenvalues[1]) * 3.0
    
    min_u = center_ndc[0] - sigma_x
    max_u = center_ndc[0] + sigma_x
    min_v = center_ndc[1] - sigma_y
    max_v = center_ndc[1] + sigma_y
    
    for u in range(floor(min_u), ceil(max_u)):
        for v in range(floor(min_v), ceil(max_v)):
            # Check if pixel is within Gaussian footprint
            # Evaluate 2D Gaussian PDF
            x = u - center_ndc[0]
            y = v - center_ndc[1]
            pdf = exp(-0.5 * (x*x/cov_2d[0][0] + y*y/cov_2d[1][1])) # Simplified
            
            if pdf > THRESHOLD:
                key = (u, v)
                tau_buffer[key] = tau_buffer.get(key, 0.0) + contribution
```

**Parameters:**
*   `LARGE_DISTANCE_CONSTANT`: Distance of light camera from origin. Value not given in sources, but must be large enough to ensure parallel rays.
*   `NARROW_FOV_RADIANS`: FOV of light camera. Value not given, but must be narrow to minimize parallax error (<2%).
*   `THRESHOLD_MIN_TRANSMITTANCE`: Threshold for mapping deeply buried Gaussians to full shadow. Value not given.
*   `SCALE_FACTOR`: Scaling factor for density contribution. Value not given.

**Pitfalls:**
*   **Parallax Error:** If the light camera is not far enough or FOV is too wide, parallax error exceeds 2%, affecting the EWA Jacobian accuracy [S1, unofficial].
*   **Inversion Artifacts:** Deeply buried Gaussians must be explicitly mapped to full shadow to prevent artifacts [S1, unofficial].
*   **Gradient Discontinuities:** The sun camera framing, sorting, and threshold branches are treated as constants. Gradients do not flow through these steps [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of Gaussians and pixels covered, similar to standard 3DGS rasterization. The additional cost is the light-space projection and sorting.
*   **Memory Complexity:** Linear in the number of Gaussians. No additional voxel grid expansion is required, unlike the voxel fallback [S1, unofficial].
*   **Performance:** Provides full continuous gradients and zero expansion by default, making it more efficient and easier to train than the voxel fallback (`--tlight_voxel`) [S1, unofficial].
*   **Strengths:** Differentiable, no parameter expansion, handles relighting for held-out sun directions.
*   **Limitations:** Approximates parallel rays, which may introduce small errors for very large scenes or extreme sun angles.

## Background
This method is part of the 4DGS-Volume-Cloud project, which extends 3D Gaussian Splatting to volume cloud rendering. It enables relighting of volume clouds by computing self-shadowing via light-space rasterization. The method was introduced in the GitHub repository `hsiang0117/4DGS-Volume-Cloud` [S1, unofficial]. It is designed to work with datasets like `CloudDatasetUniform`, which includes multiple sun directions for training and testing relighting generalization.

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
