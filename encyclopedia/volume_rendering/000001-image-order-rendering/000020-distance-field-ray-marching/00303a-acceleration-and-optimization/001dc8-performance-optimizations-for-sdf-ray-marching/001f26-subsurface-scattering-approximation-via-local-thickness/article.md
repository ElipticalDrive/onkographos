# Subsurface Scattering Approximation via Local Thickness

Also called: SSS thickness approximation, local thickness SSS

## Overview
This method approximates subsurface scattering by computing the local thickness of a surface along random inward hemispherical directions, then using the thickness to modulate a simplified subsurface scattering lighting term [S1, unofficial]. It differs from sibling methods like [Aggressive Ray Marching with SDF](2326-aggressive-ray-marching-with-sdf.md) and [SDF-Accelerated Volumetric Ray Marching](3681-sdf-accelerated-volumetric-ray-marching.md) by performing a per-pixel thickness probe rather than global acceleration or volumetric integration [S1, unofficial].

## How it works
**Inputs:** Surface position `pos`, surface normal `norm`, and a signed distance field query function `sceneSDF(p)` that returns a signed distance (negative inside, positive outside) [S1, unofficial].
**Outputs:** A scalar thickness value in the range $[0.0, 1.0]$ used to modulate lighting [S1, unofficial].
**Data Structures:** Scalar accumulators and hash-based random number generators [S1, unofficial].
**Numbered Steps:**
1. Initialize `thickness` accumulator to $0.0$ [S1, unofficial].
2. Loop $i$ from $0$ to `SSSThicknessSamples` - 1:
   a. Generate a random scalar `sampleLength` using a hash function: `sampleLength = Hash11(i) * SSSSampleDepth` [S1, unofficial].
   b. Generate a random direction `sampleDir` in the hemisphere defined by the inward normal `-norm` using a hash function: `sampleDir = GenerateSampleVector(-norm, i)` [S1, unofficial].
   c. Query the scene SDF at the probe endpoint: `dist = sceneSDF(pos + sampleDir * sampleLength).dist` [S1, unofficial].
   d. Add the probe length and the SDF distance to the accumulator: `thickness += sampleLength + dist` [S1, unofficial].
3. Average the accumulated thickness by multiplying by `SSSThicknessSamplesI` [S1, unofficial].
4. Clamp the result to $[0.0, 1.0]$ and return [S1, unofficial].
**Formulas:**
* `sampleLength = Hash11(i) * SSSSampleDepth` where `Hash11` is a 1D hash, `i` is the loop index, and `SSSSampleDepth` is the maximum probe length [S1, unofficial].
* `sampleDir = GenerateSampleVector(-norm, i)` where `norm` is the outward surface normal and `GenerateSampleVector` produces a unit vector in the inward hemisphere [S1, unofficial].
* `thickness += sampleLength + sceneSDF(pos + sampleDir * sampleLength).dist` where the SDF distance is added to the probe length to ensure positive contributions [S1, unofficial].
* `result = clamp(thickness * SSSThicknessSamplesI, 0.0, 1.0)` where `SSSThicknessSamplesI` averages the accumulation [S1, unofficial].
**Parameters:**
* `SSSSampleDepth` (float): Maximum length of the random hemispherical sampling vector into the object; default $1.0$ [S1, unofficial].
* `SSSThicknessSamples` (int): Number of samples to perform when calculating local thickness; default $32$ [S1, unofficial].
* `SSSThicknessSamplesI` (float): Inverse of `SSSThicknessSamples` ($1.0 / 32.0 = 0.03125$), used to average the accumulated thickness; default $0.03125$ [S1, unofficial].
**Edge Cases and Pitfalls:**
* Internal SDF values are negative, so the formula `sampleLength + dist` is critical; adding the sample length ensures the thickness contribution remains positive and prevents incorrect darkening [S1, unofficial].
* If the probe exits the object, `sceneSDF` returns a positive distance, which increases the thickness value and is handled by clamping to $[0.0, 1.0]$ [S1, unofficial].
* Floating-point accumulation may introduce precision errors for very large scenes, though 32 samples is typically stable [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_local_thickness(pos, norm, sceneSDF, SSSSampleDepth=1.0, SSSThicknessSamples=32):
    thickness = 0.0
    inv_samples = 1.0 / SSSThicknessSamples
    for i in range(SSSThicknessSamples):
        sample_length = hash11(i) * SSSSampleDepth
        sample_dir = generate_sample_vector(-norm, i)
        probe_pos = pos + (sample_dir * sample_length)
        sdf_dist = sceneSDF(probe_pos).dist
        thickness += sample_length + sdf_dist
    return clamp(thickness * inv_samples, 0.0, 1.0)

def hash11(p):
    p3 = fract(vec3(p) * 443.897)
    p3 += dot(p3, p3.yzx + 19.19)
    return fract((p3.x + p3.y) * p3.z)

def hash33(p3):
    p3 = fract(p3 * vec3(443.897, 441.423, 437.195))
    p3 += dot(p3, p3.yxz + 19.19)
    return fract((p3.xxy + p3.yxx) * p3.zyx)

def reflect_vector(v, n):
    return v - 2.0 * n * min(0.0, dot(v, n))

def generate_sample_vector(norm, i):
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
    return int(v) if isinstance(v, float) else [int(x) for x in v]

def normalize(v):
    length = sqrt(dot(v, v))
    return (v[0]/length, v[1]/length, v[2]/length)

def sqrt(x):
    return x ** 0.5
```

## Subtopics

### SSS Lighting Integration
The thickness value is typically used as a factor in a diffuse or subsurface scattering lighting equation, such as multiplying albedo or scattering color [G].

### Random Number Generation
The quality of the hash functions (`Hash11`, `Hash33`) affects the visual noise of the approximation, and the source uses Dave Hoskins' hash functions [S1, unofficial].

## Operating characteristics
* Time complexity is $O(N)$ where $N$ is `SSSThicknessSamples`, as each sample requires one SDF query and arithmetic operations [S1, unofficial].
* Memory complexity is $O(1)$ auxiliary space, as the algorithm only uses local scalars [G].
* Performance with $N=32$ is relatively expensive per pixel compared to simple diffuse shading but cheaper than full volumetric ray marching, making it suitable for real-time rendering if optimized [G].
* Strengths include simple implementation, compatibility with any SDF, and a plausible approximation of translucency [S1, unofficial].
* Limitations include approximation quality depending on sample count, noise with low samples, and lack of multiple scattering or wavelength-dependent effects [G].
* Use this method when a cheap approximation of subsurface scattering is needed for SDF-rendered objects and full volumetric methods are too expensive [S1, unofficial].
* Do not use it when high-fidelity SSS is required or when the object is very thin, as thickness may saturate [G].

## Background
Not given in the sources.

## Sources
[S1] cahekp/Raymarching, https://github.com/cahekp/Raymarching, fetched 2026-10-27 (unofficial)
