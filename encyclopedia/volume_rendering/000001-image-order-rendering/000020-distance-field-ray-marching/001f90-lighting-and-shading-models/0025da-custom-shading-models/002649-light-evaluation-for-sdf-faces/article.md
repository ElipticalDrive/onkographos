# Light Evaluation for SDF Faces
## Overview
This method applies diffuse, specular, and ambient lighting to a face color based on a surface normal and light direction [S1, unofficial]. It operates directly on discretized face geometry data rather than integrating light along a ray [S1, unofficial].
## How it works
**Inputs:**
- `faceColor`: A `float4` containing the base RGB color and alpha ($\alpha$) of the surface face.
- `n`: The surface normal vector ($\mathbf{n}$), typically derived from the SDF gradient [G].
- `_LightAngle`: A scalar defining the light direction via sine and cosine parameters.
- `_Diffuse`: A scalar diffuse intensity factor.
- `_Ambient`: A scalar ambient intensity factor.
**Outputs:**
- `color`: A `float4` representing the final lit color and alpha.
**Data Structures:**
- The normal $\mathbf{n}$ is a normalized 3D vector.
- The light direction $\mathbf{l}$ is constructed from `_LightAngle`.
**Steps:**
1. **Normal Adjustment:** Set the z-component of the normal to its absolute value: $n_z = |n_z|$ [S1, unofficial].
2. **Light Direction Construction:** Construct the light direction vector $\mathbf{l}$ from the sine and cosine of `_LightAngle`, then normalize it [S1, unofficial].
3. **Specular Addition:** Compute the base color clamped to zero, add the specular contribution scaled by the face alpha: $\mathbf{c} = \max(\text{faceColor.rgb}, 0) + \text{GetSpecular}(\mathbf{n}, \mathbf{l}) \times \text{faceColor.a}$ [S1, unofficial].
4. **Diffuse Modulation:** Multiply the color by the factor $(1 - \mathbf{n} \cdot \mathbf{l} \times \_Diffuse)$: $\mathbf{c} = \mathbf{c} \times (1 - \mathbf{n} \cdot \mathbf{l} \times \_Diffuse)$ [S1, unofficial].
5. **Ambient/Height Modulation:** Multiply the color by a linear interpolation between `_Ambient` and $1$ controlled by the squared z-component of the normal: $\mathbf{c} = \mathbf{c} \times \text{lerp}(\_Ambient, 1, n_z^2)$ [S1, unofficial].
6. **Output Assembly:** Set the final color to $\text{float4}(\mathbf{c}, \text{faceColor.a})$ [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_light_face(face_color, normal, light_angle, diffuse, ambient):
    # 1. Normal adjustment: n.z = abs(n.z)
    n = [normal[0], normal[1], abs(normal[2])]

    # 2. Light direction: normalize light vector from sin/cos of _LightAngle
    sin_val = math.sin(light_angle)
    cos_val = math.cos(light_angle)
    light_dir = [cos_val, sin_val, 0.0]
    mag = math.sqrt(light_dir[0]**2 + light_dir[1]**2 + light_dir[2]**2)
    light_dir = [light_dir[i] / mag for i in range(3)]

    # 3. Specular addition: col = max(faceColor.rgb, 0) + GetSpecular(n, light) * faceColor.a
    # GetSpecular implementation not provided in sources [S1, unofficial]
    specular = get_specular(n, light_dir)
    base_rgb = [max(face_color[i], 0.0) for i in range(3)]
    lit_color = [base_rgb[i] + specular[i] * face_color[3] for i in range(3)]

    # 4. Diffuse modulation: col *= (1 - dot(n, light) * _Diffuse)
    dot_nl = n[0]*light_dir[0] + n[1]*light_dir[1] + n[2]*light_dir[2]
    lit_color = [lit_color[i] * (1.0 - dot_nl * diffuse) for i in range(3)]

    # 5. Ambient/Height modulation: col *= lerp(_Ambient, 1, n.z * n.z)
    height_factor = n[2] * n[2]
    ambient_blend = ambient + (1.0 - ambient) * height_factor
    final_color = [lit_color[i] * ambient_blend for i in range(3)]

    # 6. Output assembly: color = float4(col, faceColor.a)
    return [final_color[0], final_color[1], final_color[2], face_color[3]]

def get_specular(n, l):
    # Exact implementation not provided in sources [S1, unofficial]
    return [0.0, 0.0, 0.0]
```
## Subtopics

### Gradient-Based Surface Detection with Lighting
This variant integrates lighting directly into the SDF gradient evaluation loop [S2, unofficial]. It calculates the gradient at the current ray-marched position to determine the normal dynamically, rather than relying on pre-computed normals [S2, unofficial].
- **Steps:** Calculate the gradient at the current position: $\mathbf{g} = \nabla f(\mathbf{p})$ [S2, unofficial]. Calculate the magnitude: $m = \|\mathbf{g}\|$ [S2, unofficial]. Normalize to get the normal: $\mathbf{n} = \mathbf{g} / m$ [S2, unofficial]. Calculate lighting using the color, normal, light direction, and view direction: $\text{col.rgb} = \text{calculateLighting}(\text{col.rgb}, \mathbf{n}, \text{getLightDirection}(-\text{ray.direction}), -\text{ray.direction}, 0.15)$ [S2, unofficial]. Set alpha to $1.0$ and break the marching loop: $\text{col.a} = 1.0; \text{break}$ [S2, unofficial].
- **Parameters:** `_MinGradient` is the minimum gradient magnitude threshold required to consider a voxel as a surface for lighting calculation [S2, unofficial]. `_MinVal` is the minimum density value threshold for surface detection [S2, unofficial]. `_MaxVal` is the maximum density value threshold for surface detection [S2, unofficial].
- **Pitfalls:** If the gradient magnitude is zero or very small, division by zero occurs during normalization [S2, unofficial]. The code assumes `gradMag > _MinGradient` before normalizing, implying a check or clamp is handled elsewhere or `_MinGradient` prevents zero division [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per face evaluation [S1, unofficial]. The lighting calculation involves basic vector operations and trigonometric functions [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory [S1, unofficial].
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
## Background
This method originates from implementations in Unity-based SDF renderers, specifically the `unity-nanovdb-renderer` and `UnityVolumeRendering` projects [S1, unofficial][S2, unofficial]. Its specific historical development and broader usage contexts are not given in the sources.
## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S2] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
