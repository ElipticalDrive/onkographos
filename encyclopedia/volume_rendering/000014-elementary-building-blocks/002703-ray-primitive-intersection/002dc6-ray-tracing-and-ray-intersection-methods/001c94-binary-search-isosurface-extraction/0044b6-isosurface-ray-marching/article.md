# IsoSurface Ray Marching

Also called: Pseudo Iso Surface, Pseudo IsoSurface Rendering using Ray Casting, Shader Type 3: Iso Surface with RAY CASTING Method

## Overview
IsoSurface Ray Marching is a ray casting technique that renders an isosurface by sampling a 3D volume scalar field and comparing values against a threshold [S1, unofficial]. It avoids explicit polygon extraction by detecting zero crossings during ray traversal, refining hit positions via bisection, and shading with Phong lighting [S1, unofficial]. Unlike [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md), which approximates position by iteratively halving an interval, this method integrates the marching process directly into the ray traversal loop with a fixed step size, refining only upon crossing detection [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume texture (`u_Volume3DSampler`), camera position (`u_cameraPosition`), step size (`u_stepSize`), isovalue threshold (`u_isoValue`), and vertex attributes providing 3D texture coordinates (`oTexCoords`) [S1, unofficial].
*   **Output:** A fragment color (`FragColor`) representing the shaded isosurface [S1, unofficial].

**Data Structures**
*   **Volume Texture:** 3D texture object bound to unit 0, sampled using `sampler3D` [S1, unofficial].
*   **Uniforms:**
    *   `u_MVPMatrix`: Model-View-Projection matrix [S1, unofficial].
    *   `u_Volume3DSampler`: Sampler for the 3D volume [S1, unofficial].
    *   `u_cameraPosition`: Camera position in world space [S1, unofficial].
    *   `u_stepSize`: Step vector for ray advancement [S1, unofficial].
    *   `u_isoValue`: Integer threshold, normalized in the shader by dividing by 255.0 [S1, unofficial].
*   **Constants:**
    *   `MAX_SAMPLES`: 300 [S1, unofficial].
    *   `DELTA`: 0.01 (used for gradient calculation) [S1, unofficial].

**Numbered Steps**
1.  **Initialization:** Compile and link vertex and fragment shaders from `.glsl` files. Bind attribute `aPosition`. Retrieve uniform locations for `u_MVPMatrix`, `u_Volume3DSampler`, `u_cameraPosition`, and `u_stepSize` [S1, unofficial].
2.  **Rendering Setup:** Enable blending. Set the shader program. Pass the Model-View-Projection matrix, camera position, step size, and isovalue. Bind the 3D volume texture to unit 0. Draw a unit cube using 36 unsigned short indices [S1, unofficial].
3.  **Ray Initialization:** Initialize ray origin `dataPosition` from interpolated 3D texture coordinates `oTexCoords` [S1, unofficial].
4.  **Ray Direction:** Calculate `geometryDirection` by subtracting 0.5 from `oTexCoords`, subtracting `u_cameraPosition`, and normalizing the result [S1, unofficial].
5.  **Step Vector:** Calculate `directionStep` by multiplying `geometryDirection` with `u_stepSize` [S1, unofficial].
6.  **Ray Marching Loop:** Loop `i` from 0 to `MAX_SAMPLES` (300):
    a.  Advance `dataPosition` by `directionStep` [S1, unofficial].
    b.  **Boundary Check:** If `dot(sign(dataPosition - texMin), sign(texMax - dataPosition)) < 3.0`, break the loop. This condition triggers if any component of `dataPosition` falls outside the [0, 1] bounds [S1, unofficial].
    c.  **Sampling:** Sample the volume at current `dataPosition` (`fSample`) and next position `dataPosition + directionStep` (`fSample2`) [S1, unofficial].
    d.  **Zero Crossing Detection:** If `(fSample - isoValue) < 0` AND `(fSample2 - isoValue) >= 0.0`, refine the intersection [S1, unofficial].
    e.  **Refinement:** Call `Bisection(xN, xF, isoValue)` where `xN` is `dataPosition` and `xF` is `dataPosition + directionStep`. The bisection loop runs 4 times, halving the interval `[left, right]` based on the sample value relative to `iso` [S1, unofficial].
    f.  **Normal Calculation:** Calculate normal `N` at the refined position `tc` using `GetGradient(tc)`. `GetGradient` uses central finite difference with `DELTA` and normalizes the result [S1, unofficial].
    g.  **Shading:** Compute final color using `PhongLighting(L, N, V, 250, vec3(0.5))` and assign to `FragColor`. Break the loop [S1, unofficial].

**Formulas**
*   **Gradient Approximation:**
    $$N = \text{normalize}\left(\frac{S_{+} - S_{-}}{2.0}\right)$$
    Where $S_{-}$ and $S_{+}$ are samples at $uvw - \Delta$ and $uvw + \Delta$ along each axis [S1, unofficial].
*   **Phong Lighting:**
    $$C = (\max(L \cdot N, 0) \cdot C_{diffuse} + \max(0.00001, (\frac{L+V}{|L+V|} \cdot N))^p) \cdot 1.0$$
    Where $L$ is light direction, $N$ is normal, $V$ is view direction, $p$ is shininess (250), and $C_{diffuse}$ is diffuse color (vec3(0.5)) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
# Input: aPosition (attribute), oTexCoords (varying)
# Output: gl_Position

def vertex_shader(aPosition, u_MVPMatrix):
    gl_Position = u_MVPMatrix * vec4(aPosition, 1.0)
    oTexCoords = aPosition # Assuming aPosition maps to texture coords for the unit cube

# Fragment Shader
# Input: oTexCoords (varying)
# Output: FragColor

def fragment_shader(oTexCoords, u_Volume3DSampler, u_cameraPosition, u_stepSize, u_isoValue):
    iso = u_isoValue / 255.0 # Normalize integer isoValue
    MAX_SAMPLES = 300
    DELTA = 0.01
    texMin = vec3(0.0)
    texMax = vec3(1.0)

    # Ray Initialization
    dataPosition = oTexCoords
    geometryDirection = normalize(oTexCoords - 0.5 - u_cameraPosition)
    directionStep = geometryDirection * u_stepSize

    for i in range(MAX_SAMPLES):
        # Advance
        dataPosition += directionStep

        # Boundary Check
        if dot(sign(dataPosition - texMin), sign(texMax - dataPosition)) < 3.0:
            break

        # Sampling
        fSample = sample_volume(u_Volume3DSampler, dataPosition)
        fSample2 = sample_volume(u_Volume3DSampler, dataPosition + directionStep)

        # Zero Crossing Detection
        if (fSample - iso) < 0 and (fSample2 - iso) >= 0.0:
            # Refine Intersection
            tc = bisection(dataPosition, dataPosition + directionStep, iso, u_Volume3DSampler)

            # Calculate Normal
            N = get_gradient(tc, DELTA, u_Volume3DSampler)

            # Shading
            L = vec3(0.0) # Placeholder for light direction, assumed normalized
            V = vec3(0.0) # Placeholder for view direction, assumed normalized
            # Note: Source does not specify L and V calculation, assuming they are available or constant
            # Using source formula structure
            diffuse = max(dot(L, N), 0.0) * vec3(0.5)
            half_vec = normalize(L + V)
            specular = pow(max(dot(half_vec, N), 0.00001), 250.0)
            FragColor = vec4((diffuse + specular) * 1.0, 1.0)
            return FragColor

    # If no intersection found, return background or black
    FragColor = vec4(0.0, 0.0, 0.0, 1.0)
    return FragColor

def bisection(xN, xF, iso, u_Volume3DSampler):
    left = xN
    right = xF
    for _ in range(4):
        mid = (left + right) / 2.0
        sample = sample_volume(u_Volume3DSampler, mid)
        if sample < iso:
            left = mid
        else:
            right = mid
    return (left + right) / 2.0

def get_gradient(tc, DELTA, u_Volume3DSampler):
    # Central finite difference
    # Assuming tc is vec3
    grad = vec3(0.0)
    # X axis
    grad.x = sample_volume(u_Volume3DSampler, tc + vec3(DELTA, 0.0, 0.0)) - sample_volume(u_Volume3DSampler, tc - vec3(DELTA, 0.0, 0.0))
    # Y axis
    grad.y = sample_volume(u_Volume3DSampler, tc + vec3(0.0, DELTA, 0.0)) - sample_volume(u_Volume3DSampler, tc - vec3(0.0, DELTA, 0.0))
    # Z axis
    grad.z = sample_volume(u_Volume3DSampler, tc + vec3(0.0, 0.0, DELTA)) - sample_volume(u_Volume3DSampler, tc - vec3(0.0, 0.0, DELTA))
    return normalize(grad / 2.0)

def sample_volume(sampler, pos):
    # Standard 3D texture sampling returning a scalar value
    return texture(sampler, pos).r
```

**Parameters**
*   `u_isoValue`: Uniform integer representing the isovalue threshold. Normalized in shader by dividing by 255.0 [S1, unofficial].
*   `MAX_SAMPLES`: Constant integer defining the maximum number of ray march steps per ray. Value: 300 [S1, unofficial].
*   `DELTA`: Constant float defining the step size for gradient calculation using finite differences. Value: 0.01 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Boundary Check:** The boundary check uses `dot(sign(dataPosition - texMin), sign(texMax - dataPosition)) < 3.0`. If any component is out of bounds [0,1], the sign product becomes negative or zero, reducing the dot product below 3, triggering termination [S1, unofficial].
*   **Precision:** The bisection is limited to 4 iterations, which may not provide high precision for all cases [S1, unofficial].
*   **Lighting:** The source does not specify how light direction `L` and view direction `V` are computed in the shader, which is a gap for implementation [G].

## Subtopics

### Surface Location via Segment Endpoint Values
Determines surface location based on values at segment endpoints. Choose when endpoint values are sufficient for accurate surface approximation. [Link](3756-surface-location-via-segment-endpoint-values.md)

### Isocontour Shading with Curvature
Enhances shading by incorporating curvature information. Choose when surface detail and curvature-based shading are required. [Link](3819-isocontour-shading-with-curvature.md)

### Multi-Stage Rasterization for Shaded Isosurfaces
Uses multiple rasterization passes for shading. Choose when complex shading effects are needed and performance allows for multiple passes. [Link](5394-multi-stage-rasterization-for-shaded-isosurfaces.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays (pixels). Each ray marches up to `MAX_SAMPLES` (300) steps, with bisection adding a constant overhead (4 iterations) upon crossing detection [S1, unofficial].
*   **Memory Complexity:** Dominated by the 3D volume texture storage. The shader uses minimal additional memory [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with resolution (number of rays) and volume size (texture memory). The fixed step size may lead to aliasing or missed surfaces if the step size is too large relative to volume features [S1, unofficial].
*   **Strengths:** Simple implementation, no explicit mesh generation, real-time capable with fixed step size [S1, unofficial].
*   **Limitations:** Fixed step size can lead to inaccuracies or missed surfaces; boundary check logic is specific to the [0,1] texture coordinate space; lighting calculation details are missing [S1, unofficial].
*   **When to use:** Real-time visualization of isosurfaces where explicit mesh generation is undesirable or too costly [S1, unofficial].
*   **When not to use:** When high precision is required, or when the volume has features smaller than the step size [S1, unofficial].

## Background
*   **Origin:** The method is described in the "Volumetric Render Engine" GitHub repository by mikejernil [S1, unofficial].
*   **History and Development:** The code is part of a larger volumetric rendering application that includes multiple rendering modes (slicing, raycasting, colormap classification, marching tetrahedra) [S1, unofficial].
*   **Usage:** Used in the "Volumetric Render Engine" for rendering isosurfaces from 3D volume data [S1, unofficial].

## Sources
[S1] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S2] Interactive Visual data analysis | Daniel Suttor, https://dasutt.github.io/stories/interactive-visual-data-analysis/, fetched 2026-10-28 (unofficial)
