# 3D Lookup Table Illumination Model

Also called: Henyey-Greenstein LUT

## Overview
The 3D Lookup Table (LUT) Illumination Model approximates light transport in participating media by indexing a precomputed table with three angular parameters ($\alpha, \beta, \gamma$) derived from the surface normal, light direction, and view direction [S1]. It differs from sibling methods like [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md) by replacing real-time integration of the scattering phase function with a fast trilinear texture lookup, allowing for complex scattering models (such as Henyey-Greenstein) to be evaluated efficiently on the GPU [S1].

## How it works
**Inputs:**
- Voxel position $\mathbf{p}$, view position $\mathbf{v}$, and surface normal $\mathbf{n}$ (approximated from volume gradient).
- List of light sources, each with position $\mathbf{p}_L$ and color $\mathbf{c}_L$.
- LUT resolution $n$ and scattering parameter $g$ (for Henyey-Greenstein).
- Reflection factor $a_{FactorReflection}$ and voxel albedo $a_{Color}$ [S1].

**Outputs:**
- Radiance contribution $L$ at $\mathbf{p}$ [S1].

**Data Structures:**
- **3D LUT:** A 3D texture array of size $n \times n \times n$. Values are normalized to $[0, 1]$ for CUDA compatibility [S1].
- **Reflection Texture:** A 2D or 3D texture indexed by sample position for reflection properties [S1].

**Steps:**
1.  **Compute Normal:** Approximate the normal $\mathbf{n}$ as the negative gradient of the volume density, normalized to unit length:
    $$ \mathbf{n} = -\frac{\nabla V}{\|\nabla V\|} $$
    The gradient can be computed via finite differences or a library like `vol∇` [S1].

2.  **Compute Direction Vectors:**
    - Incoming light direction: $\mathbf{L}_i = \text{normalize}(\mathbf{p}_L - \mathbf{p})$ [S1].
    - Outgoing (view) direction: $\mathbf{L}_o = \text{normalize}(\mathbf{v} - \mathbf{p})$ [S1].

3.  **Compute Angles:**
    - $\alpha$: Angle between $\mathbf{n}$ and $\mathbf{L}_i$:
      $$ \alpha = \arccos(\langle \mathbf{n}, \mathbf{L}_i \rangle) $$
    - $\beta$: Angle between $\mathbf{n}$ and $\mathbf{L}_o$:
      $$ \beta = \arccos(\langle \mathbf{n}, \mathbf{L}_o \rangle) $$
    - Project $\mathbf{L}_i$ and $\mathbf{L}_o$ onto the plane orthogonal to $\mathbf{n}$:
      $$ \mathbf{L}_i' = \mathbf{L}_i - \langle \mathbf{L}_i, \mathbf{n} \rangle \mathbf{n} $$
      $$ \mathbf{L}_o' = \mathbf{L}_o - \langle \mathbf{L}_o, \mathbf{n} \rangle \mathbf{n} $$
    - $\gamma$: Angle between the projected vectors $\mathbf{L}_i'$ and $\mathbf{L}_o'$:
      $$ \gamma = \arccos\left( \frac{\langle \mathbf{L}_i', \mathbf{L}_o' \rangle}{\|\mathbf{L}_i'\| \|\mathbf{L}_o'\|} \right) $$
      Note: If $\|\mathbf{L}_i'\|$ or $\|\mathbf{L}_o'\|$ is zero (vectors parallel to normal), $\gamma$ is undefined; handle as a degenerate case [G].

4.  **LUT Indexing:**
    The angles are constrained to $[0, \pi]$ due to symmetry [S1]. Map them to LUT coordinates $a, b, c \in [0, n-1]$:
    $$ a = \text{clamp}\left( \frac{\alpha}{\pi} (n-1), 0, n-1 \right) $$
    $$ b = \text{clamp}\left( \frac{\beta}{\pi} (n-1), 0, n-1 \right) $$
    $$ c = \text{clamp}\left( \frac{\gamma}{\pi} (n-1), 0, n-1 \right) $$
    The LUT value is retrieved via trilinear interpolation:
    $$ \text{light} = \text{TrilinearInterpolate}(\text{LUT}, a, b, c) $$
    The LUT is precomputed such that $\text{LUT}(a, b, c) = HG(\theta, g)$, where $\theta$ is the scattering angle derived from rotating $\mathbf{L}_o$ around $\mathbf{n}$ by $\gamma$ and computing the dot product with fixed $\mathbf{L}_i$ [S1].

5.  **Final Contribution:**
    Combine the LUT lookup with reflection and albedo:
    $$ \text{result} \leftarrow \text{result} + (a_{FactorReflection} \cdot \text{ReflectionTexture}(\mathbf{p}) \cdot \text{light} \cdot \mathbf{c}_L \cdot a_{Color}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shade_3d_lut(a_position, a_view_position, a_surface_normal, a_color, a_light_sources, a_factor_reflection, lut_texture, reflection_texture, n):
    """
    Calculate illumination at a voxel using 3D LUT.
    a_position: vec3, voxel position
    a_view_position: vec3, camera/view position
    a_surface_normal: vec3, normalized gradient normal
    a_color: vec3, voxel albedo
    a_light_sources: list of dicts with 'position' and 'color'
    a_factor_reflection: float, reflection scaling factor
    lut_texture: 3D texture object, size n x n x n
    reflection_texture: 2D/3D texture object
    n: int, LUT resolution
    """
    result = vec3(0.0, 0.0, 0.0)
    
    for light_source in a_light_sources:
        # 1. Compute directions
        light_out = normalize(light_source['position'] - a_position)
        light_in = normalize(a_view_position - a_position)
        n = normalize(a_surface_normal)
        
        # 2. Compute angles alpha, beta, gamma
        # Alpha: angle between normal and light direction
        alpha = acos(dot(n, light_out))
        
        # Beta: angle between normal and view direction
        beta = acos(dot(n, light_in))
        
        # Project vectors onto plane orthogonal to normal
        light_out_proj = light_out - dot(light_out, n) * n
        light_in_proj = light_in - dot(light_in, n) * n
        
        # Normalize projections to handle edge cases where projection is zero
        norm_out = length(light_out_proj)
        norm_in = length(light_in_proj)
        
        if norm_out > 1e-6 and norm_in > 1e-6:
            light_out_proj = light_out_proj / norm_out
            light_in_proj = light_in_proj / norm_in
            # Gamma: angle between projected vectors
            gamma = acos(dot(light_out_proj, light_in_proj))
        else:
            # Degenerate case: light or view is parallel to normal
            gamma = 0.0
            
        # 3. Map angles to LUT indices [0, n-1]
        # Angles are in [0, pi]
        idx_a = clamp(alpha / pi * (n - 1), 0, n - 1)
        idx_b = clamp(beta / pi * (n - 1), 0, n - 1)
        idx_c = clamp(gamma / pi * (n - 1), 0, n - 1)
        
        # 4. Lookup illumination
        # Trilinear interpolation is assumed by the texture lookup function
        light_intensity = lut_texture.sample_trilinear(idx_a, idx_b, idx_c)
        
        # 5. Lookup reflection
        reflection = a_factor_reflection * reflection_texture.sample(a_position)
        
        # 6. Accumulate result
        contribution = reflection * light_intensity * light_source['color'] * a_color
        result += contribution
        
    return result
```

**Parameters:**
- $n$: LUT resolution. Higher $n$ increases accuracy but memory usage scales as $O(n^3)$ [S1].
- $g$: Henyey-Greenstein asymmetry parameter. Used during LUT precomputation [S1].
- Angle range: $\alpha, \beta, \gamma \in [0, \pi]$ [S1].

**Edge Cases and Pitfalls:**
- **Parallel Vectors:** If $\mathbf{L}_i$ or $\mathbf{L}_o$ is parallel to $\mathbf{n}$, the projection is zero, making $\gamma$ undefined. The pseudocode handles this by setting $\gamma=0$ [G].
- **Precision:** LUT values must be normalized to $[0, 1]$ for CUDA texture sampling [S1].
- **Symmetry:** The model relies on symmetry to restrict angles to $[0, \pi]$, reducing LUT size [S1].

## Operating characteristics
- **Time Complexity:** $O(1)$ per light source per voxel, dominated by texture lookups and vector math. Independent of scene depth or sample count [S1].
- **Memory Complexity:** $O(n^3)$ for the LUT. For typical $n=64$, this is $262,144$ floats, which is small enough for GPU memory [S1].
- **Performance:** Enables real-time rendering on GPUs by avoiding expensive integration of the scattering equation during ray marching [S1].
- **Strengths:** Fast evaluation of complex phase functions (e.g., Henyey-Greenstein); decouples scattering physics from rendering loop [S1].
- **Limitations:** Approximation error depends on LUT resolution $n$; cannot handle anisotropic scattering beyond the parameterized model; requires precomputation [S1].

## Background
The method was introduced in a 2024 paper titled "GPU-Enabled Volume Renderer for Use with MATLAB" [S1]. It builds on the principle that light-particle interaction can be fully described by three angles relative to the surface normal. The LUT is typically precomputed using the Henyey-Greenstein phase function, allowing the renderer to query pre-integrated scattering results [S1].

## Sources
[S1] GPU-Enabled Volume Renderer for Use with MATLAB, https://doi.org/10.3390/digital4040049, fetched 2026-10-29
