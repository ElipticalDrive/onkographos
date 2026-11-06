# Ray Marching Volume Rendering with Henyey-Greenstein Scattering

Also called: Ray Marching with Henyey-Greenstein Scattering

## Overview
This method renders a 3D volume defined by a density texture by accumulating scattered light along a ray, utilizing the Henyey-Greenstein phase function to model anisotropic scattering and shadow rays for self-shadowing [S1, unofficial]. It distinguishes itself from standard front-to-back volume rendering by explicitly incorporating directional light transport through the Henyey-Greenstein phase function, allowing for the visualization of forward or backward scattering effects in participating media [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Camera parameters: origin `ro`, direction `rd` (derived from camera forward/right/up vectors and fragment UV coordinates).
    *   Volume texture `u_volume`: 3D texture storing density in the red channel.
    *   Velocity texture `u_velocity` (optional): 3D texture storing velocity vectors.
    *   Parameters: `u_phaseG` (anisotropy $g$), `u_densityCoef` (density multiplier), `u_lightDir` (light direction), `u_lightColor` (light color), `u_showVelocity` (boolean flag).
    *   Constants: `STEP_SIZE`, `MAX_STEPS`, `SHADOW_STEP_SIZE`, `SHADOW_FACTOR`.
*   **Outputs:**
    *   Final fragment color `col`.

**Data Structures**
*   **Volume Texture:** 3D array of floats. Density is stored in the red channel (`texture(...).r`). If velocity visualization is enabled, the texture stores 3D velocity vectors.
*   **Ray State:**
    *   `ro`: `vec3` (Ray origin).
    *   `rd`: `vec3` (Ray direction, normalized).
    *   `t`: `float` (Current distance along ray).
    *   `transmit`: `float` (Accumulated transmission, initial 1.0).
    *   `col`: `vec3` (Accumulated color, initial 0.0).

**Numbered Steps**

1.  **Ray Initialization:**
    *   Calculate ray origin `ro` from camera position.
    *   Calculate ray direction `rd` from camera forward, right, and up vectors combined with fragment UV coordinates.
    *   Perform ray-box intersection with the volume bounding box defined by `boxMin = (-0.5, -0.5, -0.5)` and `boxMax = (0.5, 0.5, 0.5)` to find entry distance `t0` and exit distance `t1`.

2.  **No Intersection Check:**
    *   If `t0 > t1` or `t1 < 0`, the ray misses the volume. Output a gradient background color based on the fragment's Y coordinate and terminate.

3.  **Marcher Initialization:**
    *   Set initial march distance `t = max(t0, 0.0)`.
    *   Set maximum march distance `tMax = t1`.
    *   Initialize accumulated color `col = 0.0` and transmission `transmit = 1.0`.
    *   Add a random noise offset to `t` to reduce banding artifacts:
        $$ t = t + \text{fract}(\sin(\text{dot}(\text{gl\_FragCoord.xy}, \text{vec2}(12.9898, 78.233))) \times 43758.5453) \times \text{STEP\_SIZE} $$
        [S1, unofficial]

4.  **Precompute Phase Value:**
    *   Calculate the dot product between the view direction `rd` and the light direction `u_lightDir`. Let this be $\cos\theta$.
    *   Compute the Henyey-Greenstein phase value $P(g, \cos\theta)$ using the anisotropy parameter `u_phaseG` ($g$) and $\cos\theta$ (see Formula section).

5.  **Ray Marching Loop:**
    *   Iterate `i` from 0 to `MAX_STEPS`.
    *   If `t > tMax`, break the loop.

6.  **Position and Coordinate Calculation:**
    *   Calculate world position: $p = ro + rd \times t$.
    *   Calculate normalized texture coordinates: $\text{texCoord} = p + 0.5$.

7.  **Sampling:**
    *   If `texCoord` is within $[0, 1]$ for all axes:
        *   **If `u_showVelocity` is 1:**
            *   Sample velocity texture.
            *   Compute magnitude `mag` of the velocity vector.
            *   Set `density = mag * 0.1 * u_densityCoef`.
            *   Set `sampleCol = normalize(velocity) * 0.5 + 0.5` (visualize velocity direction as color).
        *   **Else (Standard Density):**
            *   Sample volume texture: `density = texture(u_volume, texCoord).r * u_densityCoef`.
            *   Set `sampleCol = 1.0` (white base color).
    *   If `texCoord` is out of bounds, `density = 0`.

8.  **Lighting and Accumulation:**
    *   If `density > 0.01`:
        *   Calculate alpha for the step: $\alpha = 1.0 - \exp(-\text{density} \times \text{STEP\_SIZE})$.
        *   **If `u_showVelocity` is 1:**
            *   Add emissive color: `col += sampleCol * alpha * transmit`.
        *   **Else (Lighting):**
            *   **Shadow Ray:**
                *   Initialize `shadowDens = 0.0`.
                *   Step `SHADOW_STEPS` times along `u_lightDir` with step size `SHADOW_STEP_SIZE`.
                *   In each step, check if position is within texture bounds. If so, sample density and add to `shadowDens`. If out of bounds, exit shadow ray loop early.
            *   **Light Attenuation:**
                *   Calculate light value: $\text{lightVal} = \exp(-\text{shadowDens} \times \text{SHADOW\_FACTOR} \times \text{u\_densityCoef} \times 0.1)$.
            *   **Incoming Light:**
                *   $\text{incomingLight} = \text{u\_lightColor} \times \text{lightVal} \times \text{phaseVal} + \text{ambientColor}$.
                *   [Note: `ambientColor` is not explicitly defined in the source notes but is implied by the formula structure. It is treated as a constant or uniform input.]
            *   **Accumulate:**
                *   `col += incomingLight * alpha * transmit`.
        *   **Update Transmission:**
            *   `transmit *= (1.0 - alpha)`.
            *   If `transmit < 0.01`, break the loop to save computation.
        *   **Increment:**
            *   `t += STEP_SIZE`.

9.  **Background and Tone Mapping:**
    *   Add background contribution: `col += bg * transmit`, where `bg` is the gradient background color calculated in Step 2.
    *   Apply Reinhard tone mapping: $\text{col} = \frac{\text{col}}{\text{col} + 1.0}$.
    *   Apply gamma correction: $\text{col} = \text{pow}(\text{col}, 0.4545)$.

**Formulas**

*   **Henyey-Greenstein Phase Function:**
    $$ P(g, \cos\theta) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g\cos\theta)^{1.5}} $$
    Where:
    *   $g$ is the anisotropy parameter (`u_phaseG`).
    *   $\cos\theta$ is the dot product of the view direction and light direction.
    [S1, unofficial]

*   **Alpha Calculation:**
    $$ \alpha = 1.0 - \exp(-\text{density} \times \text{STEP\_SIZE}) $$
    [S1, unofficial]

*   **Light Attenuation:**
    $$ \text{lightVal} = \exp(-\text{shadowDens} \times \text{SHADOW\_FACTOR} \times \text{u\_densityCoef} \times 0.1) $$
    [S1, unofficial]

**Parameters**

*   `SHADOW_STEP_SIZE`: Step size for shadow ray marching. Value: 0.02. [S1, unofficial]
*   `SHADOW_FACTOR`: Multiplier for shadow density attenuation. Value: 5.0. [S1, unofficial]
*   `u_phaseG`: Anisotropy parameter $g$ for Henyey-Greenstein. Range: [-0.99, 0.99]. [S1, unofficial]
*   `u_densityCoef`: Global multiplier for sampled density values. [S1, unofficial]
*   `STEP_SIZE`: Step size for main ray marching. Value not given in the sources.
*   `MAX_STEPS`: Maximum number of march steps. Value not given in the sources.
*   `SHADOW_STEPS`: Number of steps for shadow ray. Value not given in the sources.
*   `ambientColor`: Ambient light color. Value not given in the sources.

**Pitfalls and Edge Cases**

*   **Shadow Ray Bounds:** Shadow ray marching must exit early if the shadow ray position goes outside the [0,1] texture coordinate space to prevent sampling invalid memory or incorrect densities. [S1, unofficial]
*   **Early Termination:** The main ray marching loop terminates early if transmission drops below 0.01. This is a performance optimization but may cause slight inaccuracies in deep, highly opaque volumes. [S1, unofficial]
*   **Banding:** Random noise offset is added to the initial `t` to reduce banding artifacts caused by uniform step sizes. [S1, unofficial]

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_henyey_greenstein(ro, rd, u_volume, u_velocity, u_phaseG, u_densityCoef, u_lightDir, u_lightColor, u_showVelocity, bg):
    # Constants
    STEP_SIZE = 0.01  # Not given in sources, assumed typical value for implementation
    MAX_STEPS = 100   # Not given in sources, assumed typical value
    SHADOW_STEP_SIZE = 0.02
    SHADOW_FACTOR = 5.0
    SHADOW_STEPS = 10 # Not given in sources, assumed typical value
    AMBIENT_COLOR = vec3(0.1, 0.1, 0.1) # [G] Assumed default ambient color

    # 1. Ray-Box Intersection
    boxMin = vec3(-0.5)
    boxMax = vec3(0.5)
    t0, t1 = intersect_box(ro, rd, boxMin, boxMax)

    # 2. No Intersection
    if t0 > t1 or t1 < 0:
        return gradient_background(rd.y)

    # 3. Initialization
    t = max(t0, 0.0)
    tMax = t1
    col = vec3(0.0)
    transmit = 1.0

    # Random offset for banding reduction
    noise = fract(sin(dot(gl_FragCoord.xy, vec2(12.9898, 78.233))) * 43758.5453)
    t += noise * STEP_SIZE

    # 4. Precompute Phase Value
    cos_theta = dot(normalize(rd), normalize(u_lightDir))
    g = u_phaseG
    phaseVal = (1.0 - g*g) / (4.0 * PI * pow(1.0 + g*g - 2.0*g*cos_theta, 1.5))

    # 5. Marching Loop
    for i in range(MAX_STEPS):
        if t > tMax:
            break

        # 6. Position
        p = ro + rd * t
        texCoord = p + vec3(0.5)

        density = 0.0
        sampleCol = vec3(0.0)

        # 7. Sampling
        if is_inside(texCoord, 0.0, 1.0):
            if u_showVelocity:
                vel = sample_texture(u_velocity, texCoord)
                mag = length(vel)
                density = mag * 0.1 * u_densityCoef
                sampleCol = normalize(vel) * 0.5 + 0.5
            else:
                density = sample_texture(u_volume, texCoord).r * u_densityCoef
                sampleCol = vec3(1.0)

        # 8. Lighting and Accumulation
        if density > 0.01:
            alpha = 1.0 - exp(-density * STEP_SIZE)

            if u_showVelocity:
                col += sampleCol * alpha * transmit
            else:
                # Shadow Ray
                shadowDens = 0.0
                shadowPos = p + u_lightDir * STEP_SIZE # Start shadow ray from current position
                for s in range(SHADOW_STEPS):
                    shadowTex = shadowPos + vec3(0.5)
                    if is_inside(shadowTex, 0.0, 1.0):
                        shadowDens += sample_texture(u_volume, shadowTex).r
                    else:
                        break # Exit early if out of bounds
                    shadowPos += u_lightDir * SHADOW_STEP_SIZE

                lightVal = exp(-shadowDens * SHADOW_FACTOR * u_densityCoef * 0.1)
                incomingLight = u_lightColor * lightVal * phaseVal + AMBIENT_COLOR
                col += incomingLight * alpha * transmit

            transmit *= (1.0 - alpha)
            if transmit < 0.01:
                break

        t += STEP_SIZE

    # 9. Background and Tone Mapping
    col += bg * transmit
    col = col / (col + 1.0)
    col = pow(col, 0.4545)

    return col
```

## Subtopics

### Front-to-Back Volume Rendering
The foundational accumulation method used here. It processes samples from near to far, blending each sample into the accumulated color. Choose this when you need standard alpha blending without complex light transport. [2350-front-to-back-volume-rendering.md]

### Tiled Ray Marching with Grid Sampling
Optimizes memory access by grouping rays into tiles and sampling a grid. Choose this for large volumes where memory bandwidth is a bottleneck. [2586-tiled-ray-marching-with-grid-sampling.md]

### Fixed Step Ray Marching
Uses a constant step size along the ray. Choose this for simplicity and uniform sampling, but be aware of potential aliasing in sparse regions. [3136-fixed-step-ray-marching.md]

### Volumetric Raymarching with SDF Density
Uses Signed Distance Fields for density instead of textures. Choose this for procedural volumes or when memory bandwidth for large textures is an issue. [4484-volumetric-raymarching-with-sdf-density.md]

## Operating characteristics
*   **Time Complexity:** $O(N \times M)$, where $N$ is the number of pixels (rays) and $M$ is the number of steps per ray (`MAX_STEPS`). Shadow rays add a factor of `SHADOW_STEPS` to the lighting calculation per step.
*   **Memory Complexity:** $O(V)$, where $V$ is the size of the volume texture.
*   **Performance:** The method is computationally intensive due to the shadow ray casting for every sample. The early termination based on transmission (`transmit < 0.01`) helps optimize performance in opaque regions.
*   **Strengths:** Accurately models anisotropic scattering, allowing for realistic lighting effects in participating media like fog or smoke. Simple to implement on the GPU.
*   **Limitations:** Shadow rays significantly increase computation time. Fixed step size may lead to aliasing or inaccurate density integration in sparse regions.
*   **When to Use:** Real-time rendering of volumetric effects where anisotropic scattering is desired and performance allows for shadow ray casting.
*   **When Not to Use:** When real-time performance is critical and anisotropic scattering is not required (use standard front-to-back rendering). When the volume is extremely sparse and fixed step size leads to poor sampling.

## Background
This method is derived from real-time volume rendering techniques used in fluid simulation visualizations. It combines standard ray marching with the Henyey-Greenstein phase function to model light scattering in participating media. The specific implementation details, including the random offset for banding reduction and the shadow ray formulation, are based on a GitHub repository for fluid simulation. [S1, unofficial]

## Sources
[S1] tzhazuma/Fluid_simulation, https://github.com/tzhazuma/Fluid_simulation, fetched 2026-10-26 (unofficial)
