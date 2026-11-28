# Cloud Density Sampling

Also called: Base Cloud Density Generation, Low-Frequency Cloud Density, SampleCloudDensity, Schneider's Cloud Noise

## Overview
Cloud Density Sampling generates the base volumetric density of clouds by combining low-frequency 3D noise textures using a weighted sum (Fractal Brownian Motion, FBM) and remapping the result to control cloud shape and coverage [S1, unofficial]. It differs from sibling methods like [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-noise-density.md) by storing precomputed Perlin-Worley noise in multi-channel 3D textures rather than evaluating procedural noise functions at runtime, allowing for distinct control over vertical stratification and global distribution via texture channels [S1, unofficial].

## How it works
**Inputs:**
- `samplepoint`: `vec3`, world or texture space coordinate for density evaluation [S1, unofficial].
- `weather_data`: `vec4`, containing cloud coverage in the red channel (`weather_data.r`) and potentially other atmospheric parameters [S1, unofficial].
- `relativeHeight`: `float`, normalized height of the sample point within the atmosphere [S1, unofficial].
- `wind_direction`: `vec3`, default `(1.0, 0.0, 0.0)` [S1, unofficial].
- `cloud_speed`: `float`, default `10.0` [S1, unofficial].
- `cloud_top_offset`: `float`, default `5.0` [S1, unofficial].
- `Time`: `float`, global time variable [S1, unofficial].
- `iscCheap`: `bool`, if `true`, skips high-frequency sampling [S1, unofficial].
- `lowFrequencyTexture`: `sampler3D`, precomputed Perlin-Worley noise texture [S1, unofficial].
- `highFrequencyTexture`: `sampler3D`, precomputed high-frequency noise texture (used if `!iscCheap`) [S1, unofficial].
- `curlNoiseTexture`: `sampler2D`, 2D curl noise texture (used if `!iscCheap`) [S1, unofficial].

**Outputs:**
- `final_cloud`: `float`, the computed cloud density at `samplepoint` in the range $[0.0, 1.0]$ [S1, unofficial].

**Data Structures:**
- `lowFrequencyTexture`: A 3D texture where channels store specific noise components:
    - `R`: Base noise values [S1, unofficial].
    - `G`: Component weighted by 0.625 in FBM [S1, unofficial].
    - `B`: Component weighted by 0.250 in FBM [S1, unofficial].
    - `A`: Component weighted by 0.125 in FBM [S1, unofficial].
- `highFrequencyTexture`: A 3D texture with similar channel packing for high-frequency details:
    - `R`: Component weighted by 0.625 [S1, unofficial].
    - `G`: Component weighted by 0.250 [S1, unofficial].
    - `B`: Component weighted by 0.125 [S1, unofficial].

**Formulas:**
The core remapping function is defined as [S1, unofficial]:
$$ \text{Remap}(v, v_{min}, v_{max}, u_{min}, u_{max}) = u_{min} + \frac{v - v_{min}}{v_{max} - v_{min}}(u_{max} - u_{min}) $$

Low-frequency FBM is computed from the RGBA sample $N$ of the low-frequency texture [S1, unofficial]:
$$ F_{low} = 0.625 N_g + 0.250 N_b + 0.125 N_a $$
$$ D_{base} = \text{Remap}(N_r, -(1.0 - F_{low}), 1.0, 0.0, 1.0) $$

High-frequency FBM (if used) is computed from the RGBA sample $H$ of the high-frequency texture [S1, unofficial]:
$$ F_{high} = 0.625 H_r + 0.250 H_g + 0.125 H_b $$
$$ M_{high} = \text{mix}(F_{high}, 1.0 - F_{high}, \text{clamp}(relativeHeight \times 10.0, 0.0, 1.0)) $$

**Steps:**
1. **Wind Skew:** Adjust the sample point based on wind and height: $\mathbf{p}_{skewed} = \mathbf{p} + relativeHeight \times \mathbf{wind\_direction} \times cloud\_top\_offset \times 0.005$ [S1, unofficial].
2. **Time Movement:** Apply time-based drift: $\mathbf{p}_{final} = \mathbf{p}_{skewed} + (\mathbf{wind\_direction} + (0.0, 1.0, 0.0)) \times Time \times cloud\_speed \times 0.001$ [S1, unofficial].
3. **Low-Frequency Sampling:** Sample the low-frequency texture at $\mathbf{p}_{final}$: $N = \text{SampleLowFrequencyTexture}(\mathbf{p}_{final})$ [S1, unofficial].
4. **Low-Frequency FBM:** Compute the weighted sum: $F_{low} = N_g \times 0.625 + N_b \times 0.250 + N_a \times 0.125$ [S1, unofficial].
5. **Base Cloud Remap:** Remap the red channel to create the base cloud shape: $base\_cloud = \text{Remap}(N_r, -(1.0 - F_{low}), 1.0, 0.0, 1.0)$ [S1, unofficial].
6. **Height Gradient:** Compute the vertical attenuation factor: $g_h = \text{GetDensityHeightGradientForPoint}(\mathbf{p}, \text{weather\_data}, relativeHeight)$ [S1, unofficial].
7. **Apply Height Gradient:** $base\_cloud \leftarrow base\_cloud \times g_h$ [S1, unofficial].
8. **Coverage Mask:** Retrieve cloud coverage from weather data: $C = \text{weather\_data}.r$ [S1, unofficial].
9. **Apply Coverage:** Remap the base cloud against the coverage mask: $base\_cloud\_with\_coverage = \text{Remap}(base\_cloud, C, 1.0, 0.0, 1.0)$ [S1, unofficial].
10. **Clamp:** Ensure density is within bounds: $base\_cloud\_with\_coverage = \text{clamp}(base\_cloud\_with\_coverage, 0.0, 1.0)$ [S1, unofficial].
11. **Aesthetic Enhancement:** Boost density based on coverage: $base\_cloud\_with\_coverage \leftarrow base\_cloud\_with\_coverage \times C \times 4.0$ [S1, unofficial].
12. **High-Frequency Details (if `!iscCheap` and `base_cloud_with_coverage > 0.0`):**
    a. Sample curl noise: $C_{noise} = \text{SampleCurlNoiseTexture}(\mathbf{p}_{final}.xy)$ [S1, unofficial].
    b. Displace UVs: $\mathbf{p}_{final}.xy \leftarrow \mathbf{p}_{final}.xy + C_{noise}.xy \times (1.0 - relativeHeight)$ [S1, unofficial].
    c. Sample high-frequency texture: $H = \text{SampleHighFrequencyTexture}(\mathbf{p}_{final} \times 0.1)$ [S1, unofficial].
    d. Compute high-frequency modifier: $F_{high} = H_r \times 0.625 + H_g \times 0.250 + H_b \times 0.125$ and $M_{high} = \text{mix}(F_{high}, 1.0 - F_{high}, \text{clamp}(relativeHeight \times 10.0, 0.0, 1.0))$ [S1, unofficial].
    e. Clamp base cloud against modifier: $base\_cloud\_with\_coverage = \text{clamp}(base\_cloud\_with\_coverage, 0.2 \times M_{high}, 1.0)$ [S1, unofficial].
    f. Erode by remapping: $final\_cloud = \text{Remap}(base\_cloud\_with\_coverage, 0.2 \times M_{high}, 1.0, 0.0, 1.0)$ [S1, unofficial].
13. **Return:** If `iscCheap` or step 12 was skipped, return $base\_cloud\_with\_coverage$ [S1, unofficial]. Otherwise, return $final\_cloud$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, o): return vec3(self.x + o.x, self.y + o.y, self.z + o.z)
    def __mul__(self, s): return vec3(self.x * s, self.y * s, self.z * s)
    def __getitem__(self, i): return (self.x, self.y, self.z)[i]
    def __setitem__(self, i, v): setattr(self, 'xyz'[i], v)
    @property
    def xy(self): return vec2(self.x, self.y)

class vec2:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, o): return vec2(self.x + o.x, self.y + o.y)
    def __mul__(self, s): return vec2(self.x * s, self.y * s)
    def __getitem__(self, i): return (self.x, self.y)[i]
    def __setitem__(self, i, v): setattr(self, 'xy'[i], v)

def clamp(v, mn, mx): return max(mn, min(mx, v))
def mix(a, b, t): return a + (b - a) * t
def remap(v, v_min, v_max, u_min, u_max): return u_min + (v - v_min) / (v_max - v_min) * (u_max - u_min)

def GetDensityHeightGradientForPoint(p, weather_data, relativeHeight):
    # Height falloff signal to reduce density at cloud base [G]
    return clamp(1.0 - relativeHeight, 0.0, 1.0)

def SampleLowFrequencyTexture(p): return vec4(0.5, 0.5, 0.5, 0.5)
def SampleHighFrequencyTexture(p): return vec4(0.5, 0.5, 0.5, 0.5)
def SampleCurlNoiseTexture(p): return vec2(0.0, 0.0)

def SampleCloudDensity(samplepoint, weather_data, relativeHeight, wind_direction, cloud_speed, cloud_top_offset, Time, isCheap, lowFrequencyTexture, highFrequencyTexture, curlNoiseTexture):
    # 1. Wind Skew
    p = samplepoint + relativeHeight * wind_direction * cloud_top_offset * 0.005
    
    # 2. Time Movement
    p = p + (wind_direction + vec3(0.0, 1.0, 0.0)) * Time * cloud_speed * 0.001
    
    # 3. Low-Frequency Sampling
    N = SampleLowFrequencyTexture(p)
    
    # 4. Low-Frequency FBM
    F_low = N[1] * 0.625 + N[2] * 0.250 + N[3] * 0.125
    
    # 5. Base Cloud Remap
    v_min = -(1.0 - F_low)
    v_max = 1.0
    base_cloud = remap(N[0], v_min, v_max, 0.0, 1.0)
    
    # 6. Height Gradient
    g_h = GetDensityHeightGradientForPoint(samplepoint, weather_data, relativeHeight)
    
    # 7. Apply Height Gradient
    base_cloud *= g_h
    
    # 8. Coverage Mask
    C = weather_data[0]
    
    # 9. Apply Coverage
    v_min = C
    v_max = 1.0
    base_cloud_with_coverage = remap(base_cloud, v_min, v_max, 0.0, 1.0)
    
    # 10. Clamp
    base_cloud_with_coverage = clamp(base_cloud_with_coverage, 0.0, 1.0)
    
    # 11. Aesthetic Enhancement
    base_cloud_with_coverage *= C * 4.0
    
    # 12. High-Frequency Details
    if not isCheap and base_cloud_with_coverage > 0.0:
        # 12a. Sample Curl Noise
        curl_noise = SampleCurlNoiseTexture(p.xy)
        
        # 12b. Displace UVs
        p_xy = p.xy
        p_xy += curl_noise * (1.0 - relativeHeight)
        
        # 12c. Sample High-Frequency Texture
        H = SampleHighFrequencyTexture(vec3(p_xy.x, p_xy.y, p.z) * 0.1)
        
        # 12d. Compute High-Frequency Modifier
        F_high = H[0] * 0.625 + H[1] * 0.250 + H[2] * 0.125
        mix_factor = clamp(relativeHeight * 10.0, 0.0, 1.0)
        M_high = mix(F_high, 1.0 - F_high, mix_factor)
        
        # 12e. Clamp base cloud against modifier
        base_cloud_with_coverage = clamp(base_cloud_with_coverage, 0.2 * M_high, 1.0)
        
        # 12f. Erode by remapping
        v_min = 0.2 * M_high
        v_max = 1.0
        final_cloud = remap(base_cloud_with_coverage, v_min, v_max, 0.0, 1.0)
        
        return final_cloud
    else:
        return base_cloud_with_coverage
```

**Edge Cases and Pitfalls:**
- **Precision:** The remapping formula involves division by $(v_{max} - v_{min})$. If $v_{max} \approx v_{min}$, precision loss or division by zero may occur [G].
- **Degenerate Input:** If `weather_data.r` (coverage) is 0, the coverage mask remaps density to 0, and the aesthetic enhancement results in 0 [S1, unofficial].
- **Conditional Sampling:** High-frequency details are skipped if `base_cloud_with_coverage` is 0 [S1, unofficial].

## Subtopics

### [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-noise-density.md)
Generates billowy cloud shapes by synthesizing cloud density $\rho(\mathbf{x})$ procedurally using inverted Worley fractal Brownian motion (FBM), vertical gradient attenuation, and coverage thresholding. It differs from standard Perlin-based cloud noise by using the distance-to-nearest-cell metric of Worley noise, inverted to create bright, puffy "billowy" structures characteristic of cumulus clouds, rather than the smooth, turbulent gradients of Perlin noise. Choose this when procedural generation without pre-baked textures is required and billowy shapes are desired.

### [Full Cloud Density Function](5875-full-cloud-density-function.md)
The Full Cloud Density Function is a procedural method for computing the volumetric density of clouds by combining a vertical atmospheric profile with horizontal noise-based structures and a global coverage mask. It differs from sibling methods like [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-noise-density.md) by explicitly separating the vertical stratification (via a texture) from the horizontal structural modulation (via noise and a cubemap). This separation allows for distinct control over cloud layering and global distribution. Choose this when explicit control over vertical and horizontal cloud structures is needed.

### Cone Tracing Cloud Sampling
Samples cloud density along a cone by iterating through a pre-generated noise kernel, transforming positions to weather map coordinates, and summing densities [S1, unofficial]. It differs from standard point sampling by accumulating density along a cone path, which can improve visual quality at the cost of multiple texture samples [S1, unofficial]. Choose this when cone tracing is used for anti-aliasing or volumetric integration in cloud rendering [S1, unofficial].

### Two-Level Cloud Modeling
A procedural method to generate volumetric cloud shapes using two levels of detail: a low-frequency base shape and high-frequency detail/distortion, driven by a weather system or custom textures [S2, unofficial]. It differs from monolithic noise approaches by allowing distinct control over low-frequency form and high-frequency turbulence [S2, unofficial]. Steps include sampling a base 3D texture multiplied by a height signal, eroding with a coverage signal, eroding edges with a second 3D texture, and distorting the second texture with 2D curl noise [S2, unofficial]. To achieve whispy shapes at the base of clouds, invert the Worley noise before using it for erosion [S2, unofficial]. Choose this when a two-level detail approach is preferred for distinct control over base shape and turbulence [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Depends on the size of the pre-baked 3D textures (`lowFrequencyTexture`, `highFrequencyTexture`, `curlNoiseTexture`) [S1, unofficial].
- **Measured Performance:** Not given in the sources [S1, unofficial].
- **Scaling:** Not given in the sources [S1, unofficial].
- **Strengths:** The two-level detail approach allows for distinct control over low-frequency form and high-frequency turbulence [S2, unofficial].
- **Limitations:** Requires pre-computation of textures [S1, unofficial].
- **When to Use:** Not given in the sources [S1, unofficial].
- **When Not to Use:** Not given in the sources [S1, unofficial].

## Background
Developed for real-time volumetric cloud rendering in Horizon Zero Dawn [S2, unofficial], with implementation details shared in the realtime-volumetric-cloudscapes repository [S1, unofficial]. The use of multi-channel textures for FBM accelerates noise evaluation by baking the fractal sum into texture channels [S1, unofficial].

## Sources
[S1] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S2] The Real-time Volumetric Cloudscapes of Horizon, https://advances.realtimerendering.com/s2015/The%20Real-time%20Volumetric%20Cloudscapes%20of%20Horizon%20-%20Zero%20Dawn%20-%20ARTR.pdf, fetched 2026-10-29 (unofficial)
