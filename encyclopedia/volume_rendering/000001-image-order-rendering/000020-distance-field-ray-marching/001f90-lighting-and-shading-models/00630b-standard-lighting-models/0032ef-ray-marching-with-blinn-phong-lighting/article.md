# Ray Marching with Blinn-Phong Lighting

Also called: GPU Ray Marching with Signed Distance Fields

## Overview
This method renders scenes defined by implicit surfaces (Signed Distance Fields) by casting rays from a camera, iteratively marching them to find intersections, and shading the resulting surface points using the Blinn-Phong reflectance model [S1, unofficial]. It differs from analytical ray tracing by using iterative distance estimation to handle complex, non-analytic geometry, and it differs from pure SDF rendering by explicitly applying a local lighting model (diffuse, specular, ambient) rather than just color blending or occlusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position (`u_camPos`), inverse projection matrix (`u_camInvProjMat`), world transformation matrix (`u_camToWorldMat`), maximum marching distance (`u_maxDis`), light direction (`u_lightDir`), light color (`u_lightColor`), diffuse intensity (`u_diffIntensity`), shininess exponent (`u_shininess`), specular intensity (`u_specIntensity`), ambient intensity (`u_ambientIntensity`), and background color (`u_clearColor`) [S1, unofficial].
*   **Outputs:** A fragment color (`gl_FragColor`) for each pixel [S1, unofficial].

**Data Structures:**
The implementation typically resides in a fragment shader applied to a plane fixed to the camera's near plane [S1, unofficial]. Shader code is stored in string variables (`vertCode`, `fragCode`) and passed to a shader material [S1, unofficial]. The core logic requires a scene SDF evaluation function, a ray marching loop, a normal estimation function, and a scene color retrieval function [S1, unofficial].

**Numbered Steps:**
1.  **Ray Generation:**
    *   Obtain normalized screen coordinates `uv` [S1, unofficial].
    *   Calculate ray origin `ro` from the uniform `u_camPos` [S1, unofficial].
    *   Calculate ray direction `rd` by transforming `uv` through the inverse projection matrix and world matrix, then normalizing:
        $$ \mathbf{rd} = \text{normalize}(\mathbf{u\_camToWorldMat} \cdot (\mathbf{u\_camInvProjMat} \cdot \text{vec4}(\mathbf{uv} \cdot 2 - 1, 0, 1)).xyz) $$
2.  **Ray Marching:**
    *   Call `rayMarch(ro, rd)` to find the total distance `disTravelled` along the ray to the nearest surface [S1, unofficial].
3.  **Hit Point Calculation:**
    *   Compute the hit point position `hp`:
        $$ \mathbf{hp} = \mathbf{ro} + \text{disTravelled} \cdot \mathbf{rd} $$
4.  **Normal Estimation:**
    *   Compute the surface normal `n` at `hp` using a `normal(hp)` function [S1, unofficial]. This is typically the normalized gradient of the SDF:
        $$ \mathbf{n} = \frac{\nabla f(\mathbf{hp})}{\|\nabla f(\mathbf{hp})\|} $$
        [G] where $\nabla f$ is estimated via finite differences (e.g., sampling the SDF at small offsets along axes).
5.  **Termination Check:**
    *   If `disTravelled >= u_maxDis`, the ray did not hit a surface within bounds [S1, unofficial]. Output the clear color:
        $$ \text{gl\_FragColor} = \text{vec4}(\mathbf{u\_clearColor}, 1) $$
6.  **Lighting Calculation (Blinn-Phong):**
    *   If a hit occurs, compute the diffuse term `diff`:
        $$ \text{diff} = \max(\mathbf{n} \cdot \mathbf{u\_lightDir}, 0.0) \cdot \mathbf{u\_diffIntensity} $$
    *   Compute the specular term `spec` using the diffuse term as the basis for the shininess exponent (as per the source code logic):
        $$ \text{spec} = \text{pow}(\text{diff}, \mathbf{u\_shininess}) \cdot \mathbf{u\_specIntensity} $$
    *   Retrieve the ambient intensity:
        $$ \text{ambient} = \mathbf{u\_ambientIntensity} $$
7.  **Color Accumulation:**
    *   Retrieve the scene color at the hit point: $\mathbf{sceneCol} = \text{sceneCol}(\mathbf{hp})$ [S1, unofficial].
    *   Compute the final color:
        $$ \mathbf{color} = \mathbf{u\_lightColor} \cdot (\mathbf{sceneCol} \cdot (\text{spec} + \text{ambient} + \text{diff})) $$
    *   Output the result:
        $$ \text{gl\_FragColor} = \text{vec4}(\mathbf{color}, 1) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Self-contained fragment shader logic for Ray Marching with Blinn-Phong Lighting
# All uniforms and scene functions are passed as arguments to ensure self-containment.

def main(uv, u_camPos, u_camInvProjMat, u_camToWorldMat, u_maxDis, u_clearColor,
         u_lightDir, u_diffIntensity, u_shininess, u_specIntensity, u_ambientIntensity,
         u_lightColor, scene_func, scene_col_func):
    # 1. Ray Generation
    ro = u_camPos
    ndc = vec4(uv * 2.0 - 1.0, 0.0, 1.0)
    rd = u_camInvProjMat * ndc
    rd = vec3(rd.x, rd.y, rd.z)
    rd = u_camToWorldMat * vec4(rd, 0.0)
    rd = vec3(rd.x, rd.y, rd.z)
    rd = normalize(rd)

    # 2. Ray Marching (Sphere Tracing)
    disTravelled = rayMarch(ro, rd, u_maxDis, scene_func)

    # 3. Hit Point
    hp = ro + disTravelled * rd

    # 4. Normal
    n = normal(hp, scene_func)

    # 5. Termination Check
    if disTravelled >= u_maxDis:
        return vec4(u_clearColor, 1.0)

    # 6. Lighting
    dotNL = dot(n, u_lightDir)
    diff = max(dotNL, 0.0) * u_diffIntensity
    spec = pow(diff, u_shininess) * u_specIntensity
    ambient = u_ambientIntensity

    # 7. Color Accumulation
    sc = scene_col_func(hp)
    color = u_lightColor * (sc * (spec + ambient + diff))
    return vec4(color, 1.0)

def rayMarch(ro, rd, u_maxDis, scene_func):
    t = 0.0
    epsilon = 0.001
    while t < u_maxDis:
        p = ro + t * rd
        d = scene_func(p)
        if d < epsilon:
            return t
        t += d
    return t

def normal(p, scene_func):
    # Finite difference gradient estimation [G]
    eps = 0.001
    dx = scene_func(p + vec3(eps, 0.0, 0.0)) - scene_func(p - vec3(eps, 0.0, 0.0))
    dy = scene_func(p + vec3(0.0, eps, 0.0)) - scene_func(p - vec3(0.0, eps, 0.0))
    dz = scene_func(p + vec3(0.0, 0.0, eps)) - scene_func(p - vec3(0.0, 0.0, eps))
    return normalize(vec3(dx, dy, dz))
```

**Parameters:**
*   `u_shininess`: Specular shininess exponent. Typical values are not given in the sources.
*   `u_maxDis`: Maximum ray marching distance. Prevents infinite loops and defines the visible depth range [S1, unofficial].
*   `u_diffIntensity`, `u_specIntensity`, `u_ambientIntensity`: Multipliers for the respective lighting components [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Finite difference normal estimation can be noisy if the step size is not chosen carefully relative to the SDF's scale [G].
*   **Boundary:** Rays exceeding `u_maxDis` must be handled to avoid rendering artifacts or undefined behavior [S1, unofficial].
*   **Light Direction:** `u_lightDir` is assumed to be a uniform direction [S1, unofficial]. For point lights, the direction would need to be calculated per-pixel as `normalize(lightPos - hp)`, which is not described in the source but is a common extension [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of ray march steps, as each step requires an SDF evaluation [G]. The lighting calculation is constant time per hit [G].
*   **Memory Complexity:** Low. Primarily requires storage for uniforms and temporary variables in the shader [G]. No large buffers are needed for geometry [G].
*   **Strengths:** Efficiently renders complex, smooth implicit surfaces without explicit mesh topology [G]. Simple to implement in WebGL/Three.js as a post-process or overlay shader [S1, unofficial].
*   **Limitations:** The Blinn-Phong model is a local approximation and does not account for global illumination effects like indirect lighting or soft shadows (unless added via additional ray marches) [G]. The specific specular calculation using `pow(diff, shininess)` is a simplification and may not match standard Blinn-Phong implementations which use the half-vector [G].
*   **When to use:** Real-time rendering of SDF-based scenes where performance is critical and high-fidelity global illumination is not required [G]. Suitable for stylized or abstract visuals [G].

## Background
This technique is a practical implementation of GPU ray marching for Signed Distance Fields, popularized in the context of WebGL and libraries like Three.js [S1, unofficial]. It combines the geometric power of SDFs with the simplicity of the Blinn-Phong shading model [S1, unofficial]. The specific implementation details, including the shader structure and parameter names, are documented in guides for implementing ray marching in Three.js [S1, unofficial].

## Sources
[S1] Ray Marching with Three.js Guide, https://www.scribd.com/document/882338593/Ray-Marching-in-Three-js, fetched 2026-10-28 (unofficial)
