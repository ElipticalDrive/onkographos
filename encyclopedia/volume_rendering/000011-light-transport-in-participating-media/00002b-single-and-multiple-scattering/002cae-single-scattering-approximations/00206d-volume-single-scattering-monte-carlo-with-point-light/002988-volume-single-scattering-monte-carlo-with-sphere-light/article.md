# Volume Single Scattering (Monte Carlo with Sphere Light)


## Overview
This method computes the single-scattering radiance for a participating medium using Monte Carlo integration, specifically designed for scenes with a sphere light (or environment map) acting as the background and emission source. It differs from the point-light variant by using `getPixelColorWithSphereLight` to sample the complex lighting environment via importance sampling of solid angles, rather than a direct geometric evaluation of a point source. It builds on [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by replacing the light evaluation step with a sphere-light-aware sampler.

## How it works
**Inputs:**
- `objects`: List of scene geometry.
- `ray`: View ray defined by origin $O$ and direction $D$.
- `t_MAX`: Maximum ray distance for background calculation.
- `volume`: Participating medium object with methods `getTransmittanceHomogeneos` and density/scattering coefficients.
- `light`: Light object with method `getLightPosition()`.

**Outputs:**
- `radiance`: The accumulated single-scattering radiance for the pixel.

**Data Structures:**
- `sampleMode1D`: Enum for sampling strategy. Value: `EquiAngular`.
- `hitPoint`: Intersection point on the background/sphere light.

**Steps:**
1.  **Background Calculation:** Calculate the background radiance $L_{bg}$ by tracing the view ray to the sphere light boundary.
    $$ L_{bg} = \text{getPixelColorWithSphereLight}(\text{objects}, \text{id}, t_{MAX}, \text{ray}, \text{hitPoint}, 1, \text{Importance\_Solid\_Angle}) $$
    *Note: `id` is typically the camera ID or pixel index. `Importance_Solid_Angle` indicates the sampling mode for the sphere light.* [S1, unofficial]

2.  **Set Sampling Mode:** Initialize the 1D sampler mode to `EquiAngular`. [S1, unofficial]

3.  **Monte Carlo Integration Loop:** Iterate over samples (typically one sample per pixel for single scattering in this specific implementation context, or multiple if variance reduction is added).
    a.  **Sample Distance:** Sample the distance $t_{sample}$ along the view ray using the `EquiAngular` mode. [S1, unofficial]
    b.  **Light Radiance ($L_e$):** Calculate the incident radiance from the sphere light at the scattering point $P = O + t_{sample} D$.
        $$ L_e = \text{getPixelColorWithSphereLight}(\text{objects}, -1, t_{sample}, \text{ray}, \text{hitPoint}, 1, \text{Importance\_Solid\_Angle}) $$
        *Note: The `id` parameter `-1` likely indicates a specific internal handling for light evaluation within the volume context.* [S1, unofficial]
    c.  **Transmittance from Light:** Calculate the transmittance from the scattering point $P$ to the light source.
        $$ T_{light} = \text{volume} \rightarrow \text{getTransmittanceHomogeneos}(P, \text{light.getLightPosition}()) $$
        *Note: This assumes a homogeneous medium for the light-to-point transmittance calculation.* [S1, unofficial]
    d.  **Accumulate Radiance:** The contribution is typically $L_e \cdot T_{light} \cdot \sigma_s \cdot p(\omega_i, \omega_o) \cdot \frac{1}{\text{pdf}}$. The specific accumulation formula is not explicitly detailed in the source notes beyond the component calculations, but the standard single scattering term is implied. [S1, unofficial]

4.  **Return:** Return the accumulated `radiance`. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def getPixelColor_VolumeSingleScatter_sphere2(objects, ray, t_MAX, volume, light):
    # 1. Calculate background using sphere light
    # hitPoint is determined by the intersection with the sphere light/environment boundary
    hitPoint = getSphereLightIntersection(ray, t_MAX)
    background = getPixelColorWithSphereLight(objects, id=0, t_MAX=t_MAX, ray=ray, hitPoint=hitPoint, samples=1, mode=Importance_Solid_Angle)
    
    # 2. Set sampling mode
    sampleMode1D = EquiAngular
    
    # 3. Monte Carlo Loop (Single sample implied by structure, or loop over N samples)
    radiance = Vector3(0, 0, 0)
    
    # Sample distance along the ray
    t_sample = sampleEquiAngular(sampleMode1D)
    
    # Calculate position of scattering point
    P = ray.origin + t_sample * ray.direction
    
    # Calculate Le using sphere light
    # Note: id=-1 suggests a specific internal path for light evaluation
    Le = getPixelColorWithSphereLight(objects, id=-1, t_MAX=t_sample, ray=ray, hitPoint=hitPoint, samples=1, mode=Importance_Solid_Angle)
    
    # Calculate transmittance from light
    # Assumes homogeneous medium for this specific call
    transmittanceFromLight = volume.getTransmittanceHomogeneos(P, light.getLightPosition())
    
    # Accumulate radiance
    # Standard single scattering term: Le * T_light * sigma_s * phase * (1/pdf)
    # The source notes do not explicitly provide the final multiplication formula, 
    # but imply these components are calculated for the contribution.
    # [G] Assuming standard single scattering accumulation:
    # radiance += Le * transmittanceFromLight * volume.sigma_s * volume.phase(cosTheta) / pdf
    
    return radiance
```

**Parameters:**
- `sampleMode1D`: `EquiAngular`. [S1, unofficial]
- `Importance_Solid_Angle`: Sampling mode for sphere light. [S1, unofficial]

**Edge Cases and Pitfalls:**
- `t_sample` must be within the volume bounds.
- `getTransmittanceHomogeneos` assumes homogeneity; if the medium is inhomogeneous, this approximation may introduce errors. [S1, unofficial]
- Precision issues with `getPixelColorWithSphereLight` if the sphere light is very small or very large.

## Subtopics

### Main Ray Marching with Frostbite Integration
This variant implements a ray marching approach through a volume, accumulating radiance using a physically based integration scheme from the Frostbite engine. It handles transmittance, inscatter, and early termination. It differs from the Monte Carlo sphere light method by using deterministic ray marching and a specific Frostbite integration formula rather than Monte Carlo sampling with a sphere light background. [S2, unofficial]

**Steps:**
1.  **Intersection:** Determine ray intersection with the bounding volume. If no intersection, return black. [S2, unofficial]
2.  **Initialization:** Initialize `radiance = 0` and `totalTransmittance` (passed by reference). Calculate phase function:
    $$ p(\omega_i, \omega_o) = \text{mix}(\text{HenyeyGreenstein}(-0.3, \cos\theta), \text{HenyeyGreenstein}(0.3, \cos\theta), 0.7) $$
    [S2, unofficial]
3.  **Density Check:** If `density > 0`:
    a.  Calculate sample scattering coefficient: $\sigma_s = \sigma_s \cdot \text{density}$. [S2, unofficial]
    b.  Calculate sample extinction coefficient: $\sigma_t = \sigma_t \cdot \text{density}$. [S2, unofficial]
    c.  Calculate emission (distance glow) and ambient light. [S2, unofficial]
    d.  Calculate inscatter:
        $$ L_{inscatter} = \text{emission} + \text{ambient} + \text{sunLight} \cdot p(\omega_i, \omega_o) \cdot L_{light}(\text{org}, P, \cos\theta) $$
        [S2, unofficial]
    e.  Scale inscatter by scattering coefficient and density:
        $$ L_{inscatter} \cdot= \sigma_s $$
        [S2, unofficial]
    f.  Calculate transmittance for the step:
        $$ T_{step} = \exp(-\sigma_t \cdot \text{stepS}) $$
        [S2, unofficial]
    g.  Integrate radiance using Frostbite method:
        $$ \text{radiance} += \text{totalTransmittance} \cdot \frac{L_{inscatter} - L_{inscatter} \cdot T_{step}}{\sigma_t} $$
        [S2, unofficial]
    h.  Update total transmittance:
        $$ \text{totalTransmittance} \cdot= T_{step} $$
        [S2, unofficial]
    i.  **Early Termination:** If `length(totalTransmittance) <= 1e-3`, set `totalTransmittance = 0` and return `radiance`. [S2, unofficial]
    j.  **Step:** Update position: $P += D \cdot \text{stepS}$. [S2, unofficial]

## Operating characteristics
- **Time Complexity:** Depends on the number of samples in the Monte Carlo loop and the cost of `getPixelColorWithSphereLight`. The Frostbite variant is $O(N)$ where $N$ is the number of steps along the ray.
- **Memory Complexity:** $O(1)$ for the Monte Carlo method (excluding scene data).
- **Performance:** The Monte Carlo method with sphere light is generally more accurate for complex lighting environments but can be noisy with few samples. The Frostbite variant is deterministic and faster but relies on approximations (homogeneous transmittance, specific phase function mix).
- **Strengths:** Handles complex environment lighting (sphere light).
- **Limitations:** Requires efficient sphere light sampling. Homogeneous assumption for light transmittance may be inaccurate for inhomogeneous media.

## Background
This method is derived from implementations in the `VolumeSingleScattering` repository by WangYue, which uses a sphere light for background and emission in single scattering calculations. The Frostbite variant is based on techniques described in the `volumetrics` repository by al-ro, referencing the Frostbite engine's approach to volumetric rendering.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
