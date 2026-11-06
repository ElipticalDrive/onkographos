# Bidirectional Volume Boundary Estimator

Also called: Bidirectional estimator, EstimateBoundaryIntegral, Multi-Directional Boundary Integral Estimation, Multi-Directional Form of the Boundary Integral, Multi-directional estimation of the boundary integral, Multi-directional volume boundary sampling

## Overview
A Monte Carlo estimator that samples volume boundary paths by first drawing the volume boundary vertex, then constructing two subpaths to connect it to the light source and the detector [S1][S2, unofficial]. It sets itself apart from its parent, the [Unidirectional Volume Boundary Estimator](3538-unidirectional-volume-boundary-estimator.md), by decoupling the boundary integral from the interior term and building boundary light paths in a multi-directional manner to avoid costly silhouette edge searches [S2, unofficial]. Unlike the unidirectional approach which builds paths from the detector, this method samples the boundary vertex first, enabling a symmetric construction of source and detector subpaths [S1].

## How it works
The estimator computes the boundary integral contribution by sampling a boundary segment defined by a spatial point $\mathbf{x}_B$ and direction $\boldsymbol{\omega}_B$, then estimating source and detector subpaths [S2, unofficial].

**Inputs:**
- Probability density $P(\mathbf{x}_B, \boldsymbol{\omega}_B)$ for sampling the boundary segment.
- Boolean `direct`: indicates whether to use Next-Event Estimation (NEE) for direct illumination or estimate indirect paths.
- Scene geometry $M(\pi)$ and material properties.

**Outputs:**
- The estimated boundary term contribution $T_S T_B T_D$.

**Data Structures:**
- Boundary segment vertices: $\mathbf{p}_0^S$ (source side), $\mathbf{p}_0^D$ (detector side).
- Spatial points: $\mathbf{x}_0^S = \mathbf{X}(\mathbf{p}_0^S, \pi)$, $\mathbf{x}_0^D = \mathbf{X}(\mathbf{p}_0^D, \pi)$.
- Boundary vertex: $\mathbf{x}_B$ on the silhouette.
- Direction: $\boldsymbol{\omega}_B$.

**Steps:**

1. **Sample Boundary Segment:**
   Draw $(\mathbf{x}_B, \boldsymbol{\omega}_B)$ from the probability density $P(\mathbf{x}_B, \boldsymbol{\omega}_B)$ [S2, unofficial].
   Trace a ray from $\mathbf{x}_B$ in direction $-\boldsymbol{\omega}_B$ to find intersection $\mathbf{x}_0^S$ on the mesh [S2, unofficial].
   Trace a ray from $\mathbf{x}_B$ in direction $\boldsymbol{\omega}_B$ to find intersection $\mathbf{x}_0^D$ on the mesh [S2, unofficial].
   *Note:* The spatial counterpart of the boundary segment $\mathbf{x}_0^S \mathbf{x}_0^D$ must intersect the evolving surface $M(\pi)$ at exactly one point $\mathbf{x}_B$ besides the endpoints [S2, unofficial].

2. **Validate Intersections:**
   If $\mathbf{x}_0^S$ or $\mathbf{x}_0^D$ are invalid (no intersection or occlusion), return 0 [S2, unofficial].

3. **Compute Boundary Term ($T_B$):**
   Calculate the boundary contribution function $\hat{f}_B$:
   $$ \hat{f}_B := \Delta G(\mathbf{x}_0^S \leftrightarrow \mathbf{x}_0^D) V_{\Delta B} $$
   where $V_{\Delta B}$ is the scalar normal velocity of $\mathbf{p}_0^D$ with $\mathbf{p}_0^S$ fixed [S2, unofficial].
   Compute the Jacobian $J_B(\mathbf{x}_B, \boldsymbol{\omega}_B)$. In the local parameterization where $\mathbf{X}(\cdot, \pi_0)$ and $\mathbf{P}(\cdot, \pi_0)$ are identity maps:
   $$ J_B(\mathbf{x}_B, \boldsymbol{\omega}_B) = \frac{\|\mathbf{x}_0^D - \mathbf{x}_0^S\|}{\|\mathbf{x}_B - \mathbf{x}_0^S\|} \frac{\sin \theta_B}{\sin \theta_D |\cos \theta_S|} $$
   where $\theta_B$ is the angle between $\boldsymbol{\omega}_B$ and the face edge at $\mathbf{x}_B$, $\theta_D$ is the angle between $\boldsymbol{\omega}_B$ and the visibility boundary at $\mathbf{x}_0^D$, and $\theta_S$ is the angle between $\boldsymbol{\omega}_B$ and the surface normal at $\mathbf{x}_0^S$ [S2, unofficial].
   Compute $T_B$:
   $$ T_B = \frac{\hat{f}_B J_B(\mathbf{x}_B, \boldsymbol{\omega}_B)}{P(\mathbf{x}_B, \boldsymbol{\omega}_B)} $$
   [S2, unofficial].

4. **Compute Source Term ($T_S$):**
   Map spatial points to path space: $\mathbf{p}_0^S = \mathbf{P}(\mathbf{x}_0^S, \pi)$ and $\mathbf{p}_0^D = \mathbf{P}(\mathbf{x}_0^D, \pi)$ [S2, unofficial].
   If `direct` is true:
   $$ T_S = L_e(\mathbf{x}_0^S \to \mathbf{x}_0^D) $$
   [S2, unofficial].
   Else:
   $$ T_S = \text{EstimateSourcePath}(\mathbf{p}_0^S; \mathbf{p}_0^D) $$
   [S2, unofficial].

5. **Compute Detector Term ($T_D$):**
   $$ T_D = \text{EstimateDetectorPath}(\mathbf{p}_0^D; \mathbf{p}_0^S) $$
   [S2, unofficial].

6. **Return Result:**
   Return $T_S T_B T_D$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def EstimateBoundaryIntegral(P_boundary, direct, scene):
    """
    Estimates the boundary integral using the bidirectional/multi-directional approach.
    
    Args:
        P_boundary: Function to sample (x_B, omega_B) from probability density P(x_B, omega_B).
        direct: Boolean, if True use Next-Event Estimation for direct illumination.
        scene: Object containing geometry M(pi) and material properties.
        
    Returns:
        Float: The estimated boundary term contribution T_S * T_B * T_D.
    """
    # Step 1: Sample boundary segment
    x_B, omega_B = P_boundary()
    
    # Trace rays to find endpoints of the boundary segment
    # Ray from x_B in direction -omega_B
    x_0S = scene.ray_trace(x_B, -omega_B)
    # Ray from x_B in direction omega_B
    x_0D = scene.ray_trace(x_B, omega_B)
    
    # Step 2: Validate intersections
    if x_0S is None or x_0D is None:
        return 0.0
        
    # Step 3: Compute Boundary Term T_B
    # Calculate geometric term Delta G
    # Delta G represents the change in geometry term across the boundary
    # For a static scene or specific parameterization, this might be derived from visibility
    # Here we assume a helper function exists for Delta G based on the segment
    delta_G = compute_delta_geometry(x_0S, x_0D)
    
    # V_delta_B: Scalar normal velocity of p_0^D with p_0^S fixed
    # This requires knowledge of the motion/derivative parameter pi
    # [G] Assuming scene provides normal velocity for the boundary vertex
    V_delta_B = scene.get_normal_velocity(x_0D, x_0S)
    
    f_hat_B = delta_G * V_delta_B
    
    # Compute Jacobian J_B
    # Angles: theta_B (omega_B vs face edge at x_B), theta_D (omega_B vs visibility boundary at x_0D),
    # theta_S (omega_B vs surface normal at x_0S)
    # [G] Helper functions to compute these angles based on geometry
    theta_B = angle_between(omega_B, get_face_edge_at(x_B))
    theta_D = angle_between(omega_B, get_visibility_boundary_at(x_0D))
    theta_S = angle_between(omega_B, get_surface_normal_at(x_0S))
    
    dist_x0S_x0D = norm(x_0D - x_0S)
    dist_xB_x0S = norm(x_0S - x_B)
    
    # Handle degenerate cases where distances or cosines are zero
    if dist_xB_x0S == 0 or abs(cos(theta_S)) < 1e-8:
        return 0.0
        
    J_B = (dist_x0S_x0D / dist_xB_x0S) * (sin(theta_B) / (sin(theta_D) * abs(cos(theta_S))))
    
    # Probability density of the sample
    p_val = P_boundary.pdf(x_B, omega_B)
    if p_val == 0:
        return 0.0
        
    T_B = (f_hat_B * J_B) / p_val
    
    # Step 4: Compute Source Term T_S
    # Map to path space
    p_0S = scene.project_to_path_space(x_0S)
    p_0D = scene.project_to_path_space(x_0D)
    
    if direct:
        # Direct illumination: Emitter radiance from x_0S to x_0D
        # [G] Assuming scene provides emitter radiance function
        T_S = scene.emitter_radiance(x_0S, x_0D)
    else:
        # Indirect illumination: Estimate source subpath
        # [G] Placeholder for the actual source path estimation logic
        # This typically involves unidirectional path tracing from x_0S towards the light
        T_S = estimate_source_path(p_0S, p_0D, scene)
        
    # Step 5: Compute Detector Term T_D
    # [G] Placeholder for the actual detector path estimation logic
    # This typically involves unidirectional path tracing from x_0D towards the camera
    T_D = estimate_detector_path(p_0D, p_0S, scene)
    
    # Step 6: Return product
    return T_S * T_B * T_D
```

**Parameters:**
- $V_{\Delta B}$: Scalar normal velocity of $\mathbf{p}_0^D$ with $\mathbf{p}_0^S$ fixed [S2, unofficial].
- `direct`: Boolean flag for using NEE [S2, unofficial].

**Pitfalls:**
- Identifying silhouette edges is costly [S2, unofficial]. The method avoids this by sampling $\mathbf{x}_B$ and $\boldsymbol{\omega}_B$ directly, ensuring the segment intersects the surface at $\mathbf{x}_B$ [S2, unofficial].
- Precision issues may arise if $\mathbf{x}_B$ is very close to $\mathbf{x}_0^S$ or if $\cos \theta_S$ is near zero [S2, unofficial].
- The method assumes the boundary segment's spatial counterpart intersects the evolving surface at exactly one point $\mathbf{x}_B$ besides the endpoints [S2, unofficial].

## Subtopics

### [Multi-Directional Sampling of Boundary Paths](6393-multi-directional-sampling-of-boundary-paths.md)
Reparameterizes the boundary segment from two independent endpoints to an interior point and a direction, guaranteeing endpoints lie on each other's visibility boundary by construction.

### [Multiple Importance Sampling for Boundary Paths](6397-multiple-importance-sampling-for-boundary-paths.md)
Combines the unidirectional and bidirectional volume boundary estimators using path-level MIS with the balanced heuristic to reduce variance.

## Operating characteristics
- **Time Complexity:** Depends on the cost of sampling the boundary segment and tracing rays. Avoids explicit silhouette search, which is costly for complex geometries [S2, unofficial].
- **Memory Complexity:** Low, as it does not require storing large silhouette edge lists [S2, unofficial].
- **Strengths:** Unbiased estimation without expensive silhouette identification [S2, unofficial]. Conceptually similar to multi-directional techniques for ordinary boundary paths but applied to volume boundary paths [S1].
- **Limitations:** Requires careful sampling of the boundary segment to ensure valid intersections [S2, unofficial]. Performance depends on the efficiency of the source and detector path estimation subroutines.
- **When to use:** When dealing with complex geometries where silhouette identification is prohibitive, or when unbiased boundary path sampling is required [S2, unofficial].
- **When not to use:** If direct silhouette information is already available and cheap to use, other methods might be simpler.

## Background
Originated in the context of path-space differentiable rendering [S2, unofficial]. The bidirectional estimator is conceptually similar to the multi-directional technique introduced by Zhang et al. [ZMY* 20] for sampling ordinary boundary paths [S1]. It addresses the challenge of costly silhouette edge identification in prior unbiased methods [S2, unofficial].

## Sources
[S1] Efficient Path-Space Differentiable Volume Rendering With Respect To Shapes, https://www.shuangz.com/projects/psdr-vol2-egsr23/psdr-vol2-egsr23.pdf, fetched 2026-10-27
[S2] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
