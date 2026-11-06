# SDF-based Ambient Occlusion

Also called: Signed Distance Field AO, Distance Field Ambient Occlusion, Raymarched AO

## Overview
SDF-based Ambient Occlusion estimates local occlusion by sampling a Signed Distance Field (SDF) along the surface normal at increasing distances, weighting closer samples more heavily to simulate ambient light blockage [S1, S2, S3, S4, unofficial][S5]. It approximates ambient occlusion by using the distance field itself to simulate light occlusion based on object proximity, avoiding the unrealistic results of Screen Space Ambient Occlusion (SSAO) which relies on depth buffers and surface normals in screen space [S5]. This method is capable of producing visually pleasing results by leveraging the geometric accuracy of the SDF representation [S1, unofficial].

## How it works
**Inputs and Outputs**
The method takes a surface point $\mathbf{p}$, the surface normal $\mathbf{n}$ at $\mathbf{p}$, and an SDF query function $d(\mathbf{x})$ that returns the distance to the nearest geometry from any point $\mathbf{x}$ [S2, unofficial]. It outputs a scalar occlusion factor $AO \in [0, 1]$, where $1$ represents fully lit and $0$ represents fully occluded [S2, unofficial].

**Data Structures**
The SDF is stored as a 3D grid where each position stores the distance to the nearest object surface [S1, unofficial]. Negative values indicate the grid location is inside the object, and positive values indicate it is outside [S1, unofficial].

**Algorithm Steps**
1. Initialize the accumulated occlusion `occ` to $0.0$ and a weight factor `w` to $1.0$ [S2, unofficial].
2. Loop $i$ from $0$ to $N-1$, where $N$ is the number of samples [S2, unofficial].
3. Calculate the sample distance $len_i = 0.01 + 0.02 \cdot i^2$ [S2, unofficial].
4. Calculate the sample point $\mathbf{p}_i = \mathbf{p} + len_i \cdot \mathbf{n}$ [S2, unofficial].
5. Query the SDF to get the distance to the nearest geometry: $d_i = d(\mathbf{p}_i)$ [S2, unofficial].
6. Accumulate occlusion: `occ += (len_i - d_i) * w` [S2, unofficial].
7. Decay the weight for the next sample: `w *= 0.85` [S2, unofficial].
8. Return the final AO factor: `1.0 - clamp(0.6 * occ, 0.0, 1.0)` [S2, unofficial].

**Formulas**
The occlusion accumulation follows $AO = 1.0 - \text{clamp}(scale \cdot \sum_{i=0}^{N-1} (len_i - d_i) \cdot w_i, 0.0, 1.0)$, where $len_i = 0.01 + 0.02 \cdot i^2$ and $w_i = 0.85^i$ [S2, unofficial]. Alternative implementations use linear spacing $len_i = \frac{R}{N} \cdot i$ [S3, unofficial] or different final normalizations such as $\text{clamp}(1.0 - strength \cdot occ, 0.0, 1.0)$ [S4, unofficial].

**Parameters**
Typical parameter values found in implementations include a sample count $N$ of 4 to 8 [S2, S3, S4, unofficial], a maximum sampling distance $R$ of $0.25$ [S3, unofficial], weight decay factors of $0.85$ or $0.92$ [S2, S4, unofficial], and final scale/strength multipliers of $0.6$ or $3.0$ [S2, S4, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_ambient_occlusion(p, normal, sdf_query, num_samples=8, scale=0.6, decay=0.85):
    occ = 0.0
    weight = 1.0
    for i in range(num_samples):
        len_i = 0.01 + 0.02 * (i * i)
        sample_point = p + normal * len_i
        dist = sdf_query(sample_point)
        occ += (len_i - dist) * weight
        weight *= decay
    return 1.0 - max(0.0, min(scale * occ, 1.0))

def max(a, b):
    return a if a > b else b

def min(a, b):
    return a if a < b else b
```

**Edge Cases and Pitfalls**
Floating-point precision errors in SDF queries can lead to jittering or incorrect occlusion values, especially at small distances [G]. If sampling points go outside the defined SDF volume, undefined behavior or incorrect distances may occur [G]. If the normal is not normalized, sample points will be incorrectly spaced, leading to inaccurate AO [G]. If the point $\mathbf{p}$ is inside the geometry (negative SDF value), the standard AO heuristic may need adjustment, as the concept of occlusion from the outside differs [G].

## Operating characteristics
The time complexity is $O(N)$ per shading point, where $N$ is the number of samples [G]. The SDF query itself is typically $O(1)$ for dense grids or $O(\log M)$ for hierarchical structures [S1, unofficial]. The memory complexity depends on SDF storage, with a dense 3D grid requiring $O(V)$ memory, where $V$ is the volume of the grid [G]. The method is capable of producing visually pleasing results [S1, unofficial]. It is generally faster than full ray tracing but may be slower than screen-space techniques if the SDF query is expensive [G]. It provides geometrically accurate occlusion based on actual scene geometry, avoiding the artifacts of SSAO [S5]. It works well with voxel-based or implicit representations [G]. It is limited to the resolution and extent of the SDF [G]. It cannot account for occlusion from geometry not represented in the SDF [S1, unofficial]. Performance degrades with high sample counts or large sampling distances [G]. It should be used when geometric accuracy is important and the scene is represented by an SDF [G]. It should not be used when performance is critical and the SDF is expensive to query, or when the scene geometry is not available as an SDF [G].

## Background
The technique was introduced by Evans as a method based on signed distance fields where objects are embedded into a 3D grid [S1, unofficial]. Hoberock improved upon this method's image quality, but at a performance cost tradeoff [S1, unofficial]. Wright expanded on this approach, substituting Evans' ad hoc heuristic with cone tracing, approximating it by utilizing spheres of increasing radius to check for intersections [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S3] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S4] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S5] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
