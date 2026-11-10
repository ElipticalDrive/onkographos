# 3D Gaussian Splatting Self-Shadowing (T_light)

Also called: 光源视角自阴影 (T_light)

## Overview
Calculates the self-shadowing transmittance $T_{light}$ for each Gaussian primitive along the sun direction, representing the 'front occlusion transmittance' to model self-shadowing effects in volume rendering [S1, unofficial]. It sets itself apart from sibling methods by performing a dedicated light-space rasterization pass that accumulates depth-weighted transmittance without modifying the primary camera's covariance expansion or applying amplitude compensation, ensuring consistency with the main rendering pass [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian primitives (position, covariance/scale/rotation, opacity, color/SH), sun direction (simulated via a distant perspective camera), and the accumulated transmittance state from previous Gaussians in the light path.
*   **Outputs:** Per-Gaussian self-shadowing transmittance value $T_{light}$ (or `front_tau`), which modulates the final illumination.

**Data Structures:**
*   **Gaussian Buffer:** Standard 3D Gaussian attributes.
*   **Light-Space Projection Attributes:** Projected 2D position, 2D covariance matrix, and depth.
*   **Accumulation Buffer:** Stores the accumulated front transmittance $\tau$ for the current pixel/fragment in the light pass.
*   **Record Channel:** `record_front_tau` stores the energy-weighted mean transmittance for each Gaussian.

**Steps:**
1.  **Light Camera Setup:** Construct a virtual perspective camera to simulate a directional sun light source. Place it at a large distance with a narrow Field of View (FOV) to minimize parallax error to below 2% [S1, unofficial]. This setup avoids modifying the EWA Jacobian used in the main pass.
2.  **Covariance Handling:** During this light pass, do **not** expand the projection covariance by $0.3I$ and do **not** apply amplitude compensation. Preserve the sampling expansion from the main camera pass exactly as is [S1, unofficial].
3.  **Depth Sorting:** Sort the Gaussians by their depth relative to the sun camera (light-space depth) to ensure correct front-to-back accumulation [S1, unofficial].
4.  **Rasterization and Accumulation:**
    *   Traverse the sorted Gaussians.
    *   For each Gaussian, calculate its contribution to the accumulated front transmittance $\tau$.
    *   The transmittance is weighted by the Gaussian's weight (opacity/density contribution).
    *   Record the accumulated front transmittance $\tau$ weighted by $\tau \times \text{weight}$ into the `record_front_tau` channel. This represents an energy-weighted mean over the entire sun-facing footprint, not just a center point sample [S1, unofficial].
5.  **Gradient Treatment:** Enable full continuous gradients for $\tau$, weights, normalization, weak density backup transmittance, and projection position/scale/rotation [S1, unofficial]. However, treat sun camera framing, sorting, coverage, and threshold branches as constants (detach from gradient computation) to prevent incorrect gradient propagation [S1, unofficial].
6.  **Edge Case Handling:** Handle deeply buried Gaussians (where early termination causes the accumulated weight sum $wsum=0$) by explicitly mapping them to full shadow ($T_{light}=0$) to prevent inversion artifacts [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_tlight(gaussians, sun_camera, main_pass_covariance_expansion_factor=0.3):
    """
    Computes self-shadowing transmittance T_light for 3D Gaussians.
    
    Args:
        gaussians: List of Gaussian primitives with position, covariance, opacity, etc.
        sun_camera: Virtual camera object simulating directional light.
        main_pass_covariance_expansion_factor: Factor used in main pass (0.3).
        
    Returns:
        t_light: Array of transmittance values for each Gaussian.
    """
    
    # Step 1: Project Gaussians into Light Space
    # Do NOT apply the 0.3I expansion or amplitude compensation here
    light_space_gaussians = []
    for g in gaussians:
        # Project position and covariance to light space
        pos_2d, cov_2d, depth = project_to_light_space(g, sun_camera)
        
        # Critical: Use raw projected covariance, do not expand by 0.3I
        # cov_2d_light = cov_2d  # No expansion
        # No amplitude compensation applied
        
        light_space_gaussians.append({
            'original_index': g.index,
            'pos_2d': pos_2d,
            'cov_2d': cov_2d,
            'depth': depth,
            'opacity': g.opacity,
            'weight': g.weight # Pre-computed or derived weight
        })
    
    # Step 2: Sort by Light-Space Depth (Front-to-Back)
    # Sorting is treated as a constant operation for gradients
    light_space_gaussians.sort(key=lambda x: x['depth'])
    
    # Initialize accumulation buffer for the current pixel/fragment
    # In a rasterizer, this is per-pixel. Here we simulate the accumulation logic.
    accumulated_tau = 1.0 # Initial transmittance is 1 (fully transparent)
    
    # Step 3: Rasterize and Accumulate
    record_front_tau = {} # Map from original_index to recorded tau
    
    for g in light_space_gaussians:
        # Calculate coverage/weight contribution for this Gaussian in light space
        # This involves evaluating the 2D Gaussian kernel over the pixel footprint
        contribution_weight = calculate_footprint_weight(g['cov_2d'], g['pos_2d'])
        
        # Update accumulated transmittance
        # tau_new = tau_old * (1 - contribution)
        # However, the note specifies recording tau * weight
        # We assume 'weight' here refers to the opacity/density contribution
        
        current_tau = accumulated_tau
        accumulated_tau *= (1.0 - contribution_weight)
        
        # Record the front transmittance weighted by the Gaussian's weight
        # This is an energy-weighted mean over the footprint
        record_front_tau[g['original_index']] = current_tau * g['weight']
        
        # Handle deeply buried Gaussians
        if accumulated_tau < epsilon: # Early termination threshold
            # Explicitly map to full shadow to prevent inversion artifacts
            # Any subsequent Gaussians (deeper) will have tau ~ 0
            record_front_tau[g['original_index']] = 0.0
            # Break or continue with tau=0 depending on implementation
            # Usually, we continue but tau remains 0
            
    # Step 4: Assign T_light
    t_light = []
    for g in gaussians:
        # Retrieve recorded value
        tau_val = record_front_tau.get(g.index, 0.0)
        t_light.append(tau_val)
        
    return t_light

def project_to_light_space(gaussian, camera):
    """
    Projects a 3D Gaussian to 2D in light space.
    Returns 2D position, 2D covariance, and depth.
    """
    # Standard EWA projection logic
    # 1. Transform 3D position to camera space
    pos_cam = camera.world_to_camera(gaussian.position)
    
    # 2. Project to 2D
    pos_2d = camera.projection(pos_cam)
    
    # 3. Compute 2D covariance
    # J is the Jacobian of the projection
    # cov_2d = J * cov_3d * J^T
    cov_2d = compute_ewa_covariance(gaussian.cov_3d, camera, pos_cam)
    
    # Return depth for sorting
    depth = pos_cam.z
    
    return pos_2d, cov_2d, depth

def calculate_footprint_weight(cov_2d, pos_2d):
    """
    Calculates the weight contribution of a Gaussian over the pixel footprint.
    """
    # Evaluate the 2D Gaussian integral over the pixel area
    # This is typically done via sampling or analytical integration
    # For simplicity, we assume a pre-computed weight or a sample-based estimate
    weight = integrate_gaussian_over_pixel(cov_2d, pos_2d)
    return weight
```

**Parameters:**
*   **Sun Camera FOV:** Narrow FOV to minimize parallax error (< 2%) [S1, unofficial].
*   **Covariance Expansion:** Disabled ($0.3I$ expansion is skipped) [S1, unofficial].
*   **Amplitude Compensation:** Disabled [S1, unofficial].
*   **Fallback Mode:** If the `--tlight_voxel` flag is passed, use the old $128^3$ voxel cache path instead of rasterization. This is compatible with models trained before rasterization support [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Deeply Buried Gaussians:** If early termination causes the accumulated weight sum $wsum=0$, the Gaussian must be explicitly mapped to full shadow ($T_{light}=0$) to prevent inversion artifacts [S1, unofficial].
*   **Gradient Propagation:** Sun camera framing, sorting, coverage, and threshold branches must be treated as constants (no gradients) to prevent incorrect gradient propagation [S1, unofficial].
*   **Covariance Mismatch:** Failing to skip the $0.3I$ expansion and amplitude compensation in the light pass will cause inconsistency with the main camera pass [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians $N$ for the sorting and rasterization pass, $O(N \log N)$ due to sorting, plus $O(N)$ for rasterization. The rasterization pass is comparable in cost to the main pass but uses a different camera projection.
*   **Memory Complexity:** $O(N)$ to store the light-space attributes and the `record_front_tau` channel. The voxel cache fallback uses $O(128^3)$ memory.
*   **Performance:** The method is efficient as it reuses the rasterization infrastructure. The narrow FOV and distant camera setup minimize parallax error without requiring complex ray tracing.
*   **Strengths:** Accurate self-shadowing for volume-rendered Gaussians; consistent with the main pass's sampling; efficient rasterization-based approach.
*   **Limitations:** Requires an additional rendering pass; sensitive to the correctness of the light-space projection and sorting; parallax error is minimized but not zero (though < 2%).
*   **When to Use:** When self-shadowing is required for 3D Gaussian Splatting scenes, particularly for volume-like structures (e.g., clouds, smoke) or dense objects where occlusion is significant.
*   **When Not to Use:** For simple, transparent, or non-occluding scenes where self-shadowing is negligible; if computational budget is extremely tight and an additional pass is prohibitive.

## Background
Originated in the context of 4D Gaussian Splatting for volume clouds, as described in the `4DGS-Volume-Cloud` repository [S1, unofficial]. The method addresses the lack of self-shadowing in standard 3D Gaussian Splatting, which treats Gaussians as opaque or transparent billboards without considering occlusion from other Gaussians along the light direction. It builds on the principles of light-space rasterization and front-to-back accumulation.

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
