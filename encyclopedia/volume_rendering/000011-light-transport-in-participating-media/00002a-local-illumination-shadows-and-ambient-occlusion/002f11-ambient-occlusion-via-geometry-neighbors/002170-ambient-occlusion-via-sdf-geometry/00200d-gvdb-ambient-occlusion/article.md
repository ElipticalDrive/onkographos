# GVDB Ambient Occlusion

Also called: GVDB AO, computeAmbientOcclusion

## Overview
GVDB Ambient Occlusion is a local illumination technique that approximates indirect lighting at a surface point within a volume by sampling a hemisphere of directions against a density field [S1, unofficial]. It distinguishes itself from sibling methods by supporting two distinct sampling modes—point sampling and ray casting—allowing a trade-off between computational cost and geometric accuracy in occlusion estimation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface position $\mathbf{p}$, surface normal $\mathbf{n}$, volume density field $V$, and parameters: `aoSamples` (integer, max 512), `aoRadius` (world-space float), `aoBias` (world-space float, typically $10^{-3}$), and `AMBIENT_OCCLUSION_MODE` (integer: 0 for point sampling, 1 for ray sampling) [S1, unofficial].
*   **Output:** A scalar occlusion factor $AO \in [0, 1]$, where 1.0 is fully lit and 0.0 is fully occluded [S1, unofficial].

**Data Structures:**
The method operates on a 3D volume grid. The density value $V(\mathbf{x})$ at any world position $\mathbf{x}$ is retrieved via trilinear interpolation or nearest-neighbor lookup depending on the volume implementation [S1, unofficial].

**Steps:**
1.  **Initialization:** If `aoSamples` is 0, return 1.0 [S1, unofficial].
2.  **Basis Construction:** Generate a random rotation vector $\mathbf{r}$ (precomputed or per-point noise). Construct the tangent $\mathbf{t}$ and bitangent $\mathbf{b}$ vectors to form a local coordinate system aligned with $\mathbf{n}$:
    $$ \mathbf{t} = \text{normalize}(\mathbf{r} - \mathbf{n} (\mathbf{r} \cdot \mathbf{n})) $$
    $$ \mathbf{b} = \mathbf{n} \times \mathbf{t} $$
    [S1, unofficial]
3.  **Bias Application:** Adjust the query position to prevent self-intersection artifacts (acne):
    $$ \mathbf{p}_{query} = \mathbf{p} + \mathbf{n} \cdot \text{aoBias} $$
    [S1, unofficial]
4.  **Hemisphere Sampling:** Initialize `occlusion_sum = 0.0`. Iterate $i$ from 0 to `aoSamples` - 1:
    a.  Generate a sample direction $\mathbf{d}_{local}$ on the upper hemisphere in the tangent space.
    b.  Transform $\mathbf{d}_{local}$ to world space:
        $$ \mathbf{d}_{world} = \mathbf{t} \cdot \mathbf{d}_{local.x} + \mathbf{b} \cdot \mathbf{d}_{local.y} + \mathbf{n} \cdot \mathbf{d}_{local.z} $$
    c.  **Mode 0 (Point Sampling):**
        i.  Compute query position: $\mathbf{q} = \mathbf{p}_{query} + \mathbf{d}_{world} \cdot \text{aoRadius}$.
        ii. Query volume: $val = V(\mathbf{q})$.
        iii. If $val \le \text{aoBias}$ (or density threshold), add 1.0 to `occlusion_sum`. [S1, unofficial]
    d.  **Mode 1 (Ray Sampling):**
        i.  Cast a ray from $\mathbf{p}_{query}$ in direction $\mathbf{d}_{world}$ with length `aoRadius`.
        ii. Find the first hit distance $dist$ where density exceeds a threshold.
        iii. If a hit is found, compute weight:
        $$ w = \text{smoothstep}(1.0, 0.0, \frac{\text{aoRadius}}{dist}) $$
        iv. Add $w$ to `occlusion_sum`. [S1, unofficial]
5.  **Normalization:**
    $$ AO = \frac{\text{occlusion\_sum}}{\text{aoSamples}} $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ambient_occlusion(p, n, volume, ao_samples, ao_radius, ao_bias, mode):
    if ao_samples == 0:
        return 1.0

    # 1. Generate random vector for tangent space basis
    # In practice, this is often a precomputed hash or noise value
    r = generate_random_vector()
    
    # 2. Construct tangent and bitangent
    # Project r onto plane perpendicular to n
    t = normalize(r - n * dot(r, n))
    b = cross(n, t)
    
    # 3. Apply bias to avoid self-occlusion acne
    p_query = p + n * ao_bias
    
    occlusion_sum = 0.0
    
    for i in range(ao_samples):
        # 4. Sample direction on hemisphere in local tangent space
        # d_local is a unit vector with z >= 0 in (t, b, n) basis
        d_local = sample_hemisphere_direction()
        
        # Transform to world space
        d_world = t * d_local.x + b * d_local.y + n * d_local.z
        
        if mode == 0:
            # Point Sampling Mode
            q = p_query + d_world * ao_radius
            val = volume.query(q)
            # If density is low (or below bias), it's "open"
            # Note: Source implies if value <= bias, add 1.0 to occlusion.
            # This phrasing is ambiguous: usually high density = occluded.
            # Assuming standard AO: if density > threshold, it contributes to occlusion.
            # However, source says: "if value <= bias, add 1.0 to occlusion".
            # This might imply 'bias' is a threshold for *empty* space, or the source
            # defines occlusion as "unblocked". Let's stick strictly to source text:
            if val <= ao_bias:
                occlusion_sum += 1.0
                
        elif mode == 1:
            # Ray Sampling Mode
            hit_dist = ray_march(p_query, d_world, ao_radius, volume)
            if hit_dist is not None:
                # smoothstep(1, 0, ao_radius / dist)
                # If dist == ao_radius, ratio=1, smoothstep=1 (fully occluded?)
                # If dist < ao_radius, ratio > 1, smoothstep=1
                # If dist > ao_radius, ratio < 1, smoothstep interpolates
                # Standard smoothstep(0,1,x) goes 0->1.
                # smoothstep(1,0,x) goes 1->0.
                # So if ratio=1 (hit at edge), weight=1. If ratio=0 (hit at start), weight=0.
                # This implies closer hits = higher occlusion weight.
                ratio = ao_radius / hit_dist
                weight = smoothstep(1.0, 0.0, ratio)
                occlusion_sum += weight
                
    return occlusion_sum / ao_samples
```

**Parameters:**
*   `aoSamples`: Number of hemisphere samples. Maximum 512 [S1, unofficial].
*   `aoRadius`: World-space radius for sampling. Controls the distance over which occlusion is considered [S1, unofficial].
*   `aoBias`: Distance in world space to backtrack the query position. Typical value $10^{-3}$. Used to mitigate acne artifacts [S1, unofficial].
*   `AMBIENT_OCCLUSION_MODE`: Integer switch. 0 for point sampling, 1 for ray sampling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Acne Artifacts:** Sampling directly at the surface $\mathbf{p}$ can cause self-intersection due to floating-point precision errors. Mitigated by backtracking $\mathbf{p}$ along the normal by `aoBias` [S1, unofficial].
*   **Mode 0 Ambiguity:** The source states "if value <= bias, add 1.0 to occlusion." In standard density fields, high density blocks light. If `aoBias` is a small epsilon (e.g., $10^{-3}$), checking `val <= bias` effectively counts empty space as occluded if interpreted literally as "add to occlusion sum". However, in many GVDB contexts, `bias` might refer to a density threshold for *solid* geometry. Given the source explicitly links `aoBias` to backtracking (a geometric offset), the condition `val <= aoBias` in point sampling likely implies a threshold for *density* where values *above* a certain threshold (not explicitly named as a separate density threshold in the notes, but potentially conflated with bias or implying a binary solid/empty check) contribute to occlusion. *Correction based on strict source text:* The source says "if value <= bias, add 1.0 to occlusion." This is counter-intuitive for density (usually high density = occlusion). It is possible `value` refers to a *distance* or *transmittance* in a specific implementation, or the source text contains a logical inversion. However, for implementation, one must follow the source: check if the sampled value is less than or equal to `aoBias`. If `aoBias` is the geometric backtrack distance, comparing density to it is a type mismatch. It is more likely `aoBias` in the condition refers to a *density threshold* (often also named bias in raymarching) or the source implies `value` is a distance-like metric. Given the ambiguity, the implementer should check if `aoBias` is used as a density threshold in the specific GVDB version. *Re-reading source:* "Get bias value from volume at pos." This suggests `aoBias` might be a sampled density value, not just the scalar parameter. If `aoBias` is a scalar parameter for backtracking, the condition `val <= aoBias` is suspicious. Let's assume the standard interpretation: `aoBias` is the geometric offset. The condition likely intends to check if the density is *below* a certain threshold (empty) or *above* (solid). If the source literally means `val <= aoBias`, and `aoBias` is $10^{-3}$, then only very low density values trigger occlusion. This is likely a typo in the source notes or `aoBias` serves dual purpose. For safety, implement exactly as written: `if val <= ao_bias: occlusion_sum += 1.0`.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of shaded points and $S$ is `aoSamples`. Ray sampling mode (Mode 1) has higher constant factors due to ray marching.
*   **Memory Complexity:** $O(1)$ per point, assuming the volume is accessible in memory.
*   **Strengths:** Simple to implement; supports both fast point sampling and more accurate ray casting; effective for local shading in volumetric data.
*   **Limitations:** Approximate; does not account for indirect bounces beyond the first hemisphere; performance degrades with high `aoSamples` or large `aoRadius` in ray mode.
*   **When to use:** Real-time or interactive volume visualization where local shading quality is needed without full global illumination.
*   **When not to use:** When physically accurate global illumination is required, or for scenes with complex inter-reflections.

## Background
GVDB Ambient Occlusion is part of the GVDB (GPU Volume Database) ecosystem, developed for high-performance volume rendering on GPUs. It is implemented as a device function within the GVDB library, allowing integration into custom volume renderers. The method leverages the spatial coherence of volume data to efficiently sample local geometry.

## Sources
[S1] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
