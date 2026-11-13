# PixelFlow Volume Rendering

Also called: PixelFlow Polygon Texturing, PixelFlow Volume Rendering Variant

## Overview
PixelFlow Volume Rendering is a polygon resampling method designed for the PixelFlow graphics system that overcomes the lighting and data size limitations of the RealityEngine by allowing pixel processors to manipulate 3D texture values directly [S1, unofficial]. It differs from its siblings by replacing precomputed shading coefficients with on-the-fly Phong lighting calculations derived from interpolated volume gradients, enabling general image-space lighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset stored in texture memory; a set of object-space resampling polygons covering the volume; a piecewise-linear transfer function mapping data values to color and opacity [S1, unofficial].
*   **Output:** A composited 2D image in the frame buffer [S1, unofficial].

**Data Structures:**
*   **Texture Memory:** Stores 32-bit values [S1, unofficial]. Each 32-bit value is treated as four 8-bit components: three for the normalized data gradient $(g_x, g_y, g_z)$ and one for the raw data value $(d)$ [S1, unofficial].
*   **Pixel Processor State:** Each of the $128^2$ pixel processors holds an 8-bit ALU, hardware multiply, and hardware-assisted square root/divide instructions [S1, unofficial].

**Parameters:**
*   **Resolution:** Performance scales with screen resolution; typical demonstration uses a $640 \times 512$ window [S1, unofficial].
*   **Volume Size:** Demonstrated with a $256^3$ volume [S1, unofficial].
*   **Polygon Count:** Demonstrated with 400 object-space resampling polygons [S1, unofficial].

**Steps:**
1.  **Texture Fetch:** For each pixel covered by a resampling polygon, the hardware fetches eight arbitrarily-addressed 32-bit values from texture memory in under 500 µs [S1, unofficial].
2.  **Interpolation:** The pixel processor software performs trilinear interpolation on the four 8-bit components (gradient $x, y, z$ and raw data) for the fetched values [S1, unofficial].
3.  **Gradient Normalization:** The interpolated gradient vector $\mathbf{g} = (g_x, g_y, g_z)$ is normalized to unit length:
    $$ \hat{\mathbf{g}} = \frac{\mathbf{g}}{\|\mathbf{g}\|} $$
    [S1, unofficial].
4.  **Lighting Calculation:** Phong shaded lighting coefficients are computed using the normalized gradient $\hat{\mathbf{g}}$ and fixed image-space light sources [S1, unofficial].
5.  **Classification:** Piecewise-linear transformations map the interpolated raw data value $d$ to color $C$ and opacity $\alpha$ [S1, unofficial].
6.  **Compositing:** The color $C$ is multiplied by the lighting coefficient and composited into the frame buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Executed concurrently by each of the 128^2 pixel processors
def pixel_flow_render(polygon, u, v, w):
    # 1. Fetch 8 arbitrarily-addressed 32-bit values from texture memory
    # Hardware primitive: returns 8 32-bit values in <500 µs [S1, unofficial]
    fetched_32bit = [polygon.sample_32bit(i) for i in range(8)]
    
    # 2. Unpack to 4x 8-bit components per value: [gx, gy, gz, raw]
    samples = []
    for v_val in fetched_32bit:
        samples.append([
            (v_val >> 24) & 0xFF, (v_val >> 16) & 0xFF,
            (v_val >> 8) & 0xFF, v_val & 0xFF
        ])
        
    # 3. Trilinear interpolation of the 4 components
    # Corner ordering matches hardware fetch layout [G]
    interp = trilinear_interpolate_4d(samples, u, v, w)
    gx, gy, gz, raw = interp
    
    # 4. Normalize gradient
    mag = sqrt(gx**2 + gy**2 + gz**2)
    if mag < 1e-6:
        # Guard against division by zero; default to z-axis normal [G]
        nx, ny, nz = 0.0, 0.0, 1.0
    else:
        nx, ny, nz = gx / mag, gy / mag, gz / mag
        
    # 5. Phong lighting coefficients
    # Light direction fixed in image space [S1, unofficial]
    light_dir = (0.0, 0.0, 1.0)
    diff = max(0.0, nx * light_dir[0] + ny * light_dir[1] + nz * light_dir[2])
    lighting = 0.2 + 0.8 * diff
    
    # 6. Piecewise-linear transfer function
    # Maps 8-bit raw value to color and opacity via piecewise-linear function [G]
    t = raw / 255.0
    color = t
    opacity = t
    
    # 7. Composite into frame buffer
    final_color = color * lighting
    alpha_blend(final_color, opacity)

def trilinear_interpolate_4d(samples, u, v, w):
    # samples: list of 8 tuples of 4 floats
    # Standard trilinear interpolation across 8 corner samples [G]
    def lerp(a, b, t): return a + t * (b - a)
    c00 = [lerp(samples[0][i], samples[1][i], u) for i in range(4)]
    c01 = [lerp(samples[2][i], samples[3][i], u) for i in range(4)]
    c10 = [lerp(samples[4][i], samples[5][i], u) for i in range(4)]
    c11 = [lerp(samples[6][i], samples[7][i], u) for i in range(4)]
    r0 = [lerp(c00[i], c01[i], v) for i in range(4)]
    r1 = [lerp(c10[i], c11[i], v) for i in range(4)]
    return [lerp(r0[i], r1[i], w) for i in range(4)]

def alpha_blend(color, opacity):
    # Standard over operator compositing into frame buffer [G]
    pass
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If the interpolated gradient magnitude is zero, normalization fails [G].
*   **Precision:** The use of 8-bit components for gradient and raw data limits precision compared to 32-bit floating-point representations, potentially affecting shading accuracy [S1, unofficial].
*   **Memory Limits:** Each processor card holds 0.5M texture values [S1, unofficial].

## Subtopics

### 3D Texture Proxy Geometry
[870-3d-texture-proxy-geometry.md]

### AR mapping to 3D texture
[884-ar-mapping-to-3d-texture.md]

### Volume Texture Creation
[1857-volume-texture-creation.md]

### Texture Uniting (Multi-Channel Packing)
[2470-texture-uniting-multi-channel-packing.md]

### Hardware Volume Rendering Pipeline with Pixel Textures
[5641-hardware-volume-rendering-pipeline-with-pixel-textures.md]

### 3D Transfer Function Texture Layout
[5646-3d-transfer-function-texture-layout.md]

### Memory Organization and Layout
[7154-memory-organization-and-layout.md]

### 3D Volume Atlas Packing
[7158-3d-volume-atlas-packing.md]

## Operating characteristics
*   **Time Complexity:** Performance is inversely proportional to the number of screen pixels and resampling polygons used [S1, unofficial]. Doubling the number of polygons halves the frame rate [S1, unofficial].
*   **Memory Complexity:** Each processor card holds 0.5M texture values [S1, unofficial].
*   **Measured Performance:** A system with 43 processor cards rendered a $256^3$ volume into a $640 \times 512$ window using 400 resampling polygons at over 10 frames/second [S1, unofficial].
*   **Strengths:** Supports general lighting by sources fixed in image-space; scalable memory via multiple cards; high performance for large volumes [S1, unofficial].
*   **Limitations:** Requires specialized hardware (PixelFlow); limited by 8-bit component precision; performance degrades with increased polygon count and screen resolution [S1, unofficial].
*   **When to Use:** When using the PixelFlow graphics system and requiring general image-space lighting for volume rendering [S1, unofficial].
*   **When Not to Use:** On standard hardware without PixelFlow texturing capabilities; when higher precision than 8-bit components is required [S1, unofficial].

## Background
PixelFlow Volume Rendering was proposed for the PixelFlow graphics system under development at the University of North Carolina at Chapel Hill [S1, unofficial]. It was designed to overcome the limitations of the RealityEngine, specifically regarding realistic lighting and data size [S1, unofficial]. The method leverages the PixelFlow texturing hardware, which allows $128^2$ pixel processors to access texture memory and perform user-defined manipulations on the data [S1, unofficial]. The approach was described in a technical report from UNC Chapel Hill in 1991 [S1, unofficial].

## Sources
[S1] Accelerating Volume Reconstruction with 3D Texture Hardware, http://www.cs.unc.edu/techreports/93-027.pdf, fetched 2026-10-27 (unofficial)
