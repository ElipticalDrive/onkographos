# Volume Single Scattering (Monte Carlo with Sphere Light)


## Overview
This method computes single scattering radiance in participating media using Monte Carlo integration with a sphere light for background and emission. It differs from its parent, [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md), by replacing the point light source with a sphere light and employing equi-angular sampling for distance estimation.

## How it works
**Inputs and Outputs:**
- **Inputs:** Ray origin, ray direction, maximum ray distance ($t_{MAX}$), scene objects, volume properties (density, scattering coefficient), sphere light parameters (position, radius, emission color).
- **Outputs:** Radiance value for the pixel.

**Data Structures:**
- `Vector3`: 3D vector with components $x, y, z$ [S1, unofficial].
- `Color`: RGB color representation [S1, unofficial].
- `Ray`: Defined by origin and direction [S1, unofficial].
- `Object`: Base class for scene geometry [S1, unofficial].
- `Triangle`: Inherits from `Object`, stores vertices $A, B, C$, normal, distance, and color [S1, unofficial].

**Steps:**
1. **Calculate Background Radiance:** Compute the background radiance using the sphere light. This involves tracing a ray from the camera through the scene to determine what is visible behind the volume.
   - Call `getPixelColorWithSphereLight(objects, id, t_MAX, ray, hitPoint, 1, Importance_Solid_Angle)` where `id` is the current pixel index, `t_MAX` is the maximum ray distance, `ray` is the camera ray, `hitPoint` is the intersection point, and `Importance_Solid_Angle` is the sampling mode.
2. **Set Sampling Mode:** Set the 1D sampling mode to `EquiAngular`.
   - `sampleMode1D sampleMode = EquiAngular` [S1, unofficial].
3. **Monte Carlo Integration Loop:** For each sample:
   a. **Sample Distance:** Sample the distance $t_{sample}$ along the ray using the `EquiAngular` sampling mode.
   b. **Calculate Sample Point:** Compute the sample position $x_{sample} = ray.origin + t_{sample} \cdot ray.direction$.
   c. **Calculate Light Radiance ($L_e$):** Compute the radiance arriving at $x_{sample}$ from the sphere light.
      - Call `getPixelColorWithSphereLight(objects, -1, t_sample, ray, hitPoint, 1, Importance_Solid_Angle)` where `id` is -1 (indicating a light query), `t_sample` is the sampled distance, and other parameters are as above.
   d. **Calculate Transmittance from Light:** Compute the transmittance from the sample point to the light source.
      - `transmittanceFromLight = volume->getTransmittanceHomogeneos(x_sample, light.getLightPosition())` [S1, unofficial].
   e. **Accumulate Radiance:** Add the contribution to the total radiance. The contribution is the product of the light radiance, the transmittance from the light, and the volume's scattering properties.
4. **Average and Return:** Average the accumulated radiance over the number of samples and return the result.

**Formulas:**
- Background Radiance: $L_{bg} = \text{getPixelColorWithSphereLight}(\dots)$ [S1, unofficial].
- Sampled Distance: $t_{sample} = \text{sampleEquiAngular}()$ [S1, unofficial].
- Sample Point: $x_{sample} = o + t_{sample} \cdot d$ [G].
- Light Radiance: $L_e = \text{getPixelColorWithSphereLight}(\dots)$ [S1, unofficial].
- Transmittance from Light: $T(x_{sample}, L) = \text{getTransmittanceHomogeneos}(x_{sample}, L_{pos})$ [S1, unofficial].
- Total Radiance: $L = \frac{1}{N} \sum_{i=1}^{N} L_e(x_i) \cdot T(x_i, L) \cdot \sigma_s \cdot \Delta t$ [G].

**Parameters:**
- `sampleMode1D`: Set to `EquiAngular` [S1, unofficial].
- `Importance_Solid_Angle`: Sampling mode for sphere light [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision issues with small distances or large scene scales [G].
- Degenerate triangles (zero area) may cause intersection failures [S1, unofficial].
- Boundary conditions for ray-volume intersections [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def getRadiance(ray, objects, volume, sphere_light, num_samples):
    # Step 1: Calculate background radiance
    background = getPixelColorWithSphereLight(objects, 0, ray.t_max, ray, ray.hit_point, 1, Importance_Solid_Angle)
    
    # Step 2: Set sampling mode to EquiAngular
    sample_mode = EquiAngular
    
    total_radiance = Vector3(0, 0, 0)
    
    # Step 3: Monte Carlo loop
    for i in range(num_samples):
        # Step 3a: Sample distance using EquiAngular mode
        t_sample = sample_equi_angular(ray.t_max)
        
        # Step 3b: Calculate sample point
        x_sample = ray.origin + t_sample * ray.direction
        
        # Step 3c: Calculate Le using sphere light
        Le = getPixelColorWithSphereLight(objects, -1, t_sample, ray, ray.hit_point, 1, Importance_Solid_Angle)
        
        # Step 3d: Calculate transmittance from light
        transmittance_from_light = volume.get_transmittance_homogeneous(x_sample, sphere_light.get_light_position())
        
        # Step 3e: Accumulate radiance
        # Note: The exact scattering coefficient and step size are not provided in the source.
        # Assuming a simple accumulation for demonstration.
        contribution = Le * transmittance_from_light
        total_radiance += contribution
    
    # Step 4: Average and return
    average_radiance = total_radiance / num_samples
    return average_radiance

def sample_equi_angular(t_max):
    # Equi-angular sampling implementation
    # This is a placeholder for the actual sampling logic
    # Typically involves generating a random number and mapping it to the distance
    u = random.uniform(0, 1)
    # Example mapping (not from source)
    t_sample = t_max * u
    return t_sample
```

## Subtopics

### [Main Ray Marching with Frostbite Integration](4931-main-ray-marching-with-frostbite-integration.md)
A deterministic ray marching technique for computing single scattering, replacing stochastic sampling with fixed-step integration and using the "Frostbite method" for radiance accumulation.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of samples and $M$ is the cost of ray-object intersection and light query.
- **Memory Complexity:** $O(1)$ per sample, excluding scene storage.
- **Performance:** Depends on the number of samples and scene complexity. Higher sample counts reduce noise but increase computation time.
- **Strengths:** Handles complex lighting scenarios with sphere lights; Monte Carlo method provides unbiased results.
- **Limitations:** High variance with few samples; computationally expensive for high-quality images.
- **When to Use:** When accurate single scattering with sphere lights is required and computational resources allow.
- **When Not to Use:** For real-time applications or when point lights are sufficient.

## Background
This method is part of the VolumeSingleScattering project by WangYue, implemented in C++ [S1, unofficial]. It extends the basic Monte Carlo single scattering approach to support sphere lights, providing more realistic lighting in participating media.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
