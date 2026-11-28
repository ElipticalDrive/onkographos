# Gradient-based Shading with Phase Functions

Also called: Lambert's Law Phase Function Shading, Shading and Illumination, Shading and Shadows

## Overview
Gradient-based shading with phase functions adds an illumination term to the emissive term of the volume rendering integral to simulate single-scattering effects, or computes shading for implicit iso-surfaces [S1, unofficial][S2, book]. It distinguishes itself from sibling methods by explicitly deriving surface normals from the scalar field gradient and applying phase functions (such as Lambert's law or Henyey-Greenstein) to model light interaction, rather than relying solely on frequency domain approximations or simplified Phong models [S1, unofficial][S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume field $f$, a ray origin $o$ and direction $r$, light source positions, and material parameters (e.g., scattering coefficient, phase function parameters) [G].
*   **Output:** A shaded color value for the ray intersection point [G].

**Data Structures:**
*   **Scalar Field:** A 3D grid of floating-point values representing density or intensity [G].
*   **Iso-surface Definition:** A special transfer function that evaluates to one for exactly one discrete input value and zero for all others [S1, unofficial].

**Steps:**
1.  **Ray-Volume Intersection:** Trace the ray through the volume. If shading an iso-surface, find the intersection point where the scalar field value matches the iso-value [S1, unofficial].
2.  **Gradient Computation:** Compute the normal vector $\mathbf{n}$ at the intersection point. For a continuous field, $\mathbf{n} = \nabla f / \|\nabla f\|$. For discrete samples, use central differencing on the neighbors of the sample point [S2, book].
3.  **Phase Function Evaluation:** Calculate the phase function $P(\theta)$ based on the angle $\theta$ between the incoming ray direction $\mathbf{r}$ and the normal vector $\mathbf{n}$. For diffuse particles much larger than the light wavelength, use Lambert's law derived phase function [S2, book]:
    $$ P_L(\theta) = \frac{8}{3\pi} (\sin\theta + (\pi - \theta)\cos\theta) $$
    where $\theta$ is the angle between the incoming ray $\mathbf{r}$ and the normal $\mathbf{n}$ [S2, book]. For multiple scattering attenuation, the Henyey-Greenstein function can be used, with Schlick's approximation often preferred for speed [S2, book]:
    $$ P_S(a) = \frac{1-g^2}{(1-ga)^2} $$
    where $a = \cos\theta$ and $g$ is a scattering parameter [S2, book].
4.  **Shading Application:** Combine the phase function with lighting models. This can include Phong shading with multiple light sources or global illumination techniques [S1, unofficial]. For shadows, a two-pass approach may be used: first render the volume from the light source using the absorption model to store light contribution, then apply this to the view pass [S2, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class ScalarField:
    def __init__(self, data, spacing=1.0):
        self.data = data
        self.spacing = spacing
        self.shape = (len(data), len(data[0]), len(data[0][0]))

    def get_value(self, x, y, z):
        ix, iy, iz = int(x / self.spacing), int(y / self.spacing), int(z / self.spacing)
        ix = max(0, min(ix, self.shape[0] - 1))
        iy = max(0, min(iy, self.shape[1] - 1))
        iz = max(0, min(iz, self.shape[2] - 1))
        return self.data[ix][iy][iz]

def compute_gradient_shading(grid, ray_origin, ray_dir, light_positions, iso_value, max_dist=100.0, step=0.01):
    # 1. Ray marching to find iso-surface intersection
    t = 0.0
    point = None
    while t < max_dist:
        x, y, z = ray_origin[0] + t * ray_dir[0], ray_origin[1] + t * ray_dir[1], ray_origin[2] + t * ray_dir[2]
        val = grid.get_value(x, y, z)
        if abs(val - iso_value) < 1e-4:
            point = (x, y, z)
            break
        t += step
    if point is None:
        return None

    # 2. Gradient computation via central differencing
    eps = 0.01
    dx = grid.get_value(point[0]+eps, point[1], point[2]) - grid.get_value(point[0]-eps, point[1], point[2])
    dy = grid.get_value(point[0], point[1]+eps, point[2]) - grid.get_value(point[0], point[1]-eps, point[2])
    dz = grid.get_value(point[0], point[1], point[2]+eps) - grid.get_value(point[0], point[1], point[2]-eps)

    grad_len = math.sqrt(dx*dx + dy*dy + dz*dz)
    if grad_len < 1e-6:
        normal = (0.0, 0.0, 1.0)
    else:
        normal = (dx/grad_len, dy/grad_len, dz/grad_len)

    # 3. Phase function evaluation
    cos_theta = ray_dir[0]*normal[0] + ray_dir[1]*normal[1] + ray_dir[2]*normal[2]
    cos_theta = max(-1.0, min(1.0, cos_theta))
    theta = math.acos(cos_theta)

    phase = (8.0 / (3.0 * math.pi)) * (math.sin(theta) + (math.pi - theta) * math.cos(theta))

    # 4. Shading application
    color = (0.0, 0.0, 0.0)
    for light_pos in light_positions:
        light_vec = (light_pos[0]-point[0], light_pos[1]-point[1], light_pos[2]-point[2])
        light_len = math.sqrt(light_vec[0]**2 + light_vec[1]**2 + light_vec[2]**2)
        if light_len < 1e-6:
            continue
        light_dir = (light_vec[0]/light_len, light_vec[1]/light_len, light_vec[2]/light_len)

        diff = max(0.0, normal[0]*light_dir[0] + normal[1]*light_dir[1] + normal[2]*light_dir[2])
        color = (color[0] + diff * phase, color[1] + diff * phase, color[2] + diff * phase)

    return color
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial][S2, book]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial][S2, book]
*   **Strengths:** Provides intuitive depth cues; computationally efficient compared to full multiple scattering solutions; allows for standard surface shading techniques (Phong) to be applied to volumes [S1, unofficial][S2, book].
*   **Limitations:** Relies on the assumption that the volume can be approximated by an iso-surface or that local gradients are meaningful; may produce artifacts if the volume is noisy or has low resolution [S2, book].
*   **When to use:** Not given in the sources. [S1, unofficial][S2, book]
*   **When not to use:** Not given in the sources. [S1, unofficial][S2, book]

## Background
This technique originates from the need to enhance the visual quality of direct volume rendering by incorporating lighting models traditionally used for surfaces [S1, unofficial][S2, book]. The use of gradients for normal estimation in discrete volumes was established by Höhne and Bernstein [S2, book]. The application of Lambert's law derived phase functions for diffuse particles larger than the wavelength was described by Blinn [S2, book]. The integration of these concepts into volume rendering pipelines allows for the simulation of single-scattering effects and shadows, significantly improving the perceptibility of depth [S1, unofficial][S2, book].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
