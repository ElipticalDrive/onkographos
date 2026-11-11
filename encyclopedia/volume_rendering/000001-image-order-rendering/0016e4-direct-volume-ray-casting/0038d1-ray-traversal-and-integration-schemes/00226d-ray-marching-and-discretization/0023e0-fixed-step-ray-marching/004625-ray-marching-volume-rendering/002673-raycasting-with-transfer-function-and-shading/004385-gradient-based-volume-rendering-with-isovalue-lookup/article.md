# Gradient-Based Volume Rendering with Isovalue Lookup

Also called: Gradient Magnitude Weighted Alpha, Isovalue-Gradient Volume Raycasting

## Overview
This method performs direct volume ray casting by storing pre-computed integer isovalue and gradient magnitude data in a 3D texture, allowing the fragment shader to sample surface properties directly without numerical differentiation. It distinguishes itself from standard raycasting by weighting the transfer function opacity with the gradient magnitude, which enhances edges and surfaces, and by using a lookup-based approach for lighting and visibility selection. It builds on [Raycasting with Transfer Function and Shading](3773-raycasting-with-transfer-function-and-shading.md) by replacing on-the-fly gradient computation with pre-sampled gradient magnitudes and adding isovalue-based selection logic.

## How it works
### Inputs and Outputs
**Inputs:**
- `u_ModelXYZToIsoValue`: A 3D texture storing integer isovalue (R channel) and gradient magnitude (G channel). Format is likely `RG16I` or similar integer format.
- `u_IsoValueToColorOpacity`: A 2D texture storing color (RGB) and opacity (A) for normalized isovalue.
- `u_TexCoordToRayEndPoint`: A texture mapping screen coordinates to the back-face grid position of the ray.
- `v_gridPosition`: The ray start position in grid coordinates (varying from vertex shader).
- `v_screenCoord`: The screen coordinate of the current fragment.
- Uniforms: `u_SamplingRate`, `u_GMagWeighting`, `u_OverallOpacity`, `u_AlphaCorrectionExponent`, `u_IsoMinMax`, `u_isovalueLightingRange`, `u_kA`, `u_kD`, `u_kS`, `u_n`, `u_Il`, `u_lightDir`, `u_viewDir`, `u_NonSelectedVisibilityMultiplier`, `u_DoRenderRay`, `u_RayRadius_M`, `u_GradientDelta`, `u_BoundingBoxNormalized`, `haloThreshold`.

**Outputs:**
- `outColor`: The final accumulated color and alpha for the fragment.

### Data Structures
- **Isovalue/Gradient Texture (`u_ModelXYZToIsoValue`)**: 3D texture. Each texel contains two integer channels. Channel R stores the integer isovalue. Channel G stores the integer gradient magnitude.
- **Transfer Function Texture (`u_IsoValueToColorOpacity`)**: 2D texture. Each texel contains RGBA. RGB stores the color, A stores the opacity. The x-coordinate corresponds to the normalized isovalue.

### Parameters
- `u_SamplingRate`: Step size in normalized grid coordinates.
- `u_GMagWeighting`: Exponent for gradient magnitude weighting in alpha calculation.
- `u_OverallOpacity`: Global opacity multiplier.
- `u_AlphaCorrectionExponent`: Exponent applied to transfer function opacity.
- `u_IsoMinMax`: Minimum and maximum isovalue range for valid data.
- `u_isovalueLightingRange`: Range of isovalue for which lighting is calculated.
- `u_kA`, `u_kD`, `u_kS`: Ambient, diffuse, and specular coefficients.
- `u_n`: Specular exponent.
- `u_Il`: Light intensity.
- `u_lightDir`: Direction of the light source.
- `u_viewDir`: Direction of the view vector.
- `u_NonSelectedVisibilityMultiplier`: Opacity multiplier for voxels outside the primary ray or lighting range.
- `u_DoRenderRay`: Boolean flag to enable ray-specific visibility adjustments.
- `u_RayRadius_M`: Radius of the primary ray for distance calculations.
- `u_GradientDelta`: Step size for gradient computation (used for normal estimation).
- `haloThreshold`: Threshold for halo effect removal.

### Algorithm Steps
1. **Initialization**: Initialize `accumulatedAlpha = 0.0`, `accumulatedColor = vec3(0.0)`, `accumulatedLength = 0.0`. Set `currentPos = v_gridPosition`.
2. **Ray Direction**: Sample `backfaceGridPos = texture(u_TexCoordToRayEndPoint, v_screenCoord).rgb`. Compute `front2Back = backfaceGridPos - v_gridPosition`. Compute `ray = normalize(front2Back)`. Compute `rayLength = length(front2Back)`.
3. **Step Size**: Compute `delta = u_SamplingRate`. Compute `deltaRay = delta * ray`.
4. **Ray Marching Loop**: Loop `i` from 0 to `MAX_STEPS` (e.g., 1000 or 887):
    1. **Sample Isovalue and Gradient**: Call `getNormalizedIsovalueAndGradientMagnitude(currentPos)` to get `isoN` and `gmagN`.
    2. **Skip Invalid Voxels**: If `isoN == -1.0` (isovalue outside valid range), update `currentPos += deltaRay`, `accumulatedLength += delta`, and `continue` to next iteration.
    3. **Transfer Function Lookup**: Sample `isoRGBA = texture(u_IsoValueToColorOpacity, vec2(isoN, 0.5))`. Get `alpha_TF = isoRGBA.a` and `color_TF = isoRGBA.rgb`.
    4. **Lighting Calculation**:
        - If `isWithinLightingRange(isoN)` is true:
            - Compute gradient normal: `gradient = getGradientAt(currentPos, u_GradientDelta)`.
            - Compute half vector: `halfV = normalize(u_viewDir + u_lightDir)`.
            - Compute lighting: `light = calculateLighting(gradient, gmagN, halfV)`.
        - Else: `light = vec2(0.1, 0.0)` (default ambient/diffuse).
    5. **Visibility Adjustment**: If `u_DoRenderRay` is true:
        - Compute `distFromRay = isWithinRayRange(currentPos)`.
        - If `distFromRay >= 0.0`, do nothing.
        - If `distFromRay < -(haloThreshold + 0.08)`, `alpha_TF *= u_NonSelectedVisibilityMultiplier`.
        - If `distFromRay < -haloThreshold`, `alpha_TF = 0.0`.
    6. **Alpha Computation**: Compute final alpha for the step:
       $$ \alpha_{new} = \min(1.0, \text{u\_OverallOpacity} \cdot \text{gmagN}^{\text{u\_GMagWeighting}} \cdot \alpha_{TF}^{\text{u\_AlphaCorrectionExponent}}) $$
    7. **Color Computation**:
       $$ \text{color} = \min(1.0, \text{light.x} \cdot \text{color\_TF} \cdot \text{light.y} + \text{halo}) $$
       *(Note: The source code snippet shows `light.x * isoRGBA.rgb + light.y + halo`. `light.x` is ambient+diffuse, `light.y` is specular. `halo` is likely a constant or uniform.)*
    8. **Front-to-Back Accumulation**:
       $$ \text{accumulatedAlpha} += \alpha_{new} $$
       $$ \text{accumulatedColor} += (1.0 - \text{accumulatedAlpha}) \cdot \text{color} \cdot \alpha_{new} $$
       If `accumulatedAlpha >= 1.0`, break loop.
    9. **Step Advance**: `currentPos += deltaRay`.
5. **Output**:
   $$ \text{outColor} = \text{vec4}(\min(1.0, \text{accumulatedColor} \cdot \text{accumulatedAlpha}), \min(1.0, \text{accumulatedAlpha})) $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper functions defined in shader or CPU
def getNormalizedIsovalueAndGradientMagnitude(modelPosition):
    # Sample 3D texture u_ModelXYZToIsoValue
    isoAndGMag = texture(u_ModelXYZToIsoValue, modelPosition).rg
    iso = isoAndGMag.r
    gmag = isoAndGMag.g
    
    if not isWithinMinMax(iso, u_IsoMinMax):
        return vec2(-1.0, 0.0)
    
    isoN = float(iso) / 32736.0
    gmagN = float(gmag) / 32736.0
    return vec2(isoN, gmagN)

def getGradientAt(mPos, d):
    # Compute gradient using central differences on isovalue texture
    # Note: This function samples the isovalue texture again, not the pre-stored gradient magnitude
    left = getNormalizedIsovalue(vec3(mPos.x - d.x, mPos.yz))
    right = getNormalizedIsovalue(vec3(mPos.x + d.x, mPos.yz))
    top = getNormalizedIsovalue(vec3(mPos.x, mPos.y + d.y, mPos.z))
    bottom = getNormalizedIsovalue(vec3(mPos.x, mPos.y - d.y, mPos.z))
    front = getNormalizedIsovalue(vec3(mPos.xy, mPos.z + d.z))
    back = getNormalizedIsovalue(vec3(mPos.xy, mPos.z - d.z))
    
    gradient = vec3(right - left, top - bottom, front - back)
    return gradient

def calculateLighting(gradient, gmag, halfV):
    gradientN = normalize(-gradient)
    specularExponent = u_n
    ambient = u_kA
    diffuse = u_Il * u_kD * abs(dot(u_lightDir, gradientN))
    specular = u_Il * u_kS * pow(abs(dot(halfV, gradientN)), specularExponent)
    return vec2(ambient + diffuse, specular)

def main(v_gridPosition, v_screenCoord):
    accumulatedAlpha = 0.0
    accumulatedColor = vec3(0.0)
    currentPos = v_gridPosition
    
    # Ray setup
    backfaceGridPos = texture(u_TexCoordToRayEndPoint, v_screenCoord).rgb
    front2Back = backfaceGridPos - v_gridPosition
    ray = normalize(front2Back)
    rayLength = length(front2Back)
    
    delta = u_SamplingRate
    deltaRay = delta * ray
    
    for i in range(MAX_STEPS):
        # Sample isovalue and gradient magnitude
        isoAndGMag = getNormalizedIsovalueAndGradientMagnitude(currentPos)
        isoN = isoAndGMag.x
        gmagN = isoAndGMag.y
        
        # Skip invalid voxels
        if isoN == -1.0:
            currentPos += deltaRay
            continue
            
        # Transfer function lookup
        isoRGBA = texture(u_IsoValueToColorOpacity, vec2(isoN, 0.5))
        alpha_TF = isoRGBA.a
        color_TF = isoRGBA.rgb
        
        # Lighting
        if isWithinLightingRange(isoN):
            gradient = getGradientAt(currentPos, u_GradientDelta)
            halfV = normalize(u_viewDir + u_lightDir)
            light = calculateLighting(gradient, gmagN, halfV)
        else:
            light = vec2(0.1, 0.0)
            
        # Visibility adjustment
        if u_DoRenderRay:
            distFromRay = isWithinRayRange(currentPos) # Implementation of isWithinRayRange not provided in source
            if distFromRay < -u_RayRadius_M - haloThreshold:
                alpha_TF *= u_NonSelectedVisibilityMultiplier
            if distFromRay < -u_RayRadius_M:
                alpha_TF = 0.0
                
        # Alpha weighting
        alpha_new = min(1.0, u_OverallOpacity * pow(gmagN, u_GMagWeighting) * pow(alpha_TF, u_AlphaCorrectionExponent))
        
        # Color computation
        # Note: light.x is ambient+diffuse, light.y is specular. 
        # Source code: min(vec3(1.0), light.x * isoRGBA.rgb + light.y + halo)
        # Assuming halo is a vec3 or scalar added to RGB. 
        # If light.y is scalar, it must be broadcast or halo handles it.
        # Based on source: light.x * color + light.y + halo
        color = min(vec3(1.0), light.x * color_TF + light.y + halo)
        
        # Accumulation
        accumulatedAlpha += alpha_new
        accumulatedColor += (1.0 - accumulatedAlpha) * color * alpha_new
        
        if accumulatedAlpha >= 1.0:
            break
            
        currentPos += deltaRay
        
    outColor = vec4(min(vec3(1.0), accumulatedColor * accumulatedAlpha), min(1.0, accumulatedAlpha))
    return outColor
```

### Edge Cases and Pitfalls
- **Invalid Isovalue**: If the isovalue is outside the valid range (`u_IsoMinMax`), the function returns `-1.0` for isovalue. The raycasting loop skips this step, accumulating no color or alpha, effectively treating it as transparent/empty.
- **Gradient Computation**: The `getGradientAt` function recomputes the gradient using finite differences on the isovalue texture, despite the gradient magnitude being pre-stored. This adds computational cost.
- **Precision**: Normalization of isovalue and gradient magnitude uses division by `32736.0`. Ensure integer types are cast to float before division.
- **Boundary Conditions**: `isWithinRayRange` and `isWithinLightingRange` implementations are not fully detailed in the source. Proper boundary checks are required to avoid out-of-bounds texture sampling.

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of steps per ray. Each step involves texture lookups and lighting calculations.
- **Memory Complexity**: $O(V)$ for the 3D isovalue/gradient texture and $O(I)$ for the 1D transfer function texture, where $V$ is volume size and $I$ is isovalue range.
- **Performance**: The method is efficient on GPU due to pre-computed gradient magnitudes, but the on-the-fly gradient normal computation adds overhead. Performance depends heavily on `u_SamplingRate` and `MAX_STEPS`.
- **Strengths**: Enhanced surface detail via gradient magnitude weighting; efficient isovalue lookup; supports selection-based visibility.
- **Limitations**: Requires pre-computation of gradient magnitudes; gradient normal computation is still expensive; limited by the resolution of the isovalue texture.

## Background
This method is derived from WebGL volume rendering implementations, specifically the `Volume-Rendering-WebGL` project by yngve-sk [S1, unofficial]. It represents a practical approach to interactive volume rendering in web browsers, leveraging pre-computed data to reduce runtime computation while maintaining visual quality through gradient-based shading and opacity weighting.

## Sources
[S1] yngve-sk/Volume-Rendering-WebGL, https://github.com/yngve-sk/Volume-Rendering-WebGL, fetched 2026-10-27 (unofficial)
