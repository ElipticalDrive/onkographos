# Gradient-based Diffuse Lighting with Alpha Blending

Also called: Gradient-based Diffuse Lighting

## Overview
This method computes diffuse lighting for a scalar volume by estimating surface normals from voxel gradients and compositing the lit color into a framebuffer using front-to-back alpha blending [S1]. It integrates gradient estimation, lighting application, and compositing into a single per-voxel step rather than storing precomputed gradients separately [S1].

## How it works
**Inputs:**
- `parent`: Volume data structure containing:
  - `width`: Integer width of the volume grid (bounds for x and y dimensions) [S1].
  - `zSize`: Integer depth (z-dimension) of the volume grid [S1].
  - `v2`: 1D array of voxel values (integers, typically 0-255) [S1].
  - `transfer_a`: 1D array of 32-bit integers storing alpha values indexed by voxel value [S1].
  - `transfer_rgb`: 1D array of 32-bit integers storing RGB values indexed by voxel value [S1].
- `light`: 3D vector `(light[0], light[1], light[2])` representing the light direction [S1].
- `out_a`: Accumulated alpha (float, initialized to 0.0) [S1].
- `rgb`: Accumulated color (array of 3 floats, initialized to 0.0) [S1].

**Outputs:**
- Updated `out_a`: Accumulated alpha [S1].
- Updated `rgb`: Accumulated color [S1].

**Data Structures:**
- Voxel values are accessed via linear indexing using coordinates `id_a`, `id_b`, `id_c` [S1].
- The memory layout follows `v(id_c, id_a + id_b * width)`, implying a linear index of `id_c * (width * width) + id_a + id_b * width` [S1].
- Neighbor indices are clamped to valid ranges: lower bounds to 0, `id_a_2` and `id_b_2` to `width - 1`, and `id_c_2` to `zSize - 1` [S1].

**Steps:**

1. **Gradient Estimation:**
   Calculate the 3D gradient vector $\mathbf{g} = (g_x, g_y, g_z)$ using central differences on the least significant byte (0-255) of voxel values, normalized by 255.0 [S1].
   $$g_x = \frac{V(id_c, id_{a2}, id_b) \ \& \ 0xFF}{255.0} - \frac{V(id_c, id_{a1}, id_b) \ \& \ 0xFF}{255.0}$$
   $$g_y = \frac{V(id_c, id_a, id_{b2}) \ \& \ 0xFF}{255.0} - \frac{V(id_c, id_a, id_{b1}) \ \& \ 0xFF}{255.0}$$
   $$g_z = \frac{V(id_{c2}, id_a, id_b) \ \& \ 0xFF}{255.0} - \frac{V(id_{c1}, id_a, id_b) \ \& \ 0xFF}{255.0}$$
   where $V(id_c, id_a, id_b)$ returns the clamped voxel value [S1].

2. **Gradient Normalization:**
   Calculate magnitude $|\mathbf{g}| = \sqrt{g_x^2 + g_y^2 + g_z^2}$ [S1]. If $|\mathbf{g}| > 0$, divide each component by $|\mathbf{g}|$; otherwise, set all components to 0 [S1].

3. **Diffuse Lighting Factor:**
   Calculate $D = \max(g_x l_x + g_y l_y + g_z l_z, 0)$ [S1].

4. **Transfer Function Lookup:**
   Extract alpha: $\alpha_{src} = \frac{\text{parent.transfer\_a}[v] \ \& \ 0x000000FF}{255}$ [S1].
   Extract RGB from `rgb_val = parent.transfer_rgb[v]`:
   $$R_{src} = \frac{(rgb\_val \ \& \ 0x00FF0000) \gg 16}{255.0}, \quad G_{src} = \frac{(rgb\_val \ \& \ 0x0000FF00) \gg 8}{255.0}, \quad B_{src} = \frac{rgb\_val \ \& \ 0x000000FF}{255.0}$$ [S1].

5. **Lighting Application:**
   Apply diffuse lighting with an ambient term of 0.25, clamped to 1.0:
   $$R_{lit} = \min(D \cdot R_{src} + 0.25 \cdot R_{src}, 1.0)$$
   $$G_{lit} = \min(D \cdot G_{src} + 0.25 \cdot G_{src}, 1.0)$$
   $$B_{lit} = \min(D \cdot B_{src} + 0.25 \cdot B_{src}, 1.0)$$ [S1].

6. **Alpha Blending (Front-to-Back):**
   Composite the lit color:
   $$R_{out} = (1.0 - \alpha_{out}) \cdot R_{lit} \cdot \alpha_{src} + R_{out}$$
   $$G_{out} = (1.0 - \alpha_{out}) \cdot G_{lit} \cdot \alpha_{src} + G_{out}$$
   $$B_{out} = (1.0 - \alpha_{out}) \cdot B_{lit} \cdot \alpha_{src} + B_{out}$$ [S1].
   Update accumulated alpha: $\alpha_{out} = (1.0 - \alpha_{out}) \cdot \alpha_{src} + \alpha_{out}$ [S1].

7. **Output Packing:**
   Convert final RGB to 8-bit integers and pack into a 32-bit integer with opaque alpha:
   $$R_{final} = \lfloor R_{out} \cdot 255 \rfloor, \quad G_{final} = \lfloor G_{out} \cdot 255 \rfloor, \quad B_{final} = \lfloor B_{out} \cdot 255 \rfloor$$ [S1].
   $$Pixel = (255 \ll 24) \ | \ (R_{final} \ll 16) \ | \ (G_{final} \ll 8) \ | \ B_{final}$$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_diffuse_lighting_with_alpha_blending(
    parent, light, out_a, rgb, id_a, id_b, id_c
):
    width = parent.width
    z_size = parent.zSize
    height = width  # Derived from clamping rule: id_b_2 clamped to width - 1 [S1]

    def get_voxel(a, b, c):
        a = max(0, min(a, width - 1))
        b = max(0, min(b, height - 1))
        c = max(0, min(c, z_size - 1))
        idx = c * (width * height) + a + b * width
        return (parent.v2[idx] & 0xFF) / 255.0

    # 1. Gradient Estimation
    gx = get_voxel(id_a + 1, id_b, id_c) - get_voxel(id_a - 1, id_b, id_c)
    gy = get_voxel(id_a, id_b + 1, id_c) - get_voxel(id_a, id_b - 1, id_c)
    gz = get_voxel(id_a, id_b, id_c + 1) - get_voxel(id_a, id_b, id_c - 1)

    # 2. Gradient Normalization
    grad_l = (gx**2 + gy**2 + gz**2)**0.5
    if grad_l > 0:
        nx, ny, nz = gx / grad_l, gy / grad_l, gz / grad_l
    else:
        nx, ny, nz = 0.0, 0.0, 0.0

    # 3. Diffuse Lighting Factor
    diffuse = max(nx * light[0] + ny * light[1] + nz * light[2], 0)

    # 4. Transfer Function Lookup
    current_idx = id_c * (width * height) + id_a + id_b * width
    current_val = parent.v2[current_idx]
    
    aa = (parent.transfer_a[current_val] & 0x000000FF) / 255
    rgb_val = parent.transfer_rgb[current_val]
    
    vv_red = ((rgb_val & 0x00FF0000) >> 16) / 255.0
    vv_green = ((rgb_val & 0x0000FF00) >> 8) / 255.0
    vv_blue = (rgb_val & 0x000000FF) / 255.0

    # 5. Lighting Application
    vv_red = min(diffuse * vv_red + 0.25 * vv_red, 1.0)
    vv_green = min(diffuse * vv_green + 0.25 * vv_green, 1.0)
    vv_blue = min(diffuse * vv_blue + 0.25 * vv_blue, 1.0)

    # 6. Alpha Blending
    rgb[0] = (1.0 - out_a) * vv_red * aa + rgb[0]
    rgb[1] = (1.0 - out_a) * vv_green * aa + rgb[1]
    rgb[2] = (1.0 - out_a) * vv_blue * aa + rgb[2]
    out_a = (1.0 - out_a) * aa + out_a

    return out_a, rgb
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel for the lighting and blending step, requiring 6 neighbor lookups for gradient estimation and constant-time transfer function access [S1].
- **Memory Complexity:** $O(1)$ additional memory per voxel for the current step, plus storage for the transfer functions and volume data [S1].
- **Performance:** The method is computationally lightweight per voxel, suitable for CPU-based direct volume rendering as indicated by the source repository [S1]. The 6 neighbor lookups and transfer function accesses add overhead compared to simple alpha blending [S1].
- **Strengths:** Provides surface-like shading effects without explicit surface extraction and is straightforward to implement [S1].
- **Limitations:** Gradient estimation via central differences can be noisy in low-resolution volumes, and boundary clamping may distort normals at edges [S1]. The method assumes a specific memory layout and transfer function format [S1].

## Background
The method originates from a direct volume rendering implementation on the CPU, specifically documented in the `browarsoftware/directvolumerenderingcpu` repository [S1]. It combines gradient-based normal estimation with standard front-to-back alpha compositing to render scalar volumes [S1].

## Sources
[S1] browarsoftware/directvolumerenderingcpu, https://github.com/browarsoftware/directvolumerenderingcpu, fetched 2026-10-26 [S1, unofficial]
