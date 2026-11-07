# Beam Query × Beam Data, 2D Blur

Also called: Integral of Continuous Photon Beams, Direct integration of 2D blurred in-scattered radiance

## Overview
Beam Query × Beam Data, 2D Blur is a radiance estimation technique for participating media that computes the accumulated in-scattered radiance along an eye ray by querying photon beams and blurring their energy in two dimensions perpendicular to the beam direction [S1]. It solves the problem of estimating light transport in participating media without the bias introduced by blurring along the beam's length, which is inherent in the 3D blur variant [S1]. It differs from its parent, Point Query × Point Data, 3D Blur, by using continuous beam data instead of discrete point data and restricting the spatial blur to a 2D cross-section, thereby reducing bias for the same radius [S1]. It differs from the Beam × Beam 1D estimate by performing a 2D spatial blur rather than a 1D line blur, making it more general for volumetric scenes where the beam direction varies significantly relative to the view [S1, 2].

## How it works
The method estimates the beam radiance integral by substituting the 2D blurred in-scattered radiance estimate into the radiance transport equation and integrating along the camera ray [S1].

**Inputs:**
*   Photon beams $b$, each defined by origin $\mathbf{x}_b^*$, direction $\tilde{\omega}_b$, total flux $\Phi_b$, and scattering coefficient $\sigma_s$ of the medium.
*   Camera ray defined by origin $\mathbf{x}_c$, direction $\tilde{\omega}_c$, and integration limit $s$.
*   Phase function $f(\theta_b)$, where $\theta_b$ is the angle between $\tilde{\omega}_c$ and $\tilde{\omega}_b$.
*   Extinction coefficient $\sigma_t$ (assumed constant for the analytical simplification).
*   Cross-sectional area measure $\mu_R(r^2)$ of the photon beam's blur kernel.

**Outputs:**
*   Estimated radiance $L_b(\mathbf{x}_c \leftarrow \vec{\omega}_c, s)$ along the camera ray.

**Data Structures:**
*   Photon beams are stored as a list or spatially accelerated structure (e.g., BVH, grid) allowing retrieval of all beams $b \in R_b$ that intersect the camera ray's 2D blur region.
*   Each beam $b$ stores: $\mathbf{x}_b^*$, $\tilde{\omega}_b$, $\Phi_b$.

**Steps:**
1.  **Beam Selection:** Identify all photon beams $b$ whose 2D cylindrical blur regions intersect the camera ray segment $[0, s]$. Let this set be $R_b$.
2.  **Intersection Bounds:** For each beam $b \in R_b$, calculate the intersection distances $t_c^-$ and $t_c^+$ where the camera ray enters and exits the beam's 2D blur region.
3.  **Integral Calculation:** For each beam $b$, compute the integral of the product of the camera ray transmittance and the photon beam transmittance over the interval $[t_c^-, t_c^+]$.
    *   The transmittance along the camera ray is $e^{-\sigma_t t_c}$.
    *   The transmittance along the photon beam depends on the position $t_c$. For constant cross-section beams in homogeneous media, the distance along the photon beam $t_b$ is related to $t_c$ by:
        $$ t_b = t_b^- - |t_c - t_c^-| |\cos \theta_b| $$
        where $t_b^-$ is the scalar projection of the camera ray start point onto the beam, and $\cos \theta_b = \tilde{\omega}_c \cdot \tilde{\omega}_b$.
    *   The integral is:
        $$ I_b = \int_{t_c^-}^{t_c^+} e^{-\sigma_t t_c} e^{-\sigma_t t_b} dt_c $$
4.  **Summation:** Sum the contributions from all beams and multiply by the scattering coefficient and phase function:
    $$ L_b(\mathbf{x}_c \leftarrow \vec{\omega}_c, s) \approx \frac{\sigma_s}{\mu_R(r^2)} \sum_{b \in R_b} f(\theta_b) \Phi_b I_b $$

**Formulas:**
The general estimate is:
$$ L_b(\mathbf{x}_c \leftarrow \vec{\omega}_c, s) \approx \frac{\sigma_s}{\mu_R(r^2)} \sum_{b \in R_b} f(\theta_b) \Phi_b \int_{t_c^-}^{t_c^+} e^{-\sigma_{t_c}} e^{-\sigma_{t_b}} dt_c \quad [S1] $$

For homogeneous media with constant cross-section beams, the integral $I_b$ simplifies to:
$$ I_b = \frac{e^{-\sigma_t (t_c - t_c^+)} (|\cos \theta_b| - 1) - 1}{e^{\sigma_t (t_c + t_b^-)} \sigma_t (|\cos \theta_b| - 1)} \quad [S1] $$
*Note: The source formula for the simplified integral contains a typo in the exponent of the denominator ($t_c + t_b^-$ should likely be related to the bounds, but the structure is as cited in the notes and source text Eq 30).*

**Parameters:**
*   $\sigma_s$: Scattering coefficient. Typical value: Scene-dependent [S1].
*   $\sigma_t$: Extinction coefficient. Typical value: Scene-dependent [S1].
*   $\mu_R(r^2)$: 2D area measure of the beam cross-section. For a circular kernel of radius $r$, $\mu_R(r^2) = \pi r^2$ [S1].

**Edge Cases and Pitfalls:**
*   **Heterogeneous Media:** The analytical simplification of the integral is not available. The integral must be computed numerically or the estimate is considered impractical for heterogeneous media [S1].
*   **Varying Cross-Section:** If the beam cross-section varies (e.g., conical beams), $\mu_R(r^2)$ becomes a function of $t_b$ and must be moved inside the integral, complicating the calculation [S1].
*   **Parallel Beams:** If $\cos \theta_b = 1$ (camera ray parallel to beam), the denominator in the simplified integral becomes zero. This requires a limit evaluation or numerical integration [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def beam_query_beam_data_2d_blur(camera_ray, photon_beams, sigma_s, sigma_t, r):
    """
    Estimates radiance along a camera ray using 2D blurred photon beams.
    
    Args:
        camera_ray: Object with origin x_c, direction w_c, and length s.
        photon_beams: List of photon beam objects.
        sigma_s: Scattering coefficient.
        sigma_t: Extinction coefficient.
        r: Radius of the 2D blur kernel.
        
    Returns:
        Estimated radiance L_b.
    """
    mu_R = math.pi * r * r  # 2D area measure for circular kernel
    total_radiance = 0.0
    
    # 1. Beam Selection and Intersection Bounds
    # In practice, use a spatial acceleration structure to find intersecting beams.
    # Here we iterate all for clarity.
    for beam in photon_beams:
        # Calculate intersection of camera ray with beam's 2D cylindrical region
        # The region is a cylinder of radius r centered on beam origin x_b* and direction w_b.
        # This involves solving for t_c where distance from camera ray point to beam axis <= r.
        # Let t_c_minus and t_c_plus be the entry and exit points.
        # If no intersection, continue.
        
        # Simplified intersection logic (assuming infinite cylinder for derivation context):
        # Vector from camera origin to beam origin
        d = beam.x_b_star - camera_ray.x_c
        # Project d onto beam direction to find closest point on beam axis
        t_b_proj = dot(d, beam.w_b)
        # Vector from camera ray point at t_c to beam axis
        # Distance squared function in t_c is quadratic.
        # Solve for t_c where dist^2 = r^2.
        # This yields t_c_minus and t_c_plus.
        # If discriminant < 0, no intersection.
        
        # For the purpose of this pseudocode, assume we have valid t_c_minus and t_c_plus
        # and that the beam intersects the ray segment [0, s].
        if t_c_plus < 0 or t_c_minus > camera_ray.s:
            continue
            
        t_c_start = max(0.0, t_c_minus)
        t_c_end = min(camera_ray.s, t_c_plus)
        
        if t_c_start >= t_c_end:
            continue
            
        # 2. Integral Calculation
        # Calculate cos(theta_b)
        cos_theta_b = dot(camera_ray.w_c, beam.w_b)
        abs_cos_theta_b = abs(cos_theta_b)
        
        # Calculate t_b_minus: scalar projection of camera ray start point onto beam
        # t_b_minus is the distance along the beam from beam origin to the point 
        # closest to the camera ray origin.
        # Actually, from source: t_b^- is the scalar projection of the integration start point t_c^-
        # expressed as a distance along the photon beam.
        # Let's use the source's definition: t_b^- corresponds to t_c^-
        # t_b(t_c) = t_b^- - |t_c - t_c^-| * |cos(theta_b)|
        # We need t_b_minus corresponding to t_c_start.
        
        # Vector from beam origin to camera ray point at t_c_start
        vec_to_cam = (camera_ray.x_c + camera_ray.w_c * t_c_start) - beam.x_b_star
        t_b_minus = dot(vec_to_cam, beam.w_b)
        
        # Compute the integral I_b analytically for homogeneous media
        # Formula from source Eq 30
        # Numerator: exp(-sigma_t * (t_c - t_c_plus)) * (|cos| - 1) - 1
        # Denominator: exp(sigma_t * (t_c + t_b_minus)) * sigma_t * (|cos| - 1)
        # Note: The source formula has a potential typo in the denominator exponent.
        # We implement the formula exactly as given in the source notes.
        
        denom_term = abs_cos_theta_b - 1.0
        if abs(denom_term) < 1e-6:
            # Parallel case, use numerical integration or limit
            # For simplicity, skip or use a small epsilon
            denom_term = 1e-6
            
        num = math.exp(-sigma_t * (t_c_start - t_c_end)) * denom_term - 1.0
        den = math.exp(sigma_t * (t_c_start + t_b_minus)) * sigma_t * denom_term
        
        if abs(den) < 1e-10:
            integral_val = 0.0
        else:
            integral_val = num / den
            
        # 3. Summation
        phase = phase_function(cos_theta_b)
        contribution = (sigma_s / mu_R) * phase * beam.flux * integral_val
        total_radiance += contribution
        
    return total_radiance
```

## Subtopics

### [Beam × Beam 1D estimate](4776-beam-beam-1d-estimate.md)
The Beam × Beam 1D estimate is a radiance estimation technique for participating media that uses photon beams to avoid ray marching, offering better image quality than photon points while being computationally simpler than the 3D variant. It differs from its siblings by performing spatial blur over a line (1D kernel) rather than a volume or area, which reduces the temporal function to a Dirac delta, making it the most practical implementation among the Beam × Beam family. It is chosen when computational efficiency is paramount and the scene geometry allows for 1D approximation along the beam direction.

### Beam Query × Beam Data, 3D Blur
A beam radiance estimate that directly computes the beam radiance integral along a camera ray by substituting the in-scattered radiance estimate (Equation 21) into the beam radiance integral (Equation 8), resulting in a double integral [S1]. It is fairly complicated due to the integral of a 3D convolution and is impractical for heterogeneous media, included mainly for completeness [S1]. It is used when a full 3D blur is required for theoretical completeness or specific homogeneous scenarios where analytical bounds can be derived.

## Operating characteristics
*   **Time Complexity:** Linear in the number of photon beams $N_b$ that intersect the camera ray's blur region, assuming efficient spatial querying. The integral evaluation is $O(1)$ per beam for homogeneous media.
*   **Memory Complexity:** Linear in the number of photon beams stored.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the density of photon beams and the complexity of the scene's extinction coefficient.
*   **Strengths:** Simpler evaluation than 3D blur (no integration along beam length) and less bias for the same radius because no blurring is performed along the beam [S1]. It is a limit case of anisotropic density estimation [S1].
*   **Limitations:** Impractical for heterogeneous media due to the complexity of the integral [S1]. Requires careful handling of parallel beams and varying cross-sections.
*   **When to use:** Homogeneous media where bias reduction along the beam direction is important and computational resources allow for beam-based querying.
*   **When not to use:** Heterogeneous media, or when extreme computational efficiency is required (consider 1D variant or point-based methods).

## Background
The method was introduced in "A comprehensive theory of volumetric radiance estimation using photon points and beams" [S1]. It is derived by taking the limit of discrete photon marching while shrinking the cylindrical kernel's height to the step size, reducing bias [S1]. It is related to surface radiance estimates developed by Herzog et al. [2007] and can be seen as a generalization of their photon ray splatting estimate to participating media [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
