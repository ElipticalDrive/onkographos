# Ray Marching with Single Scattering

Also called: Real-time Volumetric Rendering, Volumetric Cloud Rendering, Volumetric Cloud Rendering with Ray Marching

## Overview
Ray marching with single scattering is a real-time technique for rendering volumetric clouds using OpenGL/WebGL ray marching, procedural noise for density, and physically-based single scattering illumination [S1, S2, unofficial]. It solves the problem of rendering translucent, animated volumetric media efficiently by approximating light transport with a single scattering event per sample, balancing visual fidelity against computational cost [S1, S2, unofficial]. It differs from [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by using procedural noise instead of signed distance fields for density, and from [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) by focusing on localized cloud volumes with explicit bounding boxes and wind animation rather than global atmospheric scattering [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Eye position ($e$), screen position ($s$), camera rotation matrix ($camRot$), bounding box ($bMin, bMax$), sun position ($uUserSunPos$), sun color ($uUserSunColor$), time ($uTime$), resolution ($uResolution$), and various uniform parameters (step sizes, coefficients, noise settings).
- **Output:** Fragment color ($fColor$).

**Data Structures:**
- **Bounding Box:** Defined by $bMin$ and $bMax$. The effective bounds are adjusted by $uUserSize$: $bMin.x -= uUserSize.x$, $bMax.x += uUserSize.x$, etc. [S2, unofficial].
- **Noise Texture (Optional):** If using texture-based density, a `FloatingPointTexture` uses a `Float32Array` buffer of size $textureWidth \times textureHeight \times 4$. Pixels are stored in RGBA order, with index $(x + y \times textureWidth) \times 4$ for R, $+1$ for G, $+2$ for B, $+3$ for A [S2, unofficial]. The texture is 2D, RGBA format, FLOAT type, NEAREST filtering, CLAMP_TO_EDGE wrapping [S2, unofficial].
- **Procedural Noise:** Uses Perlin or Simplex noise (`cnoise` or `snoise`) combined with Fractal Brownian Motion (fBm) [S1, unofficial].

**Parameters:**
- $MAX\_STEPS = 50$ [S1, unofficial].
- $STEP\_SIZE = 0.08$ [S1, unofficial].
- $MAX\_STEPS\_LIGHT = 5$ [S1, unofficial].
- $STEP\_SIZE\_LIGHT = 0.6$ [S1, unofficial].
- $scatteringCoeff$: 1.5 (sunset), 1.2 (day) [S1, unofficial].
- $absorptionCoeff$: 0.4 (sunset), 0.2 (day) [S1, unofficial].
- $extinctionCoeff = scatteringCoeff + absorptionCoeff$ [S1, unofficial].
- $WIND = vec3(0.7, 0.2, -0.5)$ [S1, unofficial].
- $SUN = normalize(vec3(1.5, 1.0, 1.0))$ [S1, unofficial].
- $CAMERA = vec3(0.0, 0.0, -3.0)$ [S1, unofficial].
- $SUN\_COLOUR\_SUNSET = vec3(1.8, 1.1, 0.6)$ [S1, unofficial].
- $SUN\_COLOUR\_DAY = vec3(1.5, 1.3, 0.9)$ [S1, unofficial].
- $uUserSize, uUserSpeed, uUserSunPos, uUserSunColor, uUserDensity, uUserScale, uUserZoom, uUserControl, uRandomVector$ are user-defined uniforms [S2, unofficial].

**Formulas:**
- **Sky Color:** $screenRange = clamp(rayDir.y \times 0.5 + 0.5, 0.0, 1.0)$; $sky = mix(skyBottom, skyTop, screenRange)$ [S1, unofficial].
- **Sphere SDF:** $sdfSphere(p, r) = -length(p) + r$ [S1, unofficial].
- **Henyey-Greenstein Phase Function:**
  $$
  henyeyGreenstien(cos\Theta) = mix(forward, backward, 0.4) \times 0.4
  $$
  where
  $$
  forward = \frac{1 - g_1^2}{(1 + g_1^2 - 2g_1 cos\Theta)^{1.5}}, \quad g_1 = 0.7
  $$
  $$
  backward = \frac{1 - g_2^2}{(1 + g_2^2 - 2g_2 cos\Theta)^{1.5}}, \quad g_2 = -0.2
  $$
  [S1, unofficial].
- **Beer-Lambert Law:** $transmittance *= exp(-density \times extinctionCoeff \times STEP\_SIZE \times 1.5)$ [S1, unofficial]. Note: The source code applies an additional multiplier of 1.5 to the exponent [S1, unofficial].
- **Single Scattering Approximation:** $scattering = (1. - localTransmittance) \times currentLightRayT$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_single_scattering(e, s, camRot, uUserSize, uBoundingBox, uTime, uUserSpeed, uRandomVector, uUserSunPos, uUserSunColor, uUserDensity, uUserScale, uUserZoom, uUserControl, uMouse, sunset):
    # 1. Initialize Ray
    rayDir = normalize(s - e)
    
    # 2. Camera Setup
    CAM = vec3(0.0, 0.0, uUserZoom)
    if uUserControl == 1:
        camRot = create_rotation_matrix(uMouse.x, uMouse.y) # [G] Helper to create rotation matrix from mouse coords
    else:
        camRot = create_rotation_matrix(mod(uTime * 5.0, 350.0), 0.0) # [G] Helper to create rotation matrix from time

    # 3. Sky Color
    sky = calculate_sky_color(rayDir, sunset)

    # 4. Bounding Box Adjustment
    bMin = uBoundingBox.bMin - uUserSize
    bMax = uBoundingBox.bMax + uUserSize

    # 5. Main Loop Variables
    transmittance = 1.0
    t = 0.0
    colour = vec3(0.0)

    # 6. Main Ray Marching Loop
    for i in range(MAX_STEPS):
        p = e + t * rayDir
        
        # Check if inside volume (using sphere SDF as example from S1, or bounding box from S2)
        # S1 uses sphere SDF: dist = sdfSphere(p, 2.0)
        # S2 uses rayMarch function with bounding box intersection
        # Merging logic: Check if p is within bMin and bMax
        if not (all(p >= bMin) and all(p <= bMax)):
            t += STEP_SIZE
            continue

        # Calculate Density
        # S1: Procedural Noise
        # density = clamp(octaves(p + animationIndex * WIND), 0.0, 1.0) * smoothstep(0.0, 1.0, dist)
        # S2: Texture-based or procedural
        # density = volumetricDensityMap3d(marchPoint, textureOffset, uUserScale) / uUserDensity
        
        # Using procedural noise as primary method from S1
        animationIndex = uTime # [G] Assuming animationIndex is derived from time
        density = clamp(octaves(p + animationIndex * WIND), 0.0, 1.0)
        # Apply soft boundary if using SDF
        # dist = sdfSphere(p, 2.0)
        # density *= smoothstep(0.0, 1.0, dist)

        # Early exit for low density
        if density < 0.01:
            t += STEP_SIZE
            continue

        # Light Marching Loop
        transmittanceLight = 1.0
        tLight = 0.0
        lightColor = vec3(0.0)
        
        for j in range(MAX_STEPS_LIGHT):
            pLight = p + tLight * SUN
            
            # Calculate Light Density
            # S1: Procedural
            densityLight = clamp(octaves(pLight + animationIndex * WIND), 0.0, 1.0)
            # S2: Texture-based
            # lightDensity = volumetricDensityMap3d(lightMarchPoint, textureOffset, uUserScale) / uUserDensity
            
            # Early exit for light
            if densityLight < 0.05:
                tLight += STEP_SIZE_LIGHT
                continue

            # Update Light Transmittance (Beer-Lambert)
            # Note: Source S1 uses extinctionCoeff * 1.5 multiplier
            transmittanceLight *= exp(-densityLight * extinctionCoeff * STEP_SIZE_LIGHT * 1.5)
            
            if transmittanceLight < 0.05:
                break
                
            tLight += STEP_SIZE_LIGHT

        # Calculate Scattering
        # Phase Function
        cosTheta = dot(rayDir, SUN)
        phase = henyey_greenstein(cosTheta)
        
        # Ambient
        ambient = sky * 0.7
        
        # Scattered Light
        sunColour = uUserSunColor if not sunset else SUN_COLOUR_SUNSET # [G] Select sun color based on mode
        scattered = ((sunColour * transmittanceLight * phase * 6.0) + ambient) * scatteringCoeff
        
        # Local Transmittance
        localTransmittance = exp(-density * extinctionCoeff * STEP_SIZE * 1.5)
        
        # Single Scattering Contribution
        # S2: scattering = (1. - localTransmittance) * currentLightRayT
        # S1: colour += transmittance * density * scattered * STEP_SIZE
        # Merging: Use S1's accumulation with S2's scattering approximation if preferred, 
        # but S1's code explicitly accumulates density * scattered. 
        # We follow S1's accumulation logic as it is more detailed in the shader.
        
        colour += transmittance * density * scattered * STEP_SIZE
        
        # Update Main Transmittance
        transmittance *= localTransmittance
        
        # Early exit for main ray
        if transmittance < 0.01:
            break
            
        t += STEP_SIZE

    # 7. Final Composition
    # Add sky contribution
    colour += transmittance * sky
    
    # S2 adds cloudColor + col * globalTransmittance, but S1 accumulates directly into colour.
    # We stick to S1's final output.
    fColor = vec4(colour, 1.0)
    return fColor

def calculate_sky_color(rayDir, sunset):
    screenRange = clamp(rayDir.y * 0.5 + 0.5, 0.0, 1.0)
    if sunset:
        skyTop = vec3(0.15, 0.30, 0.60)
        skyBottom = vec3(0.85, 0.45, 0.30)
    else:
        skyTop = vec3(0.2, 0.4, 0.8)
        skyBottom = vec3(0.7, 0.85, 1.0)
    return mix(skyBottom, skyTop, screenRange)

def henyey_greenstein(cosTheta):
    g1 = 0.7
    denom1 = 1.0 + g1 * g1 - 2.0 * g1 * cosTheta
    forward = (1.0 - g1 * g1) / pow(denom1, 1.5)
    
    g2 = -0.2
    denom2 = 1.0 + g2 * g2 - 2.0 * g2 * cosTheta
    backward = (1.0 - g2 * g2) / pow(denom2, 1.5)
    
    return mix(forward, backward, 0.4) * 0.4

def octaves(p):
    f = 0.0
    amp = 0.5
    freq = 2.0
    for i in range(maxOctaves):
        noise_val = cnoise(p) # or snoise(p)
        f += amp * noise_val
        p *= freq
        freq += 0.20
        amp *= 0.5
    return f
```

**Edge Cases and Pitfalls:**
- **Low Density Optimization:** If density < 0.01, the step is skipped to save computation, but $t$ is still incremented [S1, unofficial].
- **Early Exit for Light:** If light transmittance drops below 0.05, the light march stops [S1, unofficial].
- **Early Exit for Main Ray:** If main transmittance drops below 0.01, the main march stops [S1, unofficial].
- **Max Distance Limit:** The main march stops if $t$ exceeds 20.0 [S1, unofficial].
- **WebGL 1 Limitation:** Dynamic loop bounds are not supported, so $STEP\_SIZE$ and $STEP\_SIZE\_LIGHT$ must be hardcoded constants [S2, unofficial].
- **Precision:** The Beer-Lambert law uses an additional multiplier of 1.5 in the exponent in the source implementation [S1, unofficial], which may deviate from standard physical definitions [G].

## Operating characteristics
- **Time Complexity:** $O(MAX\_STEPS \times MAX\_STEPS\_LIGHT)$ per fragment. With $MAX\_STEPS=50$ and $MAX\_STEPS\_LIGHT=5$, this is 250 noise evaluations and transmittance calculations per pixel [S1, unofficial].
- **Memory Complexity:** $O(1)$ for procedural noise. If using texture-based density, $O(textureWidth \times textureHeight \times 4)$ for the texture buffer [S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Quality can be increased by decreasing $STEP\_SIZE$ and $STEP\_SIZE\_LIGHT$, but this increases computational cost linearly [S2, unofficial].
- **Strengths:** Real-time performance on WebGL 1/2 hardware; physically-based single scattering approximation; procedural noise allows infinite resolution and easy animation [S1, S2, unofficial].
- **Limitations:** Single scattering approximation is less physically accurate than multiple scattering [S2, unofficial]; procedural noise may lack the detailed structure of texture-based volumes; WebGL 1 requires hardcoded step sizes [S2, unofficial].
- **When to use:** Real-time cloud rendering in games or interactive applications where performance is critical [S1, unofficial].
- **When not to use:** Offline rendering requiring high physical accuracy or complex multiple scattering effects [S2, unofficial].

## Background
This technique originates from real-time OpenGL/WebGL implementations of volumetric cloud rendering [S1, S2, unofficial]. It was popularized by demos and open-source projects such as `gmanishaa/volumetric-rendering` and `M4trixStrike/WebGL-Volumetric-clouds` [S1, S2, unofficial]. The method combines classic ray marching with single scattering approximation, using procedural noise for density and Henyey-Greenstein phase function for light scattering [S1, unofficial]. It is used in real-time graphics engines and web-based visualizations [S1, S2, unofficial].

## Sources
[S1] gmanishaa/volumetric-rendering, https://github.com/gmanishaa/volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
