# Sphere Light Sampling with Multiple Strategies

Also called: getPixelColorWithSphereLightDBG

## Overview
Calculates direct lighting from a spherical light source using Monte Carlo integration with various sampling strategies, including Importance Solid Angle, Importance Surface Area, Uniform Solid Angle, and others [S1, unofficial]. It sets itself apart from siblings like [Dwivedi Sampling](5537-dwivedi-sampling.md) by explicitly handling the geometry of a spherical light source through multiple distinct probability density functions (PDFs) rather than a single fixed approach, allowing the renderer to choose the most efficient estimator for the light's position relative to the shading point [S1, unofficial].

## How it works
**Inputs:**
*   Shading point $\mathbf{X}$ (position in world space) [G].
*   Surface normal $\mathbf{n}$ at $\mathbf{X}$ [G].
*   Light source: a sphere defined by center $\mathbf{C}$, radius $R$, and emitted radiance $L_i$ [S1, unofficial].
*   Sampling mode $\mathcal{M}$: an enum selecting the strategy (`Importance_Solid_Angle`, `Importance_Surface_Area`, `Uniform_Solid_Angle`, `Uniform_Hemisphere_Solid_Angle`, `Importance_Phong`, `Cosine_Weighted`) [S1, unofficial].
*   Number of samples $N$ [G].

**Outputs:**
*   Estimated direct radiance $L_r$ at $\mathbf{X}$ from the spherical light [S1, unofficial].

**Data Structures:**
*   `Vector3`: 3D floating-point vector $(x, y, z)$ with standard arithmetic and dot/cross products [G].
*   `Color`: 3D floating-point vector representing RGB radiance [G].
*   `Ray`: Origin $\mathbf{o}$, direction $\mathbf{d}$ [G].
*   `Sphere`: Center $\mathbf{C}$, radius $R$, material properties (exponent for Phong, material type for BRDF) [S1, unofficial].

**Steps:**

1.  **Calculate Geometric Limits:**
    Compute the distance $d$ from $\mathbf{X}$ to the light center $\mathbf{C}$:
    $$ d = \| \mathbf{X} - \mathbf{C} \| $$
    Compute the maximum sine and cosine of the half-angle $\theta_{max}$ subtended by the light sphere:
    $$ \sin\theta_{max} = \frac{R}{d} $$
    $$ \cos\theta_{max} = \sqrt{1 - \sin^2\theta_{max}} $$
    [S1, unofficial]

2.  **Sample Direction:**
    Generate a sample direction $\mathbf{l}$ towards the light sphere based on the selected mode $\mathcal{M}$ and $\sin\theta_{max}$ [S1, unofficial].
    *   For `Importance_Solid_Angle`, `Importance_Surface_Area`, `Importance_Phong`, and `Cosine_Weighted`, the direction is constrained within the cone defined by $\theta_{max}$ [G].
    *   For `Uniform_Solid_Angle` and `Uniform_Hemisphere_Solid_Angle`, the direction is sampled over the full sphere or hemisphere, respectively [G].

3.  **Trace Ray to Light:**
    Construct a ray $\mathbf{r}$ from $\mathbf{X}$ in direction $\mathbf{l}$ [G]. Find the intersection distance $t$ to the light sphere [S1, unofficial]. If $t > \epsilon$ (where $\epsilon$ is a small epsilon, e.g., $10^{-9}$), the ray hits the light [S1, unofficial]. If no hit or $t \le \epsilon$, the light is not visible in this sample direction [G].

4.  **Shadow Check:**
    Trace a shadow ray from $\mathbf{X}$ (offset by $\epsilon \mathbf{n}$ to prevent self-intersection) to the intersection point on the light sphere $\mathbf{P}_{light} = \mathbf{X} + t \mathbf{l}$ [S1, unofficial]. Check for intersections with other objects in the scene [G]. Let $t_{near}$ be the distance to the first intersection [G]. If an intersection exists such that $t_{near}^2 < d^2$ (or simply $t_{near} < t$), the point is in shadow ($inShadow = true$) [S1, unofficial]. Otherwise, $inShadow = false$ [G].

5.  **Compute Shading Terms:**
    *   Light direction: $\mathbf{l}_{dir} = \mathbf{P}_{light} - \mathbf{X}$ (normalized) [G].
    *   Incident angle cosine: $\cos\theta_i = \max(0, \mathbf{l}_{dir} \cdot \mathbf{n})$ [G].
    *   Light surface normal: $\mathbf{n}_{light} = \mathbf{P}_{light} - \mathbf{C}$ (normalized) [G].
    *   Outgoing angle cosine (for BSDF): $\cos\theta_o = \max(0, -\mathbf{n}_{light} \cdot \mathbf{l}_{dir})$ [S1, unofficial].

6.  **Evaluate BRDF:**
    *   If Material is `DIFFUSE`:
        $$ f_r = \frac{1}{\pi} $$
        [S1, unofficial]
    *   If Material is `SPECULAR` (Phong):
        Let $\mathbf{v}$ be the view direction (from $\mathbf{X}$ to camera, or $-\mathbf{l}_{dir}$ if evaluating for reflection) [G].
        Let $\text{temp} = \mathbf{v} \cdot \mathbf{l}_{dir}$ [S1, unofficial].
        $$ f_r = \text{pow}(\max(0, \text{temp}), \text{exponent}) \cdot \frac{\text{exponent} + 2}{2\pi} $$
        [S1, unofficial]

7.  **Accumulate Radiance:**
    Initialize $L_r = 0$. For each of the $N$ samples:
    *   Determine the contribution based on the mode [S1, unofficial]:
        *   **Importance_Solid_Angle:**
            $$ p(\mathbf{l}) = 2\pi (1 - \cos\theta_{max}) $$
            $$ L_r \leftarrow L_r + L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot p(\mathbf{l}) $$
            [S1, unofficial]
        *   **Importance_Surface_Area:**
            $$ \text{Area} = \frac{2\pi (1 - \sin\theta_{max}) R^2}{d^2} $$
            $$ L_r \leftarrow L_r + L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot \cos\theta_o \cdot \text{Area} \cdot f_r $$
            [S1, unofficial]
        *   **Uniform_Solid_Angle:**
            $$ L_r \leftarrow L_r + f_r \cdot L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot 4\pi $$
            [S1, unofficial]
        *   **Uniform_Hemisphere_Solid_Angle:**
            $$ L_r \leftarrow L_r + f_r \cdot L_i \cdot (1 - inShadow) \cdot \cos\theta_i \cdot 2\pi $$
            [S1, unofficial]
        *   **Importance_Phong:**
            $$ L_r \leftarrow L_r + L_i \cdot (1 - inShadow) \cdot \cos\theta_i $$
            [S1, unofficial]
        *   **Cosine_Weighted:**
            $$ L_r \leftarrow L_r + L_i \cdot (1 - inShadow) \cdot f_r $$
            [S1, unofficial]

8.  **Average:**
    $$ L_r = \frac{1}{N} \sum_{k=1}^{N} L_{sample, k} $$
    [S1, unofficial]

**Pitfalls:**
*   **Self-Intersection:** The shadow ray origin must be offset by $\epsilon \mathbf{n}$ (e.g., $10^{-9}$) along the normal to prevent the ray from intersecting the surface it originated from [S1, unofficial].
*   **Precision:** When $\mathbf{X}$ is very close to the light center, $d \to 0$, causing division by zero or numerical instability in $\sin\theta_{max}$. Guard against $d < \epsilon$ [G].
*   **Shadow Ray Bias:** Using a fixed epsilon for shadow ray offset can cause acne or peter-panning if not scaled appropriately [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

class Vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, o): return Vector3(self.x+o.x, self.y+o.y, self.z+o.z)
    def __sub__(self, o): return Vector3(self.x-o.x, self.y-o.y, self.z-o.z)
    def __mul__(self, s): return Vector3(self.x*s, self.y*s, self.z*s)
    def dot(self, o): return self.x*o.x + self.y*o.y + self.z*o.z
    def cross(self, o): return Vector3(self.y*o.z-self.z*o.y, self.z*o.x-self.x*o.z, self.x*o.y-self.y*o.x)
    def norm(self): return math.sqrt(self.x**2 + self.y**2 + self.z**2)
    def normalize(self): n = self.norm(); return Vector3(self.x/n, self.y/n, self.z/n) if n > 1e-9 else Vector3(0,0,0)

class Color:
    def __init__(self, r, g, b): self.r, self.g, self.b = r, g, b
    def __add__(self, o): return Color(self.r+o.r, self.g+o.g, self.b+o.b)
    def __mul__(self, s): return Color(self.r*s, self.g*s, self.b*s)
    def __truediv__(self, s): return Color(self.r/s, self.g/s, self.b/s)

class Ray:
    def __init__(self, origin, direction): self.origin, self.direction = origin, direction

class Sphere:
    def __init__(self, center, radius, radiance, material, exponent):
        self.center, self.radius, self.radiance, self.material, self.exponent = center, radius, radiance, material, exponent
    def find_intersection(self, ray):
        oc = ray.origin - self.center
        b = oc.dot(ray.direction)
        c = oc.dot(oc) - self.radius**2
        disc = b**2 - c
        if disc < 0: return -1.0
        t = -b - math.sqrt(disc)
        return t if t > 1e-9 else -1.0

class SampleMode:
    IMPORTANCE_SOLID_ANGLE = 0
    IMPORTANCE_SURFACE_AREA = 1
    UNIFORM_SOLID_ANGLE = 2
    UNIFORM_HEMISPHERE_SOLID_ANGLE = 3
    IMPORTANCE_PHONG = 4
    COSINE_WEIGHTED = 5

class Material:
    DIFFUSE = 0
    SPECULAR = 1

def create_coordinate_system(normal):
    if abs(normal.x) > 0.9: aux = Vector3(1, 0, 0)
    else: aux = Vector3(0, 1, 0)
    V = aux.cross(normal).normalize()
    W = normal.cross(V).normalize()
    return V, W

def get_sample_uniform_solid_angle(X, normal):
    u1, u2 = random.random(), random.random()
    theta = 2.0 * math.pi * u1
    phi = math.acos(2.0 * u2 - 1.0)
    sample = Vector3(math.sin(phi)*math.cos(theta), math.sin(phi)*math.sin(theta), math.cos(phi))
    V, W = create_coordinate_system(normal)
    return Vector3(sample.x*V.x + sample.y*W.x + sample.z*normal.x,
                   sample.x*V.y + sample.y*W.y + sample.z*normal.y,
                   sample.x*V.z + sample.y*W.z + sample.z*normal.z).normalize()

def get_sample_uniform_hemisphere_solid_angle(X, normal):
    u1, u2 = random.random(), random.random()
    theta = 2.0 * math.pi * u1
    phi = math.acos(math.sqrt(u2))
    sample = Vector3(math.sin(phi)*math.cos(theta), math.sin(phi)*math.sin(theta), math.cos(phi))
    V, W = create_coordinate_system(normal)
    return Vector3(sample.x*V.x + sample.y*W.x + sample.z*normal.x,
                   sample.x*V.y + sample.y*W.y + sample.z*normal.y,
                   sample.x*V.z + sample.y*W.z + sample.z*normal.z).normalize()

def get_sample_cone(normal, sin_theta_max):
    u1, u2 = random.random(), random.random()
    cos_theta_max = math.sqrt(max(0.0, 1.0 - sin_theta_max**2))
    cos_theta = 1.0 + u1 * (cos_theta_max - 1.0)
    sin_theta = math.sqrt(max(0.0, 1.0 - cos_theta**2))
    phi = 2.0 * math.pi * u2
    sample = Vector3(sin_theta*math.cos(phi), sin_theta*math.sin(phi), cos_theta)
    V, W = create_coordinate_system(normal)
    return Vector3(sample.x*V.x + sample.y*W.x + sample.z*normal.x,
                   sample.x*V.y + sample.y*W.y + sample.z*normal.y,
                   sample.x*V.z + sample.y*W.z + sample.z*normal.z).normalize()

def get_sample_direction(X, normal, light_sphere, mode, sin_theta_max):
    if mode == SampleMode.UNIFORM_SOLID_ANGLE:
        return get_sample_uniform_solid_angle(X, normal)
    elif mode == SampleMode.UNIFORM_HEMISPHERE_SOLID_ANGLE:
        return get_sample_uniform_hemisphere_solid_angle(X, normal)
    else:
        return get_sample_cone(normal, sin_theta_max)

def evaluate_brdf(material, l_dir, normal, exponent):
    if material == Material.DIFFUSE:
        return 1.0 / math.pi
    elif material == Material.SPECULAR:
        outgoing = -l_dir
        temp = outgoing.dot(l_dir)
        return max(0.0, temp)**exponent * (exponent + 2.0) / (2.0 * math.pi)
    return 1.0 / math.pi

def sphere_light_sampling(X, normal, light_sphere, sample_mode, N):
    C = light_sphere.center
    R = light_sphere.radius
    L_i = light_sphere.radiance
    vec_to_light = C - X
    d_sq = vec_to_light.dot(vec_to_light)
    d = math.sqrt(d_sq)
    if d < 1e-9:
        return Color(0, 0, 0)
    sin_theta_max = R / d
    cos_theta_max = math.sqrt(max(0.0, 1.0 - sin_theta_max * sin_theta_max))
    total_L_r = Color(0, 0, 0)
    for _ in range(N):
        l_dir = get_sample_direction(X, normal, light_sphere, sample_mode, sin_theta_max)
        ray = Ray(X, l_dir)
        t_hit = light_sphere.find_intersection(ray)
        if t_hit <= 1e-9:
            continue
        P_light = X + l_dir * t_hit
        shadow_origin = X + normal * 1e-9
        shadow_ray = Ray(shadow_origin, l_dir)
        t_shadow = -1.0
        in_shadow = False
        if t_shadow > 0 and t_shadow < t_hit:
            in_shadow = True
        l_vec = P_light - X
        l_dir_norm = l_vec.normalize()
        cos_theta_i = max(0.0, l_dir_norm.dot(normal))
        n_light = (P_light - C).normalize()
        cos_theta_o = max(0.0, -n_light.dot(l_dir_norm))
        fr = evaluate_brdf(light_sphere.material, l_dir_norm, normal, light_sphere.exponent)
        L_sample = Color(0, 0, 0)
        if sample_mode == SampleMode.IMPORTANCE_SOLID_ANGLE:
            pdf_const = 2.0 * math.pi * (1.0 - cos_theta_max)
            L_sample = L_i * (1.0 - in_shadow) * cos_theta_i * pdf_const
        elif sample_mode == SampleMode.IMPORTANCE_SURFACE_AREA:
            area = (2.0 * math.pi * (1.0 - sin_theta_max) * R * R) / d_sq
            L_sample = L_i * (1.0 - in_shadow) * cos_theta_i * cos_theta_o * area * fr
        elif sample_mode == SampleMode.UNIFORM_SOLID_ANGLE:
            L_sample = fr * L_i * (1.0 - in_shadow) * cos_theta_i * 4.0 * math.pi
        elif sample_mode == SampleMode.UNIFORM_HEMISPHERE_SOLID_ANGLE:
            L_sample = fr * L_i * (1.0 - in_shadow) * cos_theta_i * 2.0 * math.pi
        elif sample_mode == SampleMode.IMPORTANCE_PHONG:
            L_sample = L_i * (1.0 - in_shadow) * cos_theta_i
        elif sample_mode == SampleMode.COSINE_WEIGHTED:
            L_sample = L_i * (1.0 - in_shadow) * fr
        total_L_r += L_sample
    return total_L_r / N
```

## Subtopics

### [Importance Sampling for Sphere Surface Area](2223-importance-sampling-for-sphere-surface-area.md)
Estimates direct illumination from a spherical light by sampling points on its visible surface area, with a probability density function proportional to the projected area of the light as seen from the shading point. It differs from [Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) by explicitly generating points on the sphere's geometry rather than in solid angle space, requiring a geometric correction factor in the estimator to account for the varying projected area relative to the viewing direction. Choose it when accurate surface-based sampling is preferred over angular sampling.

### [Sphere Light Shading with Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md)
This method estimates direct illumination from a spherical light source using Monte Carlo integration over the solid angle subtended by the light. It sets itself apart from surface-area sampling by sampling directions uniformly within the cone defined by the light's extent, which naturally handles the geometric relationship between the shading point and the light without requiring explicit area calculations. It is a specific strategy within the broader [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md) framework [G]. Choose it for efficient direct lighting estimation when the light subtends a significant solid angle.

### [EquiAngular Sampling](2272-equiangular-sampling.md)
EquiAngular Sampling is a Monte Carlo integration technique used to estimate direct lighting from point lights in participating media by importance sampling the distance domain along a ray. It solves the high variance caused by the $1/r^2$ geometric falloff and potential partial occlusion of the light source by sampling propagation distances proportional to the inverse squared length of the connecting edge. Unlike uniform solid angle or surface area sampling, which treat the light as a distant or extended object, this method re-parameterizes the integration into angular steps relative to the orthogonal projection of the light onto the ray, ensuring equal angular contributions. It is specifically designed for single scattering in homogeneous media. Choose it when rendering participating media with point lights where distance falloff dominates variance.

### Sphere Light Shading with Uniform Solid Angle Sampling
Estimates direct illumination from a spherical light by sampling uniformly over the hemisphere above the surface [S1, unofficial]. It differs from full-sphere uniform sampling by restricting samples to the upper hemisphere, which is appropriate for surface shading where light comes from above [S1, unofficial]. The PDF is $1/2\pi$ [S1, unofficial]. Choose it for simple, unbiased direct lighting estimation when importance sampling is not required or when the light is large and covers most of the hemisphere.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of samples and $K$ is the cost of ray intersection tests. Each sample requires tracing a ray to the light and a shadow ray [G].
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample, excluding scene data structures [G].
*   **Performance:** Importance sampling strategies (Solid Angle, Surface Area) generally provide lower variance than Uniform Solid Angle when the light is small or partially occluded, as they concentrate samples in the direction of the light [G]. Uniform sampling wastes samples on directions that do not hit the light [G].
*   **Strengths:** Flexible; allows switching strategies based on light size and position [G]. Handles extended spherical lights accurately [S1, unofficial].
*   **Limitations:** Requires explicit intersection tests with the light sphere [G]. Shadow ray bias can cause artifacts if not handled carefully [G].

## Background
Originated in the context of ray tracing implementations for direct lighting calculations, specifically in the `VolumeSingleScattering` repository by WangYue (2016) [S1, unofficial]. The method is part of a broader effort to efficiently compute direct lighting in scenes with complex geometry and media, leveraging multiple sampling strategies to optimize convergence [S1, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
