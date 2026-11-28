# Gradient-Based Volume Rendering with Isovalue Lookup

Also called: Gradient Magnitude Weighted Alpha, Isovalue-Gradient Volume Raycasting

## Overview
This method performs direct volume ray casting by sampling a 3D texture storing pre-computed integer isovalue and gradient magnitude data, allowing the fragment shader to sample surface properties directly without numerical differentiation [S1, unofficial]. It distinguishes itself from standard raycasting by weighting the transfer function opacity with the gradient magnitude to enhance edges and surfaces, and by using a lookup-based approach for lighting and visibility selection [S1, unofficial]. It builds on [Raycasting with Transfer Function and Shading](3773-raycasting-with-transfer-function-and-shading.md) by replacing on-the-fly gradient computation with pre-sampled gradient magnitudes and adding isovalue-based selection logic [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `u_ModelXYZToIsoValue`: A 3D texture storing integer isovalue (R channel) and gradient magnitude (G channel) [S1, unofficial].
- `u_IsoValueToColorOpacity`: A 2D texture storing color (RGB) and opacity (A) for normalized isovalue [S1, unofficial].
- `u_TexCoordToRayEndPoint`: A texture mapping screen coordinates to the back-face grid position of the ray [S1, unofficial].
- `v_gridPosition`: The ray start position in grid coordinates (varying from vertex shader) [S1, unofficial].
- `v_screenCoord`: The screen coordinate of the current fragment [S1, unofficial].
- Uniforms: `u_SamplingRate`, `u_GMagWeighting`, `u_OverallOpacity`, `u_AlphaCorrectionExponent`, `u_IsoMinMax`, `u_isovalueLightingRange`, `u_kA`, `u_kD`, `u_kS`, `u_n`, `u_Il`, `u_lightDir`, `u_viewDir`, `u_NonSelectedVisibilityMultiplier`, `u_DoRenderRay`, `u_RayRadius_M`, `u_GradientDelta`, `u_BoundingBoxNormalized`, `haloThreshold`, and `halo` [S1, unofficial].

**Outputs:**
- `outColor`: The final accumulated color and alpha for the fragment [S1, unofficial].

### Data Structures
- **Isovalue/Gradient Texture (`u_ModelXYZToIsoValue`)**: 3D texture where the R channel stores the integer isovalue and the G channel stores the integer gradient magnitude [S1, unofficial].
- **Transfer Function Texture (`u_IsoValueToColorOpacity`)**: 2D texture in RGBA format where RGB stores the color and A stores the opacity, with the x-coordinate corresponding to the normalized isovalue [S1, unofficial].

### Parameters
- `u_SamplingRate`: Step size in normalized grid coordinates [S1, unofficial].
- `u_GMagWeighting`: Exponent for gradient magnitude weighting in alpha calculation [S1, unofficial].
- `u_OverallOpacity`: Global opacity multiplier [S1, unofficial].
- `u_AlphaCorrectionExponent`: Exponent applied to transfer function opacity [S1, unofficial].
- `u_IsoMinMax`: Minimum and maximum isovalue range for valid data [S1, unofficial].
- `u_isovalueLightingRange`: Range of isovalue for which lighting is calculated [S1, unofficial].
- `u_kA`, `u_kD`, `u_kS`: Ambient, diffuse, and specular coefficients [S1, unofficial].
- `u_n`: Specular exponent [S1, unofficial].
- `u_Il`: Light intensity [S1, unofficial].
- `u_lightDir`: Direction of the light source [S1, unofficial].
- `u_viewDir`: Direction of the view vector [S1, unofficial].
- `u_NonSelectedVisibilityMultiplier`: Opacity multiplier for voxels outside the primary ray or lighting range [S1, unofficial].
- `u_DoRenderRay`: Boolean flag to enable ray-specific visibility adjustments [S1, unofficial].
- `u_RayRadius_M`: Radius of the primary ray for distance calculations [S1, unofficial].
- `u_GradientDelta`: Step size for gradient computation (used for normal estimation) [S1, unofficial].
- `haloThreshold`: Threshold for halo effect removal [S1, unofficial].
- `halo`: Halo color/intensity constant added to the final color [G].

### Algorithm Steps
1. **Initialization**: Initialize `accumulatedAlpha = 0.0`, `accumulatedColor = vec3(0.0)`, `accumulatedLength = 0.0`, and set `currentPos = v_gridPosition` [S1, unofficial].
2. **Ray Direction**: Sample `backfaceGridPos = texture(u_TexCoordToRayEndPoint, v_screenCoord).rgb`, compute `front2Back = backfaceGridPos - v_gridPosition`, compute `ray = normalize(front2Back)`, and compute `rayLength = length(front2Back)` [S1, unofficial].
3. **Step Size**: Compute `delta = u_SamplingRate` and `deltaRay = delta * ray` [S1, unofficial].
4. **Ray Marching Loop**: Loop `i` from 0 to `MAX_STEPS` (e.g., 1000 or 887) [S1, unofficial].
    1. **Sample Isovalue and Gradient**: Call `getNormalizedIsovalueAndGradientMagnitude(currentPos)` to get `isoN` and `gmagN` [S1, unofficial].
    2. **Skip Invalid Voxels**: If `isoN == -1.0` (isovalue outside valid range), update `currentPos += deltaRay`, `accumulatedLength += delta`, and `continue` to the next iteration [S1, unofficial].
    3. **Transfer Function Lookup**: Sample `isoRGBA = texture(u_IsoValueToColorOpacity, vec2(isoN, 0.5))` to get `alpha_TF` and `color_TF` [S1, unofficial].
    4. **Lighting Calculation**: If `isWithinLightingRange(isoN)` is true, compute `gradient = getGradientAt(currentPos, u_GradientDelta)`, compute `halfV = normalize(u_viewDir + u_lightDir)`, and compute `light = calculateLighting(gradient, gmagN, halfV)` [S1, unofficial]. Otherwise, set `light = vec2(0.1, 0.0)` [S1, unofficial].
    5. **Visibility Adjustment**: If `u_DoRenderRay` is true, compute `distFromRay = isWithinRayRange(currentPos)` [S1, unofficial]. If `distFromRay >= 0.0`, do nothing [S1, unofficial]. If `distFromRay < -(haloThreshold + 0.08)`, multiply `alpha_TF *= u_NonSelectedVisibilityMultiplier` [S1, unofficial]. If `distFromRay < -haloThreshold`, set `alpha_TF = 0.0` [S1, unofficial].
    6. **Alpha Computation**: Compute final alpha for the step using $\alpha_{new} = \min(1.0, \text{u\_OverallOpacity} \cdot \text{gmagN}^{\text{u\_GMagWeighting}} \cdot \alpha_{TF}^{\text{u\_AlphaCorrectionExponent}})$ [S1, unofficial].
    7. **Color Computation**: Compute `color = min(vec3(1.0), light.x * color_TF + light.y + halo)` [S1, unofficial].
    8. **Front-to-Back Accumulation**: Update `accumulatedAlpha += alpha_new` and `accumulatedColor += (1.0 - accumulatedAlpha) * color * alpha_new` [S1, unofficial]. If `accumulatedAlpha >= 1.0`, break loop [S1, unofficial].
    9. **Step Advance**: Update `currentPos += deltaRay` [S1, unofficial].
5. **Output**: Compute `outColor = vec4(min(vec3(1.0), accumulatedColor * accumulatedAlpha), min(1.0, accumulatedAlpha))` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
MAX_STEPS = 1000  # Typical value from source [S1, unofficial]

def main(v_gridPosition, v_screenCoord, u_SamplingRate, u_GMagWeighting, u_OverallOpacity, u_AlphaCorrectionExponent, u_IsoMinMax, u_isovalueLightingRange, u_kA, u_kD, u_kS, u_n, u_Il, u_lightDir, u_viewDir, u_NonSelectedVisibilityMultiplier, u_DoRenderRay, u_RayRadius_M, u_GradientDelta, haloThreshold, halo):
    accumulatedAlpha = 0.0
    accumulatedColor = vec3(0.0)
    accumulatedLength = 0.0
    currentPos = v_gridPosition
    
    backfaceGridPos = texture(u_TexCoordToRayEndPoint, v_screenCoord).rgb
    front2Back = backfaceGridPos - v_gridPosition
    ray = normalize(front2Back)
    rayLength = length(front2Back)
    
    delta = u_SamplingRate
    deltaRay = delta * ray
    
    for i in range(MAX_STEPS):
        isoAndGMag = getNormalizedIsovalueAndGradientMagnitude(currentPos, u_IsoMinMax)
        isoN = isoAndGMag.x
        gmagN = isoAndGMag.y
        
        if isoN == -1.0:
            currentPos += deltaRay
            accumulatedLength += delta
            continue
            
        isoRGBA = texture(u_IsoValueToColorOpacity, vec2(isoN, 0.5))
        alpha_TF = isoRGBA.a
        color_TF = isoRGBA.rgb
        
        if isWithinLightingRange(isoN, u_isovalueLightingRange):
            gradient = getGradientAt(currentPos, u_GradientDelta)
            halfV = normalize(u_viewDir + u_lightDir)
            light = calculateLighting(gradient, gmagN, halfV, u_kA, u_kD, u_kS, u_n, u_Il, u_lightDir)
        else:
            light = vec2(0.1, 0.0)
            
        if u_DoRenderRay:
            distFromRay = isWithinRayRange(currentPos, u_RayRadius_M)
            if distFromRay >= 0.0:
                pass
            elif distFromRay < -(haloThreshold + 0.08):
                alpha_TF *= u_NonSelectedVisibilityMultiplier
            elif distFromRay < -haloThreshold:
                alpha_TF = 0.0
                
        alpha_new = min(1.0, u_OverallOpacity * (gmagN ** u_GMagWeighting) * (alpha_TF ** u_AlphaCorrectionExponent))
        
        color = min(vec3(1.0), light.x * color_TF + light.y + halo)
        
        accumulatedAlpha += alpha_new
        accumulatedColor += (1.0 - accumulatedAlpha) * color * alpha_new
        
        if accumulatedAlpha >= 1.0:
            break
            
        currentPos += deltaRay
        
    outColor = vec4(min(vec3(1.0), accumulatedColor * accumulatedAlpha), min(1.0, accumulatedAlpha))
    return outColor

def getNormalizedIsovalueAndGradientMagnitude(modelPosition, u_IsoMinMax):
    isoAndGMag = texture(u_ModelXYZToIsoValue, modelPosition).rg
    iso = isoAndGMag.r
    gmag = isoAndGMag.g
    
    if not isWithinMinMax(iso, u_IsoMinMax):
        return vec2(-1.0, 0.0)
        
    isoN = float(iso) / 32736.0
    gmagN = float(gmag) / 32736.0
    return vec2(isoN, gmagN)

def isWithinMinMax(iso, u_IsoMinMax):
    # [G] Standard check against uniform bounds
    return u_IsoMinMax.x <= iso <= u_IsoMinMax.y

def getGradientAt(mPos, d):
    left = getNormalizedIsovalue(vec3(mPos.x - d.x, mPos.yz))
    right = getNormalizedIsovalue(vec3(mPos.x + d.x, mPos.yz))
    top = getNormalizedIsovalue(vec3(mPos.x, mPos.y + d.y, mPos.z))
    bottom = getNormalizedIsovalue(vec3(mPos.x, mPos.y - d.y, mPos.z))
    front = getNormalizedIsovalue(vec3(mPos.xy, mPos.z + d.z))
    back = getNormalizedIsovalue(vec3(mPos.xy, mPos.z - d.z))
    gradient = vec3(right - left, top - bottom, front - back)
    return gradient

def getNormalizedIsovalue(pos):
    # [G] Extracts R channel from u_ModelXYZToIsoValue and normalizes
    iso = texture(u_ModelXYZToIsoValue, pos).r
    return float(iso) / 32736.0

def calculateLighting(gradient, gmag, halfV, u_kA, u_kD, u_kS, u_n, u_Il, u_lightDir):
    gradientN = normalize(-gradient)
    specularExponent = u_n
    ambient = u_kA
    diffuse = u_Il * u_kD * abs(dot(u_lightDir, gradientN))
    specular = u_Il * u_kS * pow(abs(dot(halfV, gradientN)), specularExponent)
    return vec2(ambient + diffuse, specular)

def isWithinLightingRange(isoN, u_isovalueLightingRange):
    # [G] Check if isoN falls within lighting range
    return u_isovalueLightingRange.x <= isoN <= u_isovalueLightingRange.y

def isWithinRayRange(currentPos, u_RayRadius_M):
    # [G] Implementation not provided in sources; typically computes distance from ray center
    return 0.0
```

### Edge Cases and Pitfalls
- **Invalid Isovalue**: If the isovalue is outside the valid range (`u_IsoMinMax`), the function returns `-1.0` for isovalue, causing the raycasting loop to skip the step without accumulating color or alpha [S1, unofficial].
- **Gradient Computation**: The `getGradientAt` function recomputes the gradient using finite differences on the isovalue texture, despite the gradient magnitude being pre-stored, which adds computational cost [S1, unofficial].
- **Precision**: Normalization of isovalue and gradient magnitude uses division by `32736.0`, requiring integer-to-float casting before division [S1, unofficial].
- **Boundary Conditions**: `isWithinRayRange` and `isWithinLightingRange` implementations are not fully detailed in the source, requiring proper boundary checks to avoid out-of-bounds texture sampling [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: The method is efficient on GPU due to pre-computed gradient magnitudes, but the on-the-fly gradient normal computation adds overhead, and performance depends heavily on `u_SamplingRate` and `MAX_STEPS` [S1, unofficial].
- **Strengths**: Enhanced surface detail via gradient magnitude weighting, efficient isovalue lookup, and support for selection-based visibility [S1, unofficial].
- **Limitations**: Requires pre-computation of gradient magnitudes, gradient normal computation remains expensive, and quality is limited by the resolution of the isovalue texture [S1, unofficial].

## Background
This method originates from the `Volume-Rendering-WebGL` project by yngve-sk [S1, unofficial]. It represents a practical approach to interactive volume rendering in web browsers, leveraging pre-computed data to reduce runtime computation while maintaining visual quality through gradient-based shading and opacity weighting [S1, unofficial].

## Sources
[S1] yngve-sk/Volume-Rendering-WebGL, https://github.com/yngve-sk/Volume-Rendering-WebGL, fetched 2026-10-27 (unofficial)
