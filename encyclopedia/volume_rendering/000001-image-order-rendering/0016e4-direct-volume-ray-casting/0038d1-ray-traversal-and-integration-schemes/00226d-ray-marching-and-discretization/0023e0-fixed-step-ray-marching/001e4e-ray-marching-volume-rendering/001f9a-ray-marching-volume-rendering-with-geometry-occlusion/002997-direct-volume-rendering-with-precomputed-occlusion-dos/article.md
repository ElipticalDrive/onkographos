# Direct Volume Rendering with Precomputed Occlusion (DOS)

Also called: Direct Volume Rendering with Occlusion

## Overview
Direct Volume Rendering with Precomputed Occlusion (DOS) is a single-pass, image-order ray marching technique that approximates volumetric attenuation from behind the current sample using precomputed 2D occlusion maps [S1, unofficial]. It solves the problem of self-occlusion in volume rendering without requiring full volumetric integration or multiple rendering passes [S1, unofficial]. Unlike standard front-to-back ray casting which integrates opacity along the ray, DOS decouples the occlusion estimation from the ray step, using spatial averaging of a pre-rendered occlusion texture to modulate the current slice's contribution [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `uVolume`: A 3D texture (`sampler3D`) storing volume density or scalar values [S1, unofficial].
- `uTransferFunction`: A 2D texture (`sampler2D`) mapping volume values to RGBA color and opacity [S1, unofficial].
- `uColor`: A 2D texture (`sampler2D`) storing the accumulated color and alpha from previous slices or frames [S1, unofficial].
- `uOcclusion`: A 2D texture (`sampler2D`) storing precomputed occlusion values [S1, unofficial].
- `uOcclusionSamples`: A 2D texture (`sampler2D`) storing 2D offset vectors used to sample the occlusion map [S1, unofficial].
- `uExtinction`: A float constant representing the extinction coefficient [S1, unofficial].
- `uSliceDistance`: A float constant representing the distance between volume slices or the step size along the ray [S1, unofficial].
- `uOcclusionScale`: A vec2 constant scaling the occlusion sample offsets [S1, unofficial].
- `uOcclusionSamplesCount`: A uint constant specifying the number of samples taken from the occlusion map [S1, unofficial].

**Outputs:**
- `oColor`: A `vec4` output at location 0, storing the accumulated color and alpha [S1, unofficial].
- `oOcclusion`: A `float` output at location 1, storing the updated occlusion value for the current pixel [S1, unofficial].

### Data Structures
The method relies on two render targets bound to the fragment shader outputs: a color buffer storing `vec4` per pixel, initialized to `vec4(0)` by a reset shader, and an occlusion buffer storing `float` per pixel, initialized to `1.0` [S1, unofficial]. The vertex shader renders a full-screen triangle using hardcoded vertices `(-1, -1)`, `(3, -1)`, and `(-1, 3)` to ensure the fragment shader processes every pixel in the viewport [S1, unofficial]. It passes a normalized 2D position `vPosition2D` (in `[0, 1]` range) and the corresponding 3D world position `vPosition3D` to the fragment shader [S1, unofficial].

### Algorithm Steps
1. **Initialization**: A reset shader clears the output buffers by setting `oColor` to `vec4(0)` and `oOcclusion` to `1.0` [S1, unofficial].
2. **Boundary Check**: In the main shader, check if `vPosition3D` is outside the volume bounds `[0, 1]`. If `any(greaterThan(vPosition3D, vec3(1))) || any(lessThan(vPosition3D, vec3(0)))`, output the previous color and occlusion unchanged and terminate [S1, unofficial].
3. **Volume Sampling**: Sample the volume value at `vPosition3D` using `uVolume`, then map this value through the transfer function `uTransferFunction` to get `transferSample` (a `vec4` with RGB color and alpha) [S1, unofficial].
4. **Extinction Calculation**: Calculate the extinction for the current step: $extinction = transferSample.a \times uExtinction$ [S1, unofficial].
5. **Alpha Contribution**: Calculate the alpha contribution for the current slice: $\alpha = 1.0 - \exp(-extinction \times uSliceDistance)$ [S1, unofficial].
6. **Occlusion Update**: Calculate the new occlusion value by sampling `uOcclusion` at multiple offsets defined in `uOcclusionSamples`, averaging the results, and multiplying by the exponential attenuation of the current slice: $oOcclusion = \left( \frac{1}{N} \sum_{i=0}^{N-1} texture(uOcclusion, vPosition2D + offset_i \times scale).r \right) \times \exp(-extinction \times uSliceDistance)$, where $N = uOcclusionSamplesCount$ and $offset_i$ is the $i$-th offset vector from `uOcclusionSamples` scaled by `uOcclusionScale` [S1, unofficial].
7. **Color Blending**: Calculate the color contribution using the previous occlusion value (`prevOcclusion`, sampled from `uOcclusion` at `vPosition2D`): $color = transferSample.rgb \times prevOcclusion \times \alpha$. Update the output color using front-to-back blending: $oColor.rgb = prevColor.rgb + color \times (1.0 - prevColor.a)$ and $oColor.a = \min(prevColor.a + \alpha, 1.0)$, where `prevColor` is sampled from `uColor` at `vPosition2D` [S1, unofficial].
8. **Final Composite**: After all slices are processed, a final composite shader mixes the accumulated color with a white background: $oColor = mix(vec4(1), vec4(color.rgb, 1), color.a)$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms/Constants (defined at shader compile time)
uVolume = None              # sampler3D
uTransferFunction = None    # sampler2D
uColor = None               # sampler2D
uOcclusion = None           # sampler2D
uOcclusionSamples = None    # sampler2D
uExtinction = 1.0           # float
uSliceDistance = 1.0        # float
uOcclusionScale = (0.0, 0.0) # vec2
uOcclusionSamplesCount = 4  # uint

# Outputs
oColor = (0.0, 0.0, 0.0, 0.0) # vec4
oOcclusion = 1.0              # float

def fragment_shader(vPosition2D, vPosition3D):
    # Step 2: Boundary Check
    if any(vPosition3D > 1.0) or any(vPosition3D < 0.0):
        prev_color = texture(uColor, vPosition2D)
        prev_occlusion = texture(uOcclusion, vPosition2D).r
        oColor = prev_color
        oOcclusion = prev_occlusion
        return

    # Step 3: Volume Sampling
    volume_value = texture(uVolume, vPosition3D).r
    transfer_sample = texture(uTransferFunction, vec2(volume_value, 0.5))

    # Step 4: Extinction Calculation
    extinction = transfer_sample.a * uExtinction

    # Step 5: Alpha Contribution
    alpha = 1.0 - exp(-extinction * uSliceDistance)

    # Step 6: Occlusion Update
    occlusion_sum = 0.0
    for i in range(uOcclusionSamplesCount):
        # Retrieve offset from uOcclusionSamples texture
        # Note: Assumes offsets are stored sequentially in the texture [G]
        offset = texture(uOcclusionSamples, vec2(i / uOcclusionSamplesCount, 0.0)).xy
        sample_pos = vPosition2D + offset * uOcclusionScale
        occlusion_sum += texture(uOcclusion, sample_pos).r

    avg_occlusion = occlusion_sum / uOcclusionSamplesCount
    oOcclusion = avg_occlusion * exp(-extinction * uSliceDistance)

    # Step 7: Color Blending
    prev_color = texture(uColor, vPosition2D)
    prev_occlusion = texture(uOcclusion, vPosition2D).r

    color = transfer_sample.rgb * prev_occlusion * alpha

    oColor.rgb = prev_color.rgb + color * (1.0 - prev_color.a)
    oColor.a = min(prev_color.a + alpha, 1.0)

def final_composite(color):
    return mix(vec4(1.0), vec4(color.rgb, 1.0), color.a)
```

### Parameters
- `uExtinction`: Float constant representing the extinction coefficient [S1, unofficial].
- `uSliceDistance`: Float constant representing the distance between volume slices or step size along the ray [S1, unofficial].
- `uOcclusionScale`: Vec2 constant scaling the occlusion sample offsets [S1, unofficial].
- `uOcclusionSamplesCount`: Uint constant specifying the number of samples taken from the occlusion map [S1, unofficial].

### Edge Cases and Pitfalls
- **Boundary Conditions**: The method explicitly checks for volume bounds `[0, 1]` to avoid sampling outside the volume texture [S1, unofficial].
- **Precision**: The use of `exp()` and floating-point accumulation can lead to precision issues in high-density volumes [G].

## Subtopics

### Ray Marching Volume Rendering with Geometry Occlusion
The parent method that provides the geometry occlusion data used by DOS. It renders opaque geometry to occlude the volume, which DOS then uses to adjust sampling rates and attenuation. [Ray Marching Volume Rendering with Geometry Occlusion](2066-ray-marching-volume-rendering-with-geometry-occlusion.md)

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Not given in the sources.
- **Strengths**: Real-time capability due to single-pass per slice design and avoidance of full volumetric integration [S1, unofficial].
- **Limitations**: Approximation of occlusion may lead to artifacts in complex volumes; dependent on the quality of precomputed occlusion maps [G].

## Background
Direct Volume Rendering with Precomputed Occlusion (DOS) is a volume rendering technique described in the `terier/vpt` repository [S1, unofficial]. It performs ray marching in a single pass per slice, using precomputed 2D occlusion maps to approximate the attenuation of light from rays behind the current sample, avoiding the need for full volumetric integration or multiple passes for occlusion [S1, unofficial].

## Sources
[S1] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
