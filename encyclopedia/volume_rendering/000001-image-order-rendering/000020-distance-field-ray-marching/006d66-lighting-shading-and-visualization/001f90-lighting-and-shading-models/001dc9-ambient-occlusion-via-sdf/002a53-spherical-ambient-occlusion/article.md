# Spherical Ambient Occlusion

Also called: Hemisphere AO, Spherical Sampling AO

## Overview
Spherical Ambient Occlusion estimates ambient occlusion by sampling the Signed Distance Field (SDF) at points distributed uniformly on a sphere around the surface point, then combining these samples to compute an occlusion factor [S1, unofficial]. It differs from its parent, [Ambient Occlusion via SDF](1631-ambient-occlusion-via-sdf.md), which typically compares the SDF value along a single normal direction against an unoccluded expectation, by using a multi-directional spherical probe to capture occlusion from all sides [S1, unofficial].

## How it works
**Inputs:**
- `hitPos`: The 3D position on the surface where shading is computed.
- `normal`: The surface normal at `hitPos` (used to define the local coordinate system for the sphere, though the specific rotation logic is implicit in the sample generation relative to the normal).
- `_aoSamples`: Number of spherical samples (integer).
- `_aoKernelSize`: Radius of the AO sampling sphere (float).

**Outputs:**
- `ao`: The final ambient occlusion factor in the range [0, 1], where 0 is fully occluded (dark) and 1 is fully open (bright).

**Data Structures:**
- No complex data structures are required. The method operates on scalar values and simple 3D vectors.

**Parameters:**
- `_aoSamples`: Number of samples. Typical value: 8. Range: 0–32 [S1, unofficial].
- `_aoKernelSize`: Sampling radius. Typical value: 0.075. Range: 0.0–0.25 [S1, unofficial].

**Algorithm:**
The method generates sample points on a unit sphere using a golden-ratio-based spiral distribution to ensure uniform density and avoid clustering at the poles [S1, unofficial]. It then scales these points by `_aoKernelSize`, samples the SDF, and aggregates the results.

1. Initialize `ao = 0.0`.
2. Define constants:
   - $\phi = \pi (3 - \sqrt{5})$ [S1, unofficial].
   - $N = \text{_aoSamples}$ [S1, unofficial].
3. Loop $i$ from $0$ to $N - 1$:
   a. Calculate spherical coordinates for the $i$-th sample:
      - $p_y = 1 - \frac{i}{N} \cdot 2$ [S1, unofficial].
      - $r = \sqrt{1 - p_y^2}$ [S1, unofficial].
      - $\theta = \phi \cdot i$ [S1, unofficial].
      - $p_x = \cos(\theta) \cdot r$ [S1, unofficial].
      - $p_z = \sin(\theta) \cdot r$ [S1, unofficial].
   b. Construct the sample direction vector $\mathbf{p} = (p_x, p_y, p_z)$ [S1, unofficial].
   c. Transform $\mathbf{p}$ to world/local space aligned with the surface normal. *Note: The source code implies $\mathbf{p}$ is in a local coordinate system where the Y-axis is the normal. In implementation, this requires constructing a basis from the normal.* [G]
   d. Calculate the sample position: $\mathbf{samplePos} = \text{hitPos} + \mathbf{p} \cdot \text{_aoKernelSize}$ [S1, unofficial].
   e. Sample the SDF: $d = \text{SDF}(\mathbf{samplePos})$ [S1, unofficial].
   f. Accumulate the occlusion contribution:
      - $s = \text{saturate}(d / \text{_aoKernelSize})$ [S1, unofficial].
      - $\text{ao} += s$ [S1, unofficial].
4. Normalize the accumulated value:
   - $\text{ao} = \text{ao} / (N \cdot 0.5)$ [S1, unofficial].
5. Apply a non-linear transfer function to enhance contrast:
   - $\text{ao} = 1.0 - (1.0 - \text{ao})^2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_ambient_occlusion(hit_pos, normal, ao_samples, ao_kernel_size):
    if ao_samples == 0:
        return 1.0

    phi = 3.141592653589793 * (3.0 - (5.0 ** 0.5))
    ao_acc = 0.0

    # Construct a local coordinate system from the normal
    # Assuming normal is normalized
    # Create two tangent vectors orthogonal to normal
    # A simple robust method:
    if abs(normal.y) < 0.9999:
        up = Vector3(0.0, 1.0, 0.0)
    else:
        up = Vector3(1.0, 0.0, 0.0)
    
    tangent1 = normalize(cross(normal, up))
    tangent2 = cross(normal, tangent1)

    for i in range(ao_samples):
        # Calculate spherical coordinates
        py = 1.0 - (i / ao_samples) * 2.0
        r = (1.0 - py * py) ** 0.5
        theta = phi * i
        px = cos(theta) * r
        pz = sin(theta) * r

        # Construct sample vector in local space (Y is normal)
        sample_dir = tangent1 * px + Vector3(0.0, 1.0, 0.0) * py + tangent2 * pz

        # Sample position
        sample_pos = hit_pos + sample_dir * ao_kernel_size

        # Sample SDF
        sdf_val = sample_sdf(sample_pos)

        # Accumulate
        # saturate(x) = clamp(x, 0.0, 1.0)
        s = clamp(sdf_val / ao_kernel_size, 0.0, 1.0)
        ao_acc += s

    # Normalize
    ao = ao_acc / (ao_samples * 0.5)

    # Non-linear transfer
    ao = 1.0 - (1.0 - ao) ** 2.0

    return ao
```

**Edge Cases and Pitfalls:**
- **Zero Samples:** If `_aoSamples` is 0, the loop does not run, and the result should be 1.0 (fully open) to avoid division by zero or undefined behavior [G].
- **Precision:** The calculation of $r = \sqrt{1 - p_y^2}$ can suffer from precision issues if $p_y$ is very close to $\pm 1$. However, with uniform distribution, this is rare.
- **Boundary Conditions:** If the sampling sphere extends outside the valid SDF domain, the SDF sampling function must handle out-of-bounds queries gracefully (e.g., by returning a large positive value or clamping) [G].
- **Normal Alignment:** The sample distribution assumes the Y-axis of the local coordinate system aligns with the surface normal. Incorrect basis construction will lead to incorrect occlusion estimates [G].

## Subtopics

### Isosurface Ambient Occlusion
A shading technique that probes nearby directions to darken points enclosed by material, providing 3-D depth cues for crevices and folds [S2, unofficial]. It probes 12 nearby directions at each isosurface point and darkens points based on the probe results [S2, unofficial]. The parameter `ao_radius` specifies the radius in cells for probing nearby directions [S2, unofficial]. A key characteristic is that the AO effect only darkens points; it does not brighten them, and the output values stay within [0, 1] [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the number of samples (`_aoSamples`). Each sample requires one SDF query.
**Memory Complexity:** $O(1)$, as it only stores a few scalar values and vectors.
**Performance:** With a typical sample count of 8, the method is relatively fast. Increasing the sample count linearly increases the cost.
**Strengths:** Provides a good balance between performance and visual quality. The golden-ratio distribution ensures uniform sampling with few samples.
**Limitations:** The fixed kernel size may not adapt well to varying feature scales. It does not account for self-shadowing from distant geometry, only local occlusion.
**When to Use:** Suitable for real-time rendering where performance is critical but some ambient occlusion is desired.
**When Not to Use:** When high-quality, global occlusion is required, or when the scene has large-scale occluding structures.

## Background
Spherical Ambient Occlusion is a technique used in real-time graphics, particularly in Unity-based projects, to add depth and detail to surfaces defined by Signed Distance Fields [S1, unofficial]. It builds on the general concept of ambient occlusion by using a spherical sampling pattern to estimate how much of the hemisphere around a surface point is occluded by nearby geometry.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
[S2] feat(immersive): pre-integrated transfer function (single-field render_scene), https://github.com/ManuelBehrendt/Mera.jl/pull/269, fetched 2026-10-27 (unofficial)
