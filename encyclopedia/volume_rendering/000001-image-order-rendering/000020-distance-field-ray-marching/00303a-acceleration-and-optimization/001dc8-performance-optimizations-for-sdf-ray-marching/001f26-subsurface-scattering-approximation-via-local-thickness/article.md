# Subsurface Scattering Approximation via Local Thickness

Also called: SSS thickness approximation, local thickness SSS

## Overview
This method approximates subsurface scattering (SSS) by estimating the local thickness of a surface along random inward hemispherical directions, then using that thickness to modulate a simplified SSS lighting term [S1, unofficial]. It differs from sibling methods like [Aggressive Ray Marching with SDF](2326-aggressive-ray-marching-with-sdf.md) or [SDF-Accelerated Volumetric Ray Marching](3681-sdf-accelerated-volumetric-ray-marching.md) by focusing on a per-pixel thickness probe rather than global acceleration or volumetric integration, providing a cheap approximation of light transport through translucent materials [S1, unofficial].

## How it works
**Inputs:** Surface position `pos`, surface normal `norm`, SDF query function `sceneSDF(p)` which returns a signed distance (negative inside, positive outside) [S1, unofficial].
**Outputs:** A scalar thickness value in the range $[0.0, 1.0]$ used to modulate lighting [S1, unofficial].

**Data Structures:**
No complex data structures are required. The method relies on scalar accumulators and random number generation.

**Parameters:**
*   `SSSSampleDepth` (float): Maximum length of the random hemispherical sampling vector into the object. Default: $1.0$ [S1, unofficial].
*   `SSSThicknessSamples` (int): Number of samples to perform when calculating local thickness. Default: $32$ [S1, unofficial].
*   `SSSThicknessSamplesI` (float): Inverse of `SSSThicknessSamples` ($1.0 / 32.0 = 0.03125$), used to average the accumulated thickness. Default: $0.03125$ [S1, unofficial].

**Steps:**
1.  Initialize `thickness` accumulator to $0.0$ [S1, unofficial].
2.  Loop $i$ from $0$ to `SSSThicknessSamples` - 1:
    a. Generate a random scalar `sampleLength` using a hash function: `sampleLength = Hash11(i) * SSSSampleDepth` [S1, unofficial].
    b. Generate a random direction `sampleDir` in the hemisphere defined by the inward normal `-norm` using a hash function: `sampleDir = GenerateSampleVector(-norm, i)` [S1, unofficial].
    c. Query the scene SDF at the probe endpoint: `dist = sceneSDF(pos + sampleDir * sampleLength).dist` [S1, unofficial].
    d. Add the probe length and the SDF distance to the accumulator: `thickness += sampleLength + dist` [S1, unofficial].
3.  Average the accumulated thickness by multiplying by `SSSThicknessSamplesI` [S1, unofficial].
4.  Clamp the result to $[0.0, 1.0]$ and return [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_local_thickness(pos, norm, sceneSDF, SSSSampleDepth=1.0, SSSThicknessSamples=32):
    """
    Approximates subsurface scattering thickness by sampling inward hemispherical directions.
    
    Args:
        pos: vec3, surface position.
        norm: vec3, surface normal (pointing outward).
        sceneSDF: function, returns a struct/object with a 'dist' field (signed distance).
        SSSSampleDepth: float, max probe length.
        SSSThicknessSamples: int, number of samples.
        
    Returns:
        float, clamped thickness value in [0.0, 1.0].
    """
    thickness = 0.0
    inv_samples = 1.0 / SSSThicknessSamples
    
    for i in range(SSSThicknessSamples):
        # Generate random length in [0, SSSSampleDepth]
        sample_length = hash11(i) * SSSSampleDepth
        
        # Generate random direction in hemisphere defined by -norm
        sample_dir = generate_sample_vector(-norm, i)
        
        # Query SDF at probe endpoint
        # Note: sceneSDF returns a result object with a 'dist' attribute
        probe_pos = pos + (sample_dir * sample_length)
        sdf_result = sceneSDF(probe_pos)
        sdf_dist = sdf_result.dist
        
        # Add length and SDF distance to thickness
        # Pitfall: SDF dist is negative inside. Adding sample_length ensures positive contribution.
        thickness += sample_length + sdf_dist
        
    # Average and clamp
    result = thickness * inv_samples
    return clamp(result, 0.0, 1.0)

def hash11(p):
    """
    1D hash function from Dave Hoskins.
    """
    p3 = fract(vec3(p) * 443.897)
    p3 += dot(p3, p3.yzx + 19.19)
    return fract((p3.x + p3.y) * p3.z)

def hash33(p3):
    """
    3D hash function from Dave Hoskins.
    """
    p3 = fract(p3 * vec3(443.897, 441.423, 437.195))
    p3 += dot(p3, p3.yxz + 19.19)
    return fract((p3.xxy + p3.yxx) * p3.zyx)

def reflect_vector(v, n):
    """
    Reflect vector v along plane defined by normal n.
    """
    return v - 2.0 * n * min(0.0, dot(v, n))

def generate_sample_vector(norm, i):
    """
    Generate random vector in hemisphere defined by normal 'norm' and random value 'i'.
    """
    rand_dir = normalize(hash33(norm + i) - vec3(0.5))
    return reflect_vector(rand_dir, norm)

def clamp(x, min_val, max_val):
    return max(min_val, min(x, max_val))

def vec3(x, y, z):
    return (x, y, z)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def fract(v):
    return v - floor(v)

def floor(v):
    return int(v) if isinstance(v, float) else [int(x) for x in v] # Simplified for pseudocode

def normalize(v):
    length = sqrt(dot(v, v))
    return (v[0]/length, v[1]/length, v[2]/length)

def sqrt(x):
    return x ** 0.5
```

**Edge Cases and Pitfalls:**
*   **Internal SDF Values:** The SDF returns negative values inside the surface. The formula `sampleLength + dist` is critical; if `dist` is negative, it reduces the contribution, but `sampleLength` ensures the total remains positive, preventing incorrect darkening [S1, unofficial].
*   **Boundary Conditions:** If the probe exits the object, `sceneSDF` returns a positive distance. This increases the thickness value, which may not be physically accurate for thin surfaces but is handled by the clamping to $[0.0, 1.0]$ [S1, unofficial].
*   **Precision:** Using floating-point arithmetic for accumulation may introduce precision errors for very large scenes or many samples, though 32 samples is typically stable [G].

## Subtopics

### SSS Lighting Integration
The thickness value is typically used as a factor in a diffuse or subsurface scattering lighting equation (e.g., multiplying albedo or scattering color). This is not defined in the source but is the standard application [G].

### Random Number Generation
The quality of the hash functions (`Hash11`, `Hash33`) affects the visual noise of the approximation. The source uses Dave Hoskins' hash functions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is `SSSThicknessSamples`. Each sample requires one SDF query and some arithmetic operations [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space. No buffers are allocated [S1, unofficial].
*   **Performance:** With $N=32$, this is relatively expensive per pixel compared to simple diffuse shading, but cheaper than full volumetric ray marching. It is suitable for real-time rendering if optimized or reduced [G].
*   **Strengths:** Simple to implement, works with any SDF, provides a plausible approximation of translucency [S1, unofficial].
*   **Limitations:** Approximation quality depends on sample count; noisy with low samples; does not account for multiple scattering or wavelength-dependent effects [G].
*   **When to use:** When a cheap approximation of subsurface scattering is needed for SDF-rendered objects and full volumetric methods are too expensive [S1, unofficial].
*   **When not to use:** When high-fidelity SSS is required, or when the object is very thin (thickness may saturate) [G].

## Background
This method originates from real-time raymarching shaders, specifically shared in the context of ShaderToy and GitHub repositories focusing on SDF raymarching techniques [S1, unofficial]. It is a practical adaptation of thickness-based subsurface scattering approximations for interactive graphics [S1, unofficial].

## Sources
[S1] cahekp/Raymarching, https://github.com/cahekp/Raymarching, fetched 2026-10-27 (unofficial)
