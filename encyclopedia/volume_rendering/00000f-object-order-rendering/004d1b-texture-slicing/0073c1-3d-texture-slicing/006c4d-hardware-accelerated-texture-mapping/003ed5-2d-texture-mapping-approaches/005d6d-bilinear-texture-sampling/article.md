# Bilinear Texture Sampling

Also called: Border Clamping Sampling, Unbounded Bilinear Texture Sampling, Unbounded Sampling

## Overview
Bilinear Texture Sampling is a method for retrieving texture values from a 2D texture map at local coordinates $(u, v)$ on a primitive plane using bilinear interpolation combined with border clamping to handle out-of-bounds coordinates [S1, unofficial]. It solves the problem of dark edge artifacts and poor coverage found in bounded sampling approaches that use zero-padding for out-of-bounds areas [S1, unofficial]. It differs from bounded strategies like BBSplat, where color decays from the Gaussian center, by allowing texture information to extend beyond the primitive boundary for smoother transitions [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $T$: The texture map (assumed to be $T \times T$ resolution, where $T$ is an integer representing the dimension size) [S1, unofficial].
    *   $u, v$: Local coordinates on the primitive plane [S1, unofficial].
    *   $\sigma$: A shared hyperparameter controlling the span of the texture region within the primitive [S1, unofficial].
*   **Output:**
    *   $C$: The interpolated color value at $(u, v)$ [S1, unofficial].

**Data Structures**
*   The texture map $T$ is a 2D array of size $T \times T$.
*   Coordinates $u, v$ are floating-point values representing local space.
*   Intermediate variables $i_u, i_v$ are integer indices; $f_u, f_v$ are fractional components.

**Steps**
1.  **Transform to Pixel Coordinates:** Convert local coordinates $(u, v)$ to pixel coordinates in the texture map. The transformation scales the local span defined by $\sigma$ to the texture resolution $T$ and shifts the origin [S1, unofficial]:
    $$ u_{px} = \frac{u + \sigma}{2\sigma} \cdot T - 0.5 $$
    $$ v_{px} = \frac{v + \sigma}{2\sigma} \cdot T - 0.5 $$
2.  **Border Clamping:** Clamp the transformed coordinates to the valid texture range $[0, T-1]$ to handle out-of-bounds access [S1, unofficial]:
    $$ u_{px} = \text{clamp}(u_{px}, 0, T - 1) $$
    $$ v_{px} = \text{clamp}(v_{px}, 0, T - 1) $$
3.  **Decompose Coordinates:** Compute the integer and fractional parts of the clamped coordinates [S1, unofficial]:
    $$ i_u = \lfloor u_{px} \rfloor, \quad f_u = u_{px} - i_u $$
    $$ i_v = \lfloor v_{px} \rfloor, \quad f_v = v_{px} - i_v $$
4.  **Compute Corner Indices:** Determine the indices for the four surrounding texels, ensuring they do not exceed the texture boundary [S1, unofficial]:
    $$ i_{u+1} = \min(i_u + 1, T - 1) $$
    $$ i_{v+1} = \min(i_v + 1, T - 1) $$
5.  **Bilinear Interpolation:** Sample the texture using the four corners weighted by the fractional parts [S1, unofficial]:
    $$ C = (1 - f_u)(1 - f_v)T[i_u, i_v] + f_u(1 - f_v)T[i_{u+1}, i_v] + (1 - f_u)f_vT[i_u, i_{v+1}] + f_u f_v T[i_{u+1}, i_{v+1}] $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilinear_sample(T_map, T_dim, u, v, sigma):
    """
    Perform unbounded bilinear texture sampling with border clamping.
    
    Args:
        T_map: 2D array of shape (T_dim, T_dim) containing texture values.
        T_dim: Integer resolution of the texture map (T x T).
        u: Local coordinate u on the primitive plane.
        v: Local coordinate v on the primitive plane.
        sigma: Hyperparameter controlling the span of the texture region.
        
    Returns:
        C: Interpolated color value.
    """
    # 1. Transform to pixel coordinates
    u_px = ((u + sigma) / (2 * sigma)) * T_dim - 0.5
    v_px = ((v + sigma) / (2 * sigma)) * T_dim - 0.5
    
    # 2. Border clamping to valid texture range [0, T_dim - 1]
    u_px = max(0, min(T_dim - 1, u_px))
    v_px = max(0, min(T_dim - 1, v_px))
    
    # 3. Compute integer and fractional parts
    i_u = int(u_px)
    f_u = u_px - i_u
    i_v = int(v_px)
    f_v = v_px - i_v
    
    # 4. Compute corner indices with boundary check
    i_u_next = min(i_u + 1, T_dim - 1)
    i_v_next = min(i_v + 1, T_dim - 1)
    
    # 5. Bilinear interpolation
    # T_map[i, j] accesses the texel at row i, column j
    c00 = T_map[i_u, i_v]
    c10 = T_map[i_u_next, i_v]
    c01 = T_map[i_u, i_v_next]
    c11 = T_map[i_u_next, i_v_next]
    
    C = ((1 - f_u) * (1 - f_v) * c00 +
         f_u * (1 - f_v) * c10 +
         (1 - f_u) * f_v * c01 +
         f_u * f_v * c11)
         
    return C
```

**Parameters**
*   $\sigma$: A shared hyperparameter controlling the span of the texture region within the primitive [S1, unofficial]. Similar to Gaussian Billboards, it controls the relative span of the textured region [S1, unofficial].
*   $T$: The resolution of the texture map ($T \times T$) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Out-of-bounds coordinates:** The method explicitly handles coordinates outside the $[0, 1]$ local range by clamping to the texture boundary, preventing crashes or undefined behavior [S1, unofficial].
*   **Dark Edge Artifacts:** Using bounded sampling (like BBSplat) with standard 2DGS opacity can cause dark edge artifacts because color decays from the center. This method prevents this by ensuring equal weights across the four bilinear corners, relying solely on 2DGS opacity for alpha fall-off [S1, unofficial].
*   **Precision:** The transformation formula involves division by $2\sigma$. If $\sigma$ is very small, numerical instability may occur. Not given in the sources.

## Subtopics

### 2D Texture-Based Volume Rendering
*   **What it is:** A broader category of techniques that map 2D textures onto volume primitives.
*   **Difference:** Bilinear Texture Sampling is a specific sampling strategy within this category, distinguished by its use of border clamping and unbounded coordinate handling.
*   **When to choose:** When smooth texture transitions across primitive boundaries are required without dark edges.
*   **Link:** [2D Texture-Based Volume Rendering](916-2d-texture-based-volume-rendering.md)

### Palette Texture Volume Rendering
*   **What it is:** A method using palette-based texture mapping.
*   **Difference:** Uses a palette index lookup rather than direct bilinear interpolation of color values.
*   **When to choose:** When memory bandwidth is limited or color quantization is desired.
*   **Link:** [Palette Texture Volume Rendering](6502-palette-texture-volume-rendering.md)

### Cabral Texture Mapping
*   **What it is:** A texture mapping approach based on Cabral's method.
*   **Difference:** Uses a different coordinate transformation and sampling logic compared to the bilinear border-clamping approach.
*   **When to choose:** When specific geometric mapping properties of Cabral's method are required.
*   **Link:** [Cabral Texture Mapping](7257-cabral-texture-mapping.md)

### Wall Surface Projection & Texture Mapping
*   **What it is:** Projects textures onto wall-like surfaces.
*   **Difference:** Focuses on planar surface projection rather than primitive-local bilinear sampling.
*   **When to choose:** For architectural or planar scene reconstruction.
*   **Link:** [Wall Surface Projection & Texture Mapping](9883-wall-surface-projection-texture-mapping.md)

### Multi-constraints-based Projective Texture Mapping
*   **What it is:** Uses multiple constraints for projective texture mapping.
*   **Difference:** More complex constraint solving compared to the direct bilinear interpolation of this method.
*   **When to choose:** When high-fidelity projective consistency is needed across multiple views.
*   **Link:** [Multi-constraints-based projective texture mapping](12075-multi-constraints-based-projective-texture-mapping.md)

### Projective Space Texture Mapping with Multi-Resolution Templates
*   **What it is:** Uses multi-resolution templates in projective space.
*   **Difference:** Involves template matching and multi-resolution handling, whereas this method is a direct per-pixel sampling operation.
*   **When to choose:** When handling varying levels of detail in projective space is critical.
*   **Link:** [Projective Space Texture Mapping with Multi-Resolution Templates](13236-projective-space-texture-mapping-with-multi-resolution-templ.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, involving a constant number of arithmetic operations and texture lookups.
*   **Memory Complexity:** $O(T^2)$ for the texture map storage, where $T$ is the texture resolution.
*   **Strengths:** Prevents dark edge artifacts common in bounded sampling; provides smoother transitions by extending texture information beyond primitive boundaries; simple to implement.
*   **Limitations:** Relies on a shared hyperparameter $\sigma$ which may require tuning; texture complexity is limited by the resolution $T$ [S1, unofficial].
*   **When to use:** When rendering textured primitives (e.g., 2DGS) and avoiding dark edges at boundaries is important.
*   **When not to use:** When bounded color decay from the center is explicitly desired (e.g., BBSplat style).

## Background
Bilinear Texture Sampling was introduced in the context of "Less Gaussians, Texture More" (LGTM), a method for 4K feed-forward textured splatting [S1, unofficial]. It was developed to address limitations in vanilla 3DGS texture complexity and bounded sampling artifacts [S1, unofficial]. The method is similar to Gaussian Billboards (Weiss & Bradley 2024), where $\sigma$ controls the relative span of the textured region [S1, unofficial]. It differs from BBSplat (Svitov et al. 2025) by using unbounded sampling with border clamping instead of bounded sampling with color decay [S1, unofficial].

## Sources
[S1] Less Gaussians, Texture More: 4K Feed-Forward Textured Splatting, https://doi.org/10.48550/arXiv.2603.25745, fetched 2026-10-30 (unofficial)
