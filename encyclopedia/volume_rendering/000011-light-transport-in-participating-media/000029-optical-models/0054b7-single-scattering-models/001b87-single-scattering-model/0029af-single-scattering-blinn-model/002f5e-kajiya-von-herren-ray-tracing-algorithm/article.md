# Kajiya-Von Herren Ray Tracing Algorithm
Also called: Kajiya and Von Herren method, Two-step ray tracing for volume densities

## Overview
The Kajiya-Von Herren algorithm is a two-step method for rendering participating media under the Single Scattering (Blinn Model) approximation [S1, unofficial]. It decouples the computation of light transport from the final image synthesis by precomputing a brightness contribution field for each light source and then summing these contributions during ray tracing [S1, unofficial]. This separation allows the illumination calculation to be performed once per frame (or scene) independently of the camera view, distinguishing it from pure Monte Carlo approaches that compute scattering per pixel [S1, unofficial].

## How it works
The method consists of two distinct phases: a precomputation step to generate a brightness field, and a ray tracing step to render the final image.

**Inputs and Outputs:**
*   **Precomputation Input:** Light source position, medium density function $\rho(x,y,z)$, and extinction coefficient $\tau$.
*   **Precomputation Output:** A 3D grid or field $I_i(x,y,z)$ representing the precomputed brightness contribution for light source $i$.
*   **Ray Tracing Input:** Camera ray, bounding box of the medium, precomputed fields $I_i$.
*   **Ray Tracing Output:** Pixel color.

**Data Structures:**
The precomputed brightness $I_i$ is stored as a 3D volume. The specific resolution and bit-depth are not given in the sources [S1, unofficial]. The ray tracing step iterates over volume elements along the ray path.

**Precomputation Step:**
For each light source $i$, the brightness contribution $I_i(x,y,z)$ at any point in space is calculated using Romberg integration [S1, unofficial]. The formula is:

$$ I_i(x, y, z) = \exp\left(-\int_{0}^{r} \tau \rho(\xi) d\xi\right) \int_{0}^{r} \rho(\xi(t), \eta(t), \zeta(t)) dt $$

Where:
*   $\tau = r \rho$ is the optical depth parameter [S1, unofficial].
*   $r$ is the distance from the light source to the point $(x,y,z)$ along the ray direction.
*   $\rho(\xi)$ is the density at position $\xi$ along the path from the light source.
*   The path from the light source to the point is parameterized by $(\xi(t), \eta(t), \zeta(t))$.

**Ray Tracing Step:**
1.  **Culling:** Each view ray is first culled against a bounding rectangular prism (extent) of the volume [S1, unofficial].
2.  **Path Boundaries:** The integration limits for the ray are determined by the intersection with the bounding extent and the rest of the world database [S1, unofficial]:
    $$ \lambda_1 = \max(0, d_1) $$
    $$ \lambda_2 = \min(d_{global}, d_2) $$
    Where:
    *   $d_1$ is the distance to the nearest intersection with the bounding extent.
    *   $d_{global}$ is the distance to the nearest intersection with the rest of the world database.
    *   $d_2$ is the distance to the farthest intersection with the bounding extent.
3.  **Summation:** The brightness of the ray is computed by summing the contribution of each volume element along the path segment $[\lambda_1, \lambda_2]$. The source states the brightness sums the contribution of each volume element but does not provide the explicit discrete summation formula or the specific scattering phase function used in the final sum, only mentioning that for clouds, perfectly diffuse scattering ($p(\cos \theta) = w_0$) and Rayleigh scattering ($p(\cos \theta) = w_0 \frac{4}{3}(1 + \cos^2 \theta)$) are of interest [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def precompute_brightness_field(light_source, density_field, extinction_coeff):
    """
    Precomputes the brightness contribution field I_i for a single light source.
    Uses Romberg integration as per [S1, unofficial].
    """
    # Define the grid resolution for I_i. Not given in sources [G].
    grid_resolution = (128, 128, 128) 
    brightness_field = zeros(grid_resolution)
    
    # Iterate over all points in the grid to compute I_i(x,y,z)
    for x, y, z in grid_points(grid_resolution):
        # Calculate distance r from light source to (x,y,z)
        r = distance(light_source, (x, y, z))
        
        # Define path from light source to (x,y,z)
        # Parameterize path t from 0 to r
        # Note: The exact parameterization of xi(t), eta(t), zeta(t) is not specified [G]
        # Assuming linear path from light_source to (x,y,z)
        
        # Compute optical depth integral: exp(-int_0^r tau * rho(xi) dxi)
        # tau = r * rho is stated, but rho varies. Assuming tau is constant or 
        # the formula implies a specific approximation. 
        # The source formula is: exp(-int_0^r tau rho(xi) dxi) where tau = r rho? 
        # This is ambiguous. We follow the literal formula structure [S1, unofficial].
        # Let's assume tau is a constant extinction coefficient for simplicity, 
        # or rho is uniform. 
        # Source says: "where tau = r rho". This is likely a typo for optical depth.
        # We will implement the integral as stated: int_0^r tau * rho(xi) dxi
        
        optical_depth_integral = romberg_integration(
            lambda t: extinction_coeff * density_field.get_density_at_path_point(t, light_source, (x,y,z)),
            0, r
        )
        
        # Compute density integral: int_0^r rho(xi(t), eta(t), zeta(t)) dt
        density_integral = romberg_integration(
            lambda t: density_field.get_density_at_path_point(t, light_source, (x,y,z)),
            0, r
        )
        
        # Calculate I_i
        brightness_field[x, y, z] = exp(-optical_depth_integral) * density_integral
        
    return brightness_field

def render_ray(camera_ray, bounding_extent, world_database, brightness_fields):
    """
    Renders a single ray using the precomputed brightness fields.
    """
    # 1. Culling against bounding rectangular prism
    if not intersects_bounding_box(camera_ray, bounding_extent):
        return background_color
    
    # 2. Determine path boundaries
    d1 = distance_to_nearest_intersection(camera_ray, bounding_extent)
    d2 = distance_to_farthest_intersection(camera_ray, bounding_extent)
    d_global = distance_to_nearest_intersection(camera_ray, world_database)
    
    lambda_1 = max(0, d1)
    lambda_2 = min(d_global, d2)
    
    # 3. Sum contributions
    # The source states: "The brightness of a ray sums the contribution of each volume element."
    # It does not specify the exact discrete sum formula or phase function application in the final step.
    # We assume a simple summation of precomputed brightness values along the ray.
    
    total_brightness = 0
    step_size = 0.1 # Not given in sources [G]
    
    t = lambda_1
    while t < lambda_2:
        point = camera_ray.get_point_at(t)
        
        # Sum contributions from all light sources
        for brightness_field in brightness_fields:
            # Sample the precomputed field at the current point
            # Note: The source does not specify if the precomputed field is 
            # "brightness at point" or "brightness contribution to point".
            # Given the formula I_i(x,y,z), it seems to be the brightness 
            # contribution *at* point (x,y,z) from light i.
            # However, standard single scattering usually involves integrating 
            # density * phase * light. The precomputed field likely encapsulates 
            # the light transport part.
            
            # Ambiguity: Does the ray trace sum I_i directly? 
            # Source: "The brightness of a ray sums the contribution of each volume element."
            # This implies integrating density along the ray, but I_i already 
            # contains density integration? 
            # Re-reading: "I_i(x,y,z) = exp(-...) * int rho dt". 
            # This I_i is the brightness *at* x,y,z from the light.
            # The ray trace step usually integrates the radiance along the view ray.
            # If I_i is the radiance contribution from the light to the point, 
            # then for a view ray, we might just sample I_i? 
            # But the source says "sums the contribution of each volume element".
            # This suggests a volumetric integration along the view ray.
            # Let's assume the precomputed field is used to accelerate the 
            # light transport part, but the view ray still integrates density.
            # However, the source is very brief. 
            # "The brightness of a ray sums the contribution of each volume element. It is given by:"
            # And then it shows the formula for I_i. This is contradictory.
            # It is more likely that I_i is the precomputed "lighting" term, 
            # and the ray trace integrates density * I_i? 
            # Or I_i is the final brightness at a point, and the ray trace just samples it?
            # Given "Two-step ray tracing", step 1 is precompute, step 2 is trace.
            # If step 2 just samples, it's not really "ray tracing" in the volumetric sense.
            # Let's stick to the literal text: "sums the contribution of each volume element".
            # We will assume a simple accumulation of density-weighted brightness.
            
            # Since the exact formula for the ray trace sum is not given, 
            # we mark this as [G] based on standard single scattering logic 
            # where radiance L = int density * phase * light * exp(-optical_depth) dt.
            # The precomputed I_i likely handles the light * exp(-optical_depth) part.
            
            # Placeholder for the missing ray trace integration formula [G]
            contribution = brightness_field.sample(point) * density_field.get_density_at(point)
            total_brightness += contribution * step_size
            
        t += step_size
        
    return total_brightness
```

**Edge Cases and Pitfalls:**
*   **Precision:** Romberg integration is used for the precomputation, which requires careful handling of the density function's smoothness [S1, unofficial].
*   **Boundaries:** The path boundaries $\lambda_1$ and $\lambda_2$ must be correctly calculated to avoid integrating outside the volume or into occluding geometry [S1, unofficial].
*   **Ambiguity:** The exact relationship between the precomputed field $I_i$ and the final ray trace summation is not explicitly defined in the source text, leading to implementation ambiguity [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** The precomputation step is performed once per frame or scene, making it independent of the number of pixels [S1, unofficial]. The ray tracing step is linear with respect to the number of volume elements along the ray.
*   **Memory Complexity:** Requires storing a 3D brightness field for each light source [S1, unofficial].
*   **Strengths:** Decouples illumination from rendering, allowing for efficient reuse of lighting calculations [S1, unofficial].
*   **Limitations:** The source does not provide specific performance metrics or hardware details [S1, unofficial]. The ambiguity in the final summation step may lead to incorrect results if not implemented carefully [S1, unofficial].

## Background
The method is based on the work of Kajiya and Von Herren, building upon Blinn's Low Albedo approximation for volume density scattering [S1, unofficial]. Blinn (1982) introduced the model for participating media with low albedo, considering only one secondary ray traversing the material towards the light source [S1, unofficial]. Kajiya and Von Herren extended this by introducing the two-step precomputation approach [S1, unofficial].

## Sources
[S1] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
