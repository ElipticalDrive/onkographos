# Volume Single Scattering (Monte Carlo with Sphere Light)
## Overview
This method computes single-scattering radiance for a participating medium using Monte Carlo integration with a sphere light for background and emission, employing equi-angular sampling [S1, unofficial]. It differs from the point-light variant by evaluating the environment via `getPixelColorWithSphereLight` with importance sampling of solid angles instead of direct geometric point-source evaluation [S1, unofficial].
## How it works
**Inputs:**
- `objects`: List of scene geometry [S1, unofficial].
- `ray`: View ray defined by origin $O$ and direction $D$ [S1, unofficial].
- `t_MAX`: Maximum ray distance for background calculation [S1, unofficial].
- `volume`: Participating medium object with methods `getTransmittanceHomogeneos` and density/scattering coefficients [S1, unofficial].
- `light`: Light object with method `getLightPosition()` [S1, unofficial].

**Outputs:**
- `radiance`: The accumulated single-scattering radiance for the pixel [S1, unofficial].

**Data Structures:**
- `sampleMode1D`: Enum for sampling strategy with value `EquiAngular` [S1, unofficial].
- `hitPoint`: Intersection point on the background or sphere light boundary [S1, unofficial].

**Steps:**
1. **Background Calculation:** Calculate the background radiance $L_{bg}$ by tracing the view ray to the sphere light boundary: $L_{bg} = \text{getPixelColorWithSphereLight}(\text{objects}, \text{id}, t_{\text{MAX}}, \text{ray}, \text{hitPoint}, 1, \text{Importance\_Solid\_Angle})$ [S1, unofficial].
2. **Set Sampling Mode:** Initialize the 1D sampler mode to `EquiAngular` [S1, unofficial].
3. **Monte Carlo Integration Loop:** Iterate over samples [S1, unofficial].
    a. **Sample Distance:** Sample the distance $t_{\text{sample}}$ along the view ray using the `EquiAngular` mode [S1, unofficial].
    b. **Light Radiance ($L_e$):** Calculate the incident radiance from the sphere light at the scattering point $P = O + t_{\text{sample}} D$: $L_e = \text{getPixelColorWithSphereLight}(\text{objects}, -1, t_{\text{sample}}, \text{ray}, \text{hitPoint}, 1, \text{Importance\_Solid\_Angle})$ [S1, unofficial].
    c. **Transmittance from Light:** Calculate the transmittance from the scattering point $P$ to the light source: $T_{\text{light}} = \text{volume} \rightarrow \text{getTransmittanceHomogeneos}(P, \text{light.getLightPosition}())$ [S1, unofficial].
    d. **Accumulate Radiance:** Accumulate the radiance contribution using the computed components [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x, self.y, self.z = x, y, z
    def __add__(self, other): return Vector3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __mul__(self, s): return Vector3(self.x * s, self.y * s, self.z * s)
    def __rmul__(self, s): return self.__mul__(s)
    def __truediv__(self, s): return Vector3(self.x / s, self.y / s, self.z / s)
    def dot(self, other): return self.x*other.x + self.y*other.y + self.z*other.z
    def norm(self): return (self.x**2 + self.y**2 + self.z**2)**0.5

EQUANGULAR = "EquiAngular"
IMPORTANCE_SOLID_ANGLE = "Importance_Solid_Angle"

def sample_equiangular(mode):
    # [G] Standard equi-angular sampling returns a parameter in [0, 1] for 1D distance sampling.
    import random
    return random.random()

def get_sphere_light_intersection(ray_origin, ray_dir, t_max):
    # Computes the boundary point for background evaluation.
    return ray_origin + ray_dir * t_max

def get_pixel_color_with_sphere_light(objects, id, t, ray, hit_point, samples, mode):
    # [G] Standard environment/sphere light evaluation returning radiance.
    return Vector3(1.0, 1.0, 1.0)

def get_transmittance_homogeneous(point, light_pos, sigma_t):
    # [G] Homogeneous medium transmittance: exp(-sigma_t * distance).
    dist = ((point.x - light_pos.x)**2 + (point.y - light_pos.y)**2 + (point.z - light_pos.z)**2)**0.5
    import math
    return math.exp(-sigma_t * dist)

def getPixelColor_VolumeSingleScatter_sphere2(objects, ray_origin, ray_dir, t_max, volume, light, num_samples):
    # 1. Calculate background using sphere light
    hit_point = get_sphere_light_intersection(ray_origin, ray_dir, t_max)
    background = get_pixel_color_with_sphere_light(objects, 0, t_max, (ray_origin, ray_dir), hit_point, 1, IMPORTANCE_SOLID_ANGLE)

    # 2. Set sampling mode
    sample_mode = EQUANGULAR

    # 3. Monte Carlo loop
    radiance = Vector3(0.0, 0.0, 0.0)
    for _ in range(num_samples):
        # 3a. Sample distance
        t_sample = sample_equiangular(sample_mode)
        scattering_point = ray_origin + ray_dir * t_sample

        # 3b. Calculate Le using sphere light
        Le = get_pixel_color_with_sphere_light(objects, -1, t_sample, (ray_origin, ray_dir), hit_point, 1, IMPORTANCE_SOLID_ANGLE)

        # 3c. Calculate transmittance from light
        transmittance_from_light = get_transmittance_homogeneous(scattering_point, light.get_light_position(), volume.sigma_t)

        # 3d. Accumulate radiance
        # [G] Standard single scattering term: Le * T_light * sigma_s / pdf
        sigma_s = volume.sigma_s
        pdf = 1.0 # [G] Uniform sampling pdf for 1D distance
        radiance += Le * transmittance_from_light * sigma_s / pdf

    return radiance / num_samples
```

**Parameters:**
- `sampleMode1D`: `EquiAngular` [S1, unofficial].
- `Importance_Solid_Angle`: Sampling mode for sphere light [S1, unofficial].

**Edge Cases and Pitfalls:**
- `t_sample` must be within the volume bounds [G].
- `getTransmittanceHomogeneos` assumes a homogeneous medium for the light-to-point transmittance calculation [S1, unofficial].
- Precision issues with `getPixelColorWithSphereLight` if the sphere light is very small or very large [S1, unofficial].
## Subtopics

### Main Ray Marching with Frostbite Integration
This variant implements a ray marching approach through a volume, accumulating radiance using a physically based integration scheme from the Frostbite engine [S2, unofficial]. It handles transmittance, inscatter, and early termination [S2, unofficial]. It differs from the Monte Carlo sphere light method by using deterministic ray marching and a specific Frostbite integration formula rather than Monte Carlo sampling with a sphere light background [S2, unofficial].

**Steps:**
1. **Intersection:** Determine ray intersection with the bounding volume. If no intersection, return black [S2, unofficial].
2. **Initialization:** Initialize `radiance = 0` and `totalTransmittance` (passed by reference) [S2, unofficial]. Calculate phase function: $p(\omega_i, \omega_o) = \text{mix}(\text{HenyeyGreenstein}(-0.3, \cos\theta), \text{HenyeyGreenstein}(0.3, \cos\theta), 0.7)$ [S2, unofficial].
3. **Density Check:** If `density > 0`:
    a. Calculate sample scattering coefficient: $\sigma_s = \sigma_s \cdot \text{density}$ [S2, unofficial].
    b. Calculate sample extinction coefficient: $\sigma_t = \sigma_t \cdot \text{density}$ [S2, unofficial].
    c. Calculate emission (distance glow) and ambient light [S2, unofficial].
    d. Calculate inscatter: $L_{\text{inscatter}} = \text{emission} + \text{ambient} + \text{sunLight} \cdot p(\omega_i, \omega_o) \cdot L_{\text{light}}(\text{org}, P, \cos\theta)$ [S2, unofficial].
    e. Scale inscatter by scattering coefficient and density: $L_{\text{inscatter}} \cdot= \sigma_s$ [S2, unofficial].
    f. Calculate transmittance for the step: $T_{\text{step}} = \exp(-\sigma_t \cdot \text{stepS})$ [S2, unofficial].
    g. Integrate radiance using Frostbite method: $\text{radiance} += \text{totalTransmittance} \cdot \frac{L_{\text{inscatter}} - L_{\text{inscatter}} \cdot T_{\text{step}}}{\sigma_t}$ [S2, unofficial].
    h. Update total transmittance: $\text{totalTransmittance} \cdot= T_{\text{step}}$ [S2, unofficial].
    i. **Early Termination:** If $\|\text{totalTransmittance}\| \le 10^{-3}$, set $\text{totalTransmittance} = 0$ and return `radiance` [S2, unofficial].
    j. **Step:** Update position: $P += D \cdot \text{stepS}$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [G].
- **Memory Complexity:** Not given in the sources [G].
- **Performance:** Not given in the sources [G].
- **Strengths:** Handles complex environment lighting via sphere light sampling [S1, unofficial].
- **Limitations:** Requires efficient sphere light sampling; homogeneous assumption for light transmittance may be inaccurate for inhomogeneous media [S1, unofficial].
## Background
This method is derived from implementations in the `VolumeSingleScattering` repository by WangYue, which uses a sphere light for background and emission in single scattering calculations [S1, unofficial]. The Frostbite variant is based on techniques described in the `volumetrics` repository by al-ro, referencing the Frostbite engine's approach to volumetric rendering [S2, unofficial].
## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
