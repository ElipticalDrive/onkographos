# Ray Marching with Blinn-Phong Lighting

Also called: GPU Ray Marching with Signed Distance Fields

## Overview
This method renders scenes defined by implicit Signed Distance Fields (SDFs) by casting rays from a camera and iteratively marching them to the nearest surface [S1, unofficial]. It differs from analytical ray tracing by using distance estimation to handle complex geometry, and it applies a standard Blinn-Phong reflectance model to compute lighting at the hit point [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position (`u_camPos`), inverse projection matrix (`u_camInvProjMat`), world transformation matrix (`u_camToWorldMat`), maximum marching distance (`u_maxDis`), clear color (`u_clearColor`), light direction (`u_lightDir`), diffuse intensity (`u_diffIntensity`), shininess exponent (`u_shininess`), specular intensity (`u_specIntensity`), ambient intensity (`u_ambientIntensity`), and light color (`u_lightColor`).
*   **Outputs:** A final pixel color (`gl_FragColor`).

**Data Structures:**
The implementation typically resides in a fragment shader string (`fragCode`) and a vertex shader string (`vertCode`) [S1, unofficial]. The shader contains uniforms for camera and lighting parameters, and functions for scene evaluation (`scene`), ray marching (`rayMarch`), scene color retrieval (`sceneCol`), and normal estimation (`normal`) [S1, unofficial].

**Steps:**
1.  **Ray Generation:** Calculate the ray origin $\mathbf{ro}$ from the camera position uniform. Calculate the ray direction $\mathbf{rd}$ by transforming normalized screen coordinates $\mathbf{uv}$ (mapped to $[-1, 1]$) through the inverse projection matrix and world matrix, then normalizing the result [S1, unofficial].
2.  **Ray Marching:** Execute the `rayMarch(ro, rd)` function to find the total distance travelled along the ray to the nearest surface [S1, unofficial].
3.  **Hit Point Calculation:** Compute the hit point position $\mathbf{hp}$ by adding the travelled distance scaled by the ray direction to the ray origin: $\mathbf{hp} = \mathbf{ro} + \text{disTravelled} \cdot \mathbf{rd}$ [S1, unofficial].
4.  **Normal Estimation:** Compute the surface normal $\mathbf{n}$ at $\mathbf{hp}$ using the `normal(hp)` function, typically via finite differences of the SDF [S1, unofficial].
5.  **Termination Check:** If the travelled distance exceeds `u_maxDis`, output the `u_clearColor` with alpha 1.0 [S1, unofficial].
6.  **Lighting Calculation:** If a hit occurs:
    *   Calculate the diffuse term: $\text{diff} = \max(\mathbf{n} \cdot \mathbf{l}, 0) \cdot I_{\text{diff}}$, where $\mathbf{l}$ is `u_lightDir` and $I_{\text{diff}}$ is `u_diffIntensity` [S1, unofficial].
    *   Calculate the specular term: $\text{spec} = \text{diff}^{S} \cdot I_{\text{spec}}$, where $S$ is `u_shininess` and $I_{\text{spec}}$ is `u_specIntensity` [S1, unofficial].
    *   Retrieve ambient intensity $I_{\text{amb}}$ from `u_ambientIntensity` [S1, unofficial].
7.  **Color Accumulation:** Compute the final color by multiplying the light color $\mathbf{C}_{\text{light}}$ with the scene color at the hit point $\mathbf{C}_{\text{scene}}(\mathbf{hp})$ scaled by the sum of the lighting terms: $\mathbf{C}_{\text{final}} = \mathbf{C}_{\text{light}} \cdot \mathbf{C}_{\text{scene}}(\mathbf{hp}) \cdot (\text{spec} + I_{\text{amb}} + \text{diff})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms
u_camPos: vec3
u_camInvProjMat: mat4
u_camToWorldMat: mat4
u_maxDis: float
u_clearColor: vec3
u_lightDir: vec3
u_diffIntensity: float
u_shininess: float
u_specIntensity: float
u_ambientIntensity: float
u_lightColor: vec3

# Helper functions (defined in shader)
def rayMarch(ro: vec3, rd: vec3) -> float:
    # Iterative distance estimation using SDF
    # Returns total distance travelled or u_maxDis if no hit
    pass

def normal(p: vec3) -> vec3:
    # Estimate normal via finite differences of SDF
    # Returns normalized gradient vector
    pass

def sceneCol(p: vec3) -> vec3:
    # Retrieve base color of the material at point p
    pass

def main(uv: vec2):
    # 1. Ray Generation
    ro = u_camPos
    # Transform UV to [-1, 1] and apply camera matrices
    rd = (u_camInvProjMat * vec4(uv * 2.0 - 1.0, 0, 1)).xyz
    rd = (u_camToWorldMat * vec4(rd, 0)).xyz
    rd = normalize(rd)

    # 2. Ray Marching
    disTravelled = rayMarch(ro, rd)

    # 3. Hit Point Calculation
    hp = ro + disTravelled * rd

    # 4. Normal Estimation
    n = normal(hp)

    # 5. Termination Check
    if disTravelled >= u_maxDis:
        return vec4(u_clearColor, 1.0)

    # 6. Lighting Calculation
    # Light direction is assumed to be normalized in uniform or here
    l = normalize(u_lightDir)
    
    # Diffuse term: max(dot(n, l), 0) * intensity
    dotNL = max(dot(n, l), 0.0)
    diff = dotNL * u_diffIntensity
    
    # Specular term: diff^shininess * intensity
    # Note: This implementation uses the diffuse term as the base for specular
    # rather than the standard half-vector formulation.
    spec = pow(diff, u_shininess) * u_specIntensity
    
    # Ambient term
    ambient = u_ambientIntensity

    # 7. Color Accumulation
    scene_color = sceneCol(hp)
    final_color = u_lightColor * (scene_color * (spec + ambient + diff))
    
    return vec4(final_color, 1.0)
```

**Parameters:**
*   `u_shininess`: Typical values range from 1 to 128, controlling the sharpness of the specular highlight [S1, unofficial].
*   `u_maxDis`: The maximum distance a ray can travel before being considered a miss [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Finite difference normal estimation can be noisy if the step size is not chosen carefully relative to the SDF scale [G].
*   **Boundaries:** Rays exceeding `u_maxDis` must be handled to avoid rendering artifacts or infinite loops in the marching function [S1, unofficial].
*   **Lighting Model Deviation:** The specular calculation uses $\text{diff}^S$ rather than the standard Blinn-Phong half-vector $\mathbf{h} = \text{normalize}(\mathbf{l} + \mathbf{v})$. This simplifies computation but may produce less physically accurate specular highlights compared to the standard model [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of marching steps per ray. The number of steps depends on the scene complexity and the minimum distance threshold used in the marching function.
*   **Memory Complexity:** Low, as it primarily stores shader code and uniform data. No large geometry buffers are required [S1, unofficial].
*   **Strengths:** Efficiently renders complex implicit surfaces without explicit mesh data. Simple to implement in WebGL/Three.js environments [S1, unofficial].
*   **Limitations:** The simplified specular model may lack visual fidelity compared to standard Blinn-Phong. Performance degrades with high-resolution SDFs or deep scenes requiring many marching steps [S1, unofficial].
*   **When to use:** Ideal for real-time rendering of procedural geometry in web browsers using WebGL [S1, unofficial].
*   **When not to use:** When physically accurate specular reflections are required, or when rendering explicit polygonal meshes where rasterization is more efficient.

## Background
This technique is a practical implementation of GPU ray marching for implicit surfaces, popularized in web graphics contexts. It was detailed by Nabil Mansour in a guide on implementing ray marching in Three.js using WebGL shaders [S1, unofficial]. The method leverages the Blinn-Phong lighting model, a standard in computer graphics, adapted for the iterative nature of ray marching.

## Sources
[S1] Ray Marching with Three.js Guide, https://www.scribd.com/document/882338593/Ray-Marching-in-Three-js, fetched 2026-10-28 (unofficial)
