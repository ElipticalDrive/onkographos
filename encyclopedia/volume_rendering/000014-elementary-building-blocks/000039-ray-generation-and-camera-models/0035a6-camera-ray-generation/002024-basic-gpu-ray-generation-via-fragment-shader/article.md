# Basic GPU Ray Generation via Fragment Shader

Also called: Fragment shader ray tracing, Fragment Shader Ray Generation, Ray Tracing Fragment Shader

## Overview
This method generates camera rays for ray tracing by rendering a single quad to the screen, allowing the fragment shader to compute the ray origin and direction for each pixel using interpolated vertex coordinates [S4, S5, unofficial]. It differs from [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md) by avoiding explicit matrix multiplications in favor of geometric basis vectors derived from camera parameters, and from [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) by supporting perspective projection through a viewing plane distance [S4, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Window resolution (`iResolution`), camera position (`iRayOrigin`), camera facing direction (`iRayDir`), camera up direction (`iRayUp`), distance to viewing plane (`iPlaneDist`), and interpolated texture coordinates (`texCoord`) ranging from (0,0) to (1,1) across the quad [S4, S5, unofficial].
*   **Outputs:** A ray (origin and direction) for each pixel, which is then used for intersection testing and shading [S4, S5, unofficial].

**Data Structures:**
*   **Uniforms:** `iResolution` (`vec2`), `iRayOrigin` (`vec3`), `iRayDir` (`vec3`), `iRayUp` (`vec3`), `iPlaneDist` (`float`) [S4, S5, unofficial].
*   **Varying/Input:** `texCoord` (`vec2`) interpolated from vertex attributes (0,0) to (1,1) [S4, S5, unofficial].

**Steps:**
1.  **Setup Geometry:** Render a single quad (two triangles covering the screen) using a minimal vertex shader that performs no transformations [S4, S5, unofficial].
2.  **Compute Normalized Coordinates:** In the fragment shader, convert the interpolated `texCoord` (range [0,1]) to a normalized device coordinate `p` (range [-1,1]) [S4, S5, unofficial]:
    $$ p = 2.0 \cdot \text{texCoord} - 1.0 $$
3.  **Apply Aspect Ratio:** Scale the x-component of `p` by the aspect ratio to prevent distortion [S4, S5, unofficial]:
    $$ p.x = p.x \cdot \frac{\text{iResolution}.x}{\text{iResolution}.y} $$
4.  **Compute Camera Basis Vectors:** Calculate the camera's right vector (`camSide`) using the cross product of the forward (`iRayDir`) and up (`iRayUp`) vectors [S4, S5, unofficial]:
    $$ \text{camSide} = \text{normalize}(\text{cross}(\text{iRayDir}, \text{iRayUp})) $$
5.  **Calculate Ray Direction:** Compute the ray direction by combining the basis vectors scaled by `p` and the viewing plane distance [S4, S5, unofficial]:
    $$ \text{raydir} = \text{normalize}(p.x \cdot \text{camSide} + p.y \cdot \text{iRayUp} + \text{iPlaneDist} \cdot \text{iRayDir}) $$
6.  **Set Ray Origin:** The ray origin is the camera position [S4, S5, unofficial]:
    $$ \text{rayorig} = \text{iRayOrigin} $$
7.  **Trace and Shade:** Use the generated ray to perform intersection tests (e.g., sphere tracing) and compute the final color [S4, S5, unofficial]. Apply Lambertian shading with albedo 0.8, light intensity 8.0, light direction vec3(-0.2, 1.0, -0.3), and shadow bias 0.0001 [S2, unofficial].

**Formulas:**
*   Normalized coordinates: $p = 2.0 \cdot \text{texCoord} - 1.0$ [S4, S5, unofficial].
*   Aspect-scaled coordinates: $p.x = p.x \cdot \frac{\text{iResolution}.x}{\text{iResolution}.y}$ [S4, S5, unofficial].
*   Ray direction: $\text{raydir} = \text{normalize}(p.x \cdot \text{camSide} + p.y \cdot \text{iRayUp} + \text{iPlaneDist} \cdot \text{iRayDir})$ [S4, S5, unofficial].
*   Sphere intersection parameter: $t = \begin{cases} t_1 & \text{if } t_1 < t_2 \land t_1 \ge 0 \\ t_2 & \text{otherwise} \end{cases}$, where $t_1 = -OdotD + \sqrt{\text{base}}$, $t_2 = -OdotD - \sqrt{\text{base}}$, $\text{base} = OdotD^2 - OdotO + \text{radius}^2$, $OdotD = \text{dot}(\text{rayorig} - \text{pos}, \text{raydir})$, and $OdotO = \text{dot}(\text{rayorig} - \text{pos}, \text{rayorig} - \text{pos})$ [S4, unofficial].
*   Lambertian diffuse shading: $L = \frac{\text{albedo}}{\pi} \max(0, \mathbf{n} \cdot -\mathbf{l}) I_{\text{light}}$ [S2, unofficial].

**Parameters:**
*   `iPlaneDist`: Controls the field of view. A smaller value results in a wider FOV [S4, S5, unofficial].
*   `iResolution`: Window dimensions in pixels [S4, S5, unofficial].
*   `albedo`: 0.8 [S2, unofficial].
*   `light_intensity`: 8.0 [S2, unofficial].
*   `light_direction`: vec3(-0.2, 1.0, -0.3) [S2, unofficial].
*   `shadow_bias`: 0.0001 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Aspect Ratio Distortion:** Failing to scale the x-coordinate by the aspect ratio (`width/height`) will distort the field of view, making circles appear as ellipses [S5, unofficial].
*   **Camera Basis Orthogonality:** The `camSide` vector is computed via `cross(iRayDir, iRayUp)` [S4, S5, unofficial]. If `iRayUp` is not orthogonal to `iRayDir`, the resulting basis is not orthonormal, leading to skewed rays [G].
*   **Winding Order:** The quad's vertex winding order determines the interpolation direction of `texCoord` [S4, S5, unofficial]. Ensure the quad covers the screen correctly (e.g., vertices at (0,0), (1,0), (1,1), (0,1)) and that the interpolation maps to screen space correctly (bottom-left to top-right) [S4, S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_ray_direction(tex_coord, resolution, ray_dir, ray_up, plane_dist):
    p = [2.0 * tex_coord[0] - 1.0, 2.0 * tex_coord[1] - 1.0]
    p[0] *= resolution[0] / resolution[1]
    cam_side = normalize(cross(ray_dir, ray_up))
    raydir = normalize([p[0] * cam_side[0] + p[1] * ray_up[0] + plane_dist * ray_dir[0],
                        p[0] * cam_side[1] + p[1] * ray_up[1] + plane_dist * ray_dir[1],
                        p[0] * cam_side[2] + p[1] * ray_up[2] + plane_dist * ray_dir[2]])
    return raydir

def trace_sphere(rayorig, raydir, pos, radius):
    OdotD = dot(rayorig, raydir) - dot(pos, raydir)
    OdotO = dot(rayorig, rayorig) - 2.0 * dot(rayorig, pos) + dot(pos, pos)
    base = OdotD * OdotD - OdotO + radius * radius
    if base >= 0:
        root = math.sqrt(base)
        t1 = -OdotD + root
        t2 = -OdotD - root
        if t1 >= 0 or t2 >= 0:
            t = t1 if (t1 < t2 and t1 >= 0) else t2
            pt = [rayorig[i] + raydir[i] * t for i in range(3)]
            normal = normalize([pt[i] - pos[i] for i in range(3)])
            return {"point": pt, "normal": normal, "t": t, "hit": True}
    return {"point": [0.0, 0.0, 0.0], "normal": [0.0, 0.0, 0.0], "t": -1.0, "hit": False}

def shade_lambertian(hit_point, normal, light_dir, albedo, light_intensity, shadow_bias):
    light_dir_norm = normalize(light_dir)
    diff = max(0.0, dot(normal, [-light_dir_norm[i] for i in range(3)]))
    return (albedo / math.pi) * diff * light_intensity

def main():
    rayorig = iRayOrigin
    raydir = compute_ray_direction(texCoord, iResolution, iRayDir, iRayUp, iPlaneDist)
    hit = trace_sphere(rayorig, raydir, sphere_pos, sphere_radius)
    if hit["hit"]:
        light_dir = [-0.2, 1.0, -0.3]
        color = shade_lambertian(hit["point"], hit["normal"], light_dir, 0.8, 8.0, 0.0001)
    else:
        color = [0.0, 0.0, 0.0]
    fragColor = [color[0], color[1], color[2], 1.0]
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S4, S5, unofficial].
*   **Memory Complexity:** Not given in the sources [S4, S5, unofficial].
*   **Performance:** Not given in the sources [S4, S5, unofficial].
*   **Strengths:** Simple to implement, leverages GPU interpolation, minimal geometry overhead [S4, S5, unofficial].
*   **Limitations:** Not given in the sources [S4, S5, unofficial].
*   **When to Use:** Not given in the sources [S4, S5, unofficial].
*   **When Not to Use:** Not given in the sources [S4, S5, unofficial].

## Background
This technique is a fundamental approach to GPU ray tracing, popularized by educational resources and demos [S4, S5, unofficial]. It allows developers to implement ray tracing in the fragment shader without complex vertex transformations, making it accessible for learning and simple applications [S4, S5, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
[S3] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S4] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1819/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S5] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
