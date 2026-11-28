# Gradient-based Diffuse Lighting with Alpha Blending

Also called: Gradient-based Diffuse Lighting

## Overview
This method computes diffuse lighting for a scalar volume by estimating surface normals from voxel gradients and compositing the lit color into a framebuffer using front-to-back alpha blending [S1, unofficial]. It integrates gradient estimation, lighting application, and compositing into a single per-voxel step rather than storing precomputed gradients separately [S1, unofficial].

## How it works
**Inputs:**
- `parent`: Volume data structure containing `width`, `zSize`, `transfer_a`, `transfer_rgb`, and `volume_data` [S1, unofficial].
- `light`: 3D vector `(light[0], light[1], light[2])` representing the light direction [S1, unofficial].
- `out_a`: Accumulated alpha (float) [S1, unofficial].
- `rgb`: Accumulated color (array of 3 floats) [S1, unofficial].
- `id_a`, `id_b`, `id_c`: Current voxel coordinates [S1, unofficial].
- `value`: Current voxel value used for transfer function lookup [S1, unofficial].

**Outputs:**
- Updated `out_a`: Accumulated alpha [S1, unofficial].
- Updated `rgb`: Accumulated color [S1, unofficial].

**Data Structures:**
- Voxel values are accessed via linear indexing using coordinates `id_a`, `id_b`, `id_c` with the layout `v(id_c, id_a + id_b * width)` [S1, unofficial].
- Neighbor indices are clamped to valid ranges: lower bounds to 0, `id_a_2` and `id_b_2` to `parent.width - 1`, and `id_c_2` to `parent.zSize - 1` [S1, unofficial].

**Steps:**

1. **Gradient Estimation:**
   Calculate the 3D gradient vector $\mathbf{g} = (g_x, g_y, g_z)$ using central differences on the least significant byte (0-255) of voxel values, normalized by 255.0 [S1, unofficial].
   $$g_x = \frac{v(id_c, id_{a2} + id_b \cdot width) \ \& \ 0xFF}{255.0} - \frac{v(id_c, id_{a1} + id_b \cdot width) \ \& \ 0xFF}{255.0}$$
   $$g_y = \frac{v(id_c, id_a + id_{b2} \cdot width) \ \& \ 0xFF}{255.0} - \frac{v(id_c, id_a + id_{b1} \cdot width) \ \& \ 0xFF}{255.0}$$
   $$g_z = \frac{v(id_{c2}, id_a + id_b \cdot width) \ \& \ 0xFF}{255.0} - \frac{v(id_{c1}, id_a + id_b \cdot width) \ \& \ 0xFF}{255.0}$$
   where $id_{a1} = id_a - 1$, $id_{a2} = id_a + 1$, $id_{b1} = id_b - 1$, $id_{b2} = id_b + 1$, $id_{c1} = id_c - 1$, and $id_{c2} = id_c + 1$ [S1, unofficial].

2. **Gradient Normalization:**
   Calculate magnitude $|\mathbf{g}| = \sqrt{g_x^2 + g_y^2 + g_z^2}$ [S1, unofficial]. If $|\mathbf{g}| > 0$, divide each component by $|\mathbf{g}|$; otherwise, set all components to 0 [S1, unofficial].

3. **Diffuse Lighting Factor:**
   Calculate $D = \max(g_x l_x + g_y l_y + g_z l_z, 0)$ [S1, unofficial].

4. **Transfer Function Lookup:**
   Extract alpha: $\alpha_{src} = \frac{\text{parent.transfer\_a}[v] \ \& \ 0x000000FF}{255}$ [S1, unofficial].
   Extract RGB from `rgb_val = parent.transfer_rgb[v]`:
   $$R_{src} = \frac{(rgb\_val \ \& \ 0x00FF0000) \gg 16}{255.0}, \quad G_{src} = \frac{(rgb\_val \ \& \ 0x0000FF00) \gg 8}{255.0}, \quad B_{src} = \frac{rgb\_val \ \& \ 0x000000FF}{255.0}$$ [S1, unofficial].

5. **Lighting Application:**
   Apply diffuse lighting with an ambient term of 0.25, clamped to 1.0:
   $$R_{lit} = \min(D \cdot R_{src} + 0.25 \cdot R_{src}, 1.0)$$
   $$G_{lit} = \min(D \cdot G_{src} + 0.25 \cdot G_{src}, 1.0)$$
   $$B_{lit} = \min(D \cdot B_{src} + 0.25 \cdot B_{src}, 1.0)$$ [S1, unofficial].

6. **Alpha Blending (Front-to-Back):**
   Composite the lit color:
   $$R_{out} = (1.0 - \alpha_{out}) \cdot R_{lit} \cdot \alpha_{src} + R_{out}$$
   $$G_{out} = (1.0 - \alpha_{out}) \cdot G_{lit} \cdot \alpha_{src} + G_{out}$$
   $$B_{out} = (1.0 - \alpha_{out}) \cdot B_{lit} \cdot \alpha_{src} + B_{out}$$ [S1, unofficial].
   Update accumulated alpha: $\alpha_{out} = (1.0 - \alpha_{out}) \cdot \alpha_{src} + \alpha_{out}$ [S1, unofficial].

7. **Output Packing:**
   Convert final RGB to 8-bit integers and pack into a 32-bit integer with opaque alpha:
   $$R_{final} = \lfloor R_{out} \cdot 255 \rfloor, \quad G_{final} = \lfloor G_{out} \cdot 255 \rfloor, \quad B_{final} = \lfloor B_{out} \cdot 255 \rfloor$$ [S1, unofficial].
   $$Pixel = (255 \ll 24) \ | \ (R_{final} \ll 16) \ | \ (G_{final} \ll 8) \ | \ B_{final}$$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_diffuse_lighting_with_alpha_blending(parent, light, out_a, rgb, id_a, id_b, id_c, value):
    width = parent.width
    z_size = parent.zSize

    def get_voxel(a, b, c):
        a = max(0, min(a, width - 1))
        b = max(0, min(b, width - 1))
        c = max(0, min(c, z_size - 1))
        idx = c * (width * width) + a + b * width
        return (parent.volume_data[idx] & 0xFF) / 255.0

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
    aa = (parent.transfer_a[value] & 0x000000FF) / 255.0
    rgb_val = parent.transfer_rgb[value]
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

**Edge Cases and Pitfalls:**
- Neighbor indices must be clamped to valid bounds: lower bounds to 0, `id_a_2` and `id_b_2` to `parent.width - 1`, and `id_c_2` to `parent.zSize - 1` [S1, unofficial].
- If the gradient magnitude is zero, the normalized gradient components are explicitly set to 0 to avoid division by zero [S1, unofficial].
- The method assumes a specific memory layout `v(id_c, id_a + id_b * width)` and transfer function format [S1, unofficial].

## Subtopics

### Gradient-Based Shading Interpolation
This variant interpolates between surface-shaded and unshaded results using the gradient magnitude to prevent homogeneous regions from being surface shaded [S2]. The interpolation factor uses the formula $1 - (1 - ||\nabla f||)^2$ to weight the surface shading contribution, allowing midrange gradient magnitudes to be interpreted as surface features [S2]. The final result is modulated by light attenuation (shadows) regardless of the surface shading state [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] browarsoftware/directvolumerenderingcpu, https://github.com/browarsoftware/directvolumerenderingcpu, fetched 2026-10-26 (unofficial)
[S2] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
