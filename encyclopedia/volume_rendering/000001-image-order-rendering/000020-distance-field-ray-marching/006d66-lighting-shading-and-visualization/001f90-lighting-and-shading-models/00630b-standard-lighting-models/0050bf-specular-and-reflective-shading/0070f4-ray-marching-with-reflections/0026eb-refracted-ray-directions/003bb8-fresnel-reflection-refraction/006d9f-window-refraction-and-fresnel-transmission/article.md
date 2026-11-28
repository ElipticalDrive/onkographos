# Window Refraction and Fresnel Transmission

Also called: Fresnel Transmission, Window Refraction

## Overview
This method models light transmission through a planar window (a medium with a refractive index between two parallel surfaces) in a ray-marching context. It calculates the refracted ray direction using Snell's law and applies Fresnel coefficients to determine the transmission loss, updating the ray's luminosity accordingly. It is distinct from simple transmission models by explicitly accounting for the angular dependence of reflectance at the interface boundaries.

## How it works
**Inputs:**
- Ray origins: `origins` (array of shape $(N, 3)$).
- Ray directions: `directions` (array of shape $(N, 3)$), assumed normalized.
- Incident luminosity: `incident_rays.luminosities` (array of shape $(N,)$).
- Window parameters: `left_interface_x_position`, `right_interface_x_position`, `n_ext`, `n_glass`, `diameter`, `eps`.
- Window normal: `window_normal` (typically $[1, 0, 0]$ for a window perpendicular to the x-axis).

**Outputs:**
- Refracted ray directions: `direction_refracted_rays` (array of shape $(N, 3)$).
- Updated luminosity: `output_rays.luminosities` (array of shape $(N,)$).

**Data Structures:**
The `Window` object holds the following fields:
- `left_interface_x_position`: float, x-coordinate of the left interface.
- `right_interface_x_position`: float, x-coordinate of the right interface.
- `n_ext`: float, refractive index of the external medium.
- `n_glass`: float, refractive index of the glass medium.
- `diameter`: float, diameter of the circular interfaces.
- `eps`: float, numerical stability parameter.
- `w2obj`: SimpleTransform object, coordinate transformation matrix.

**Parameters:**
- `n_ext`: Refractive index of the external medium. Typical value: 1.000293 [S1, unofficial].
- `n_glass`: Refractive index of the medium (glass). Typical value: 1.494 [S1, unofficial].
- `diameter`: Diameter of the interfaces. Typical value: 0.137 [S1, unofficial].
- `eps`: Parameter used for numerical stability in division operations. Default value: 1e-15 [S1, unofficial].

**Steps:**

1.  **Compute Intersection with Left Interface:**
    Calculate the intersection time $t$ with the left interface plane at $x = \text{left\_interface\_x\_position}$.
    $$ t = \frac{\text{left\_interface\_x\_position} - \text{origins}[:, 0]}{\text{directions}[:, 0] + \epsilon} $$
    where $\epsilon$ is a small value to prevent division by zero.

2.  **Compute Intersection Coordinates:**
    Calculate the $y$ and $z$ coordinates of the intersection point on the left interface.
    $$ y = \text{origins}[:, 1] + t \times \text{directions}[:, 1] $$
    $$ z = \text{origins}[:, 2] + t \times \text{directions}[:, 2] $$

3.  **Filter Invalid Intersections:**
    Check if the intersection is within the valid time range and inside the circular aperture.
    $$ \text{condition} = (t > 10^{-3}) \ \& \ (y^2 + z^2 < (\text{diameter} / 2)^2) $$
    If the condition is false, set $t$ to infinity to discard the ray.

4.  **Calculate Refracted Direction:**
    Use Snell's law in vector form to compute the refracted direction.
    Let $\mu = n_{ext} / n_{glass}$.
    Let $\mathbf{n}$ be the `window_normal`.
    Let $\mathbf{d}$ be the `directions`.
    $$ c = \mathbf{n} \cdot \mathbf{d} $$
    $$ \text{tmp} = 1 - \mu^2 (1 - c^2) $$
    $$ \text{direction\_refracted\_rays} = \sqrt{\text{tmp}} \times \mathbf{n} + \mu \times (\mathbf{d} - c \times \mathbf{n}) $$
    Note: $\sqrt{\text{tmp}}$ must be real; if $\text{tmp} < 0$, total internal reflection occurs, and transmission is zero.

5.  **Calculate Fresnel Transmission Coefficient:**
    Compute the reflectance for s-polarized ($r_s$) and p-polarized ($r_p$) light.
    Let $\theta_i$ be the angle of incidence such that $\cos(\theta_i) = c = \mathbf{n} \cdot \mathbf{d}$.
    Let $\sin(\theta_i) = \sqrt{1 - c^2}$.
    Let $n_1 = n_{ext}$ and $n_2 = n_{glass}$.
    $$ r_s = \left| \frac{n_1 \cos(\theta_i) - n_2 \sqrt{1 - (\frac{n_1}{n_2} \sin(\theta_i))^2}}{n_1 \cos(\theta_i) + n_2 \sqrt{1 - (\frac{n_1}{n_2} \sin(\theta_i))^2}} \right|^2 $$
    $$ r_p = \left| \frac{n_1 \sqrt{1 - (\frac{n_1}{n_2} \sin(\theta_i))^2} - n_2 \cos(\theta_i)}{n_1 \sqrt{1 - (\frac{n_1}{n_2} \sin(\theta_i))^2} + n_2 \cos(\theta_i)} \right|^2 $$
    The transmission coefficient is:
    $$ \text{trans\_coeff} = 1 - 0.5 \times (r_s + r_p) $$

6.  **Update Luminosity:**
    $$ \text{output\_rays.luminosities} = \text{incident\_rays.luminosities} \times \text{trans\_coeff} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def window_refraction_and_fresnel_transmission(
    origins,
    directions,
    incident_luminosities,
    left_interface_x_position,
    right_interface_x_position,
    n_ext,
    n_glass,
    diameter,
    eps=1e-15
):
    """
    Models refraction and Fresnel transmission through a planar window.
    
    Parameters:
    origins: (N, 3) array of ray origins.
    directions: (N, 3) array of ray directions (normalized).
    incident_luminosities: (N,) array of incident ray luminosities.
    left_interface_x_position: float, x-coordinate of the left interface.
    right_interface_x_position: float, x-coordinate of the right interface.
    n_ext: float, refractive index of the external medium.
    n_glass: float, refractive index of the glass medium.
    diameter: float, diameter of the circular interfaces.
    eps: float, numerical stability parameter.
    
    Returns:
    refracted_directions: (N, 3) array of refracted ray directions.
    output_luminosities: (N,) array of updated ray luminosities.
    """
    N = origins.shape[0]
    refracted_directions = np.zeros_like(directions)
    output_luminosities = np.zeros(N)
    
    # Step 1: Compute intersection time t with the left interface
    # t = (left_interface_x_position - origins[:, 0]) / (directions[:, 0] + eps)
    t = (left_interface_x_position - origins[:, 0]) / (directions[:, 0] + eps)
    
    # Step 2: Compute intersection coordinates y and z
    y = origins[:, 1] + t * directions[:, 1]
    z = origins[:, 2] + t * directions[:, 2]
    
    # Step 3: Filter invalid intersections
    # condition = (t > 1e-3) & ((y ** 2 + z ** 2) < (diameter / 2) ** 2)
    valid_mask = (t > 1e-3) & ((y ** 2 + z ** 2) < (diameter / 2) ** 2)
    
    # Initialize output arrays
    refracted_directions[:] = directions[:] # Default to incident direction if invalid
    output_luminosities[:] = 0.0
    
    # Only process valid rays
    if np.any(valid_mask):
        valid_t = t[valid_mask]
        valid_origins = origins[valid_mask]
        valid_directions = directions[valid_mask]
        valid_incident_luminosities = incident_luminosities[valid_mask]
        
        # Step 4: Calculate refracted direction using Snell's law
        # window_normal is assumed to be [1, 0, 0] for a window perpendicular to x-axis
        window_normal = np.array([1.0, 0.0, 0.0])
        
        mu = n_ext / n_glass
        
        # c = dot_product(window_normal, directions)
        c = np.sum(window_normal * valid_directions, axis=1)
        
        # tmp = 1 - mu ** 2 * (1 - c ** 2)
        tmp = 1.0 - mu ** 2 * (1.0 - c ** 2)
        
        # Handle total internal reflection where tmp < 0
        # In this case, transmission is 0, and direction might be undefined or reflected.
        # For transmission, we set direction to zero or keep incident, but luminosity becomes 0.
        # Here we compute sqrt(tmp) safely.
        sqrt_tmp = np.sqrt(np.maximum(tmp, 0.0))
        
        # direction_refracted_rays = sqrt(tmp) * window_normal + mu * (directions - c * window_normal)
        refracted_directions[valid_mask] = (
            sqrt_tmp[:, np.newaxis] * window_normal + 
            mu * (valid_directions - c[:, np.newaxis] * window_normal)
        )
        
        # Step 5: Calculate Fresnel transmission coefficient
        # n1 = n_ext, n2 = n_glass
        n1 = n_ext
        n2 = n_glass
        
        # cos(theta) = c
        cos_theta = c
        
        # sin(theta) = sqrt(1 - cos(theta)^2)
        sin_theta = np.sqrt(np.maximum(1.0 - cos_theta ** 2, 0.0))
        
        # term = n1/n2 * sin(theta)
        term = (n1 / n2) * sin_theta
        
        # sqrt(1 - term^2)
        sqrt_term = np.sqrt(np.maximum(1.0 - term ** 2, 0.0))
        
        # r_s = abs((n1*cos(theta) - n2*sqrt_term) / (n1*cos(theta) + n2*sqrt_term)) ** 2
        numerator_s = n1 * cos_theta - n2 * sqrt_term
        denominator_s = n1 * cos_theta + n2 * sqrt_term
        # Avoid division by zero
        denominator_s = np.where(np.abs(denominator_s) < eps, eps, denominator_s)
        r_s = (numerator_s / denominator_s) ** 2
        
        # r_p = abs((n1*sqrt_term - n2*cos(theta)) / (n1*sqrt_term + n2*cos(theta))) ** 2
        numerator_p = n1 * sqrt_term - n2 * cos_theta
        denominator_p = n1 * sqrt_term + n2 * cos_theta
        # Avoid division by zero
        denominator_p = np.where(np.abs(denominator_p) < eps, eps, denominator_p)
        r_p = (numerator_p / denominator_p) ** 2
        
        # Transmission = 1 - 0.5 * (r_s + r_p)
        trans_coeff = 1.0 - 0.5 * (r_s + r_p)
        
        # Step 6: Update ray luminosity
        output_luminosities[valid_mask] = valid_incident_luminosities * trans_coeff
        
    return refracted_directions, output_luminosities
```

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of rays. The operations are vectorized element-wise arithmetic.
- **Memory Complexity:** $O(N)$ for storing intermediate arrays (t, y, z, tmp, etc.).
- **Strengths:** Accurately models the angular dependence of transmission loss at dielectric interfaces. Simple to implement for planar windows.
- **Limitations:** Assumes a planar window with parallel surfaces. Does not account for multiple internal reflections within the glass (only the first transmission is modeled). Total internal reflection is handled by setting transmission to zero (implicitly via `sqrt(tmp)` clamping and subsequent luminosity update).
- **When to use:** When rendering scenes with planar glass windows or transparent planar barriers where Fresnel effects are important.
- **When not to use:** For curved glass surfaces, thick glass with significant internal scattering, or when multiple internal reflections are required.

## Background
This method is derived from the `gradoptics` library, which focuses on Gaussian optics and ray tracing for atom clouds and similar media. The specific implementation for window refraction and Fresnel transmission is found in the `gradoptics` repository [S1, unofficial]. It is used in simulations involving light propagation through optical elements like windows or lenses in atomic physics experiments.

## Sources
[S1] magis-slac/gradoptics, https://github.com/magis-slac/gradoptics, fetched 2026-10-27 (unofficial)
