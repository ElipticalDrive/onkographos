# 3D Gaussian Splatting via Instancing

Also called: Hardware-Accelerated Gaussian Rasterization, Instanced 3D Gaussian Splatting

## Overview
3D Gaussian Splatting via Instancing renders volumetric data or 2D Gaussian projections by submitting a single instanced draw call per Gaussian primitive, utilizing the standard hardware vertex/fragment pipeline to avoid the memory traffic and sorting overhead of tile-based rasterizers [S1, unofficial]. It solves the problem of efficiently rendering millions of semi-transparent primitives by dynamically sizing quads based on opacity and computing the Gaussian decay in the fragment shader, setting itself apart from its parent [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md) by explicitly handling the spatial extent and opacity cutoff of Gaussian kernels [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of $N$ Gaussians, each defined by a center position, peak opacity $\sigma$, color, and variance parameters [S1, S2, unofficial]. For 3D volume rendering, a 3D texture volume is also required [S2, unofficial].
*   **Output:** A 2D image (for 2D splatting) or a 3D volume texture (for 3D splatting) [S1, S2, unofficial].

**Data Structures:**
*   **Gaussian Attributes:** Per-instance data passed to the vertex shader [S1, S2, unofficial]. For 2D splatting, this includes the peak opacity $\sigma$ and local vertex coordinates [S1, unofficial]. For 3D splatting, this includes the instance ID (representing the Z-slice) and 2D position within the slice [S2, unofficial].
*   **Volume (3D only):** A struct containing `FboHandle` and `TextureHandle` [S2, unofficial]. The texture is allocated with `GL_RGB16F` format, `GL_RGB` type, and `GL_HALF_FLOAT` data type, with wrap modes set to `GL_CLAMP_TO_EDGE` and filters set to `GL_LINEAR` [S2, unofficial].

**Numbered Steps:**

1.  **Pre-processing (Culling):**
    *   Iterate through all Gaussians.
    *   If peak opacity $\sigma < 1/255$, cull the Gaussian [S1, unofficial]. This threshold corresponds to the minimum quantization level for 8-bit output [S1, unofficial].

2.  **Setup Render Target (3D Volume Only):**
    *   Generate a Framebuffer Object (FBO) and a 3D Texture [S2, unofficial].
    *   Bind the 3D texture to the FBO's `GL_COLOR_ATTACHMENT0` using `glFramebufferTexture` [S2, unofficial].
    *   Set texture parameters: `GL_TEXTURE_WRAP_S/T/R` to `GL_CLAMP_TO_EDGE`, `GL_TEXTURE_MIN/MAG_FILTER` to `GL_LINEAR` [S2, unofficial].

3.  **Vertex Shader Execution (Per Instance):**
    *   **For 2D Splatting (WebSplatter approach):**
        *   Receive instance data: peak opacity $\sigma$ and local vertex coordinates [S1, unofficial].
        *   Compute the cutoff radius $r$ using the formula:
            $$r = \sqrt{\ln(255 \cdot \sigma)}$$
        *   Scale the quad vertices by $r$ to tightly bound the significant opacity area [S1, unofficial].
        *   Pass a local coordinate $v_i$ for each of the four quad vertices to the fragment shader [S1, unofficial].
    *   **For 3D Splatting (Prideout approach):**
        *   Pass `gl_InstanceID` as `vInstance` to the geometry shader [S2, unofficial].
        *   Pass 2D position `vPosition` to the geometry shader [S2, unofficial].

4.  **Geometry Shader Execution (3D Volume Only):**
    *   Receive `vInstance` and `vPosition` from the vertex shader [S2, unofficial].
    *   Set `gl_Layer = vInstance[0]` to render to the specific slice of the 3D texture [S2, unofficial].
    *   Compute `gPosition.z` using `InverseSize` (where `InverseSize = 1.0 / depth`):
        $$gPosition.z = 1.0 - 2.0 \cdot vInstance[0] \cdot InverseSize$$
    *   Pass `gPosition.xy = vPosition` to the fragment shader [S2, unofficial].

5.  **Rasterization and Fragment Shader Execution:**
    *   **For 2D Splatting:**
        *   The hardware interpolates the local coordinates $v_i$ passed from the vertex shader [S1, unofficial]. For a fragment with barycentric coordinates $(\alpha, \beta, \gamma)$ within a triangle defined by vertices $(v_0, v_1, v_2)$, the interpolated local coordinate $p$ is:
            $$p = \alpha v_0 + \beta v_1 + \gamma v_2$$
        *   Compute the squared distance from the center: $d^2 = p \cdot p$ [S1, unofficial].
        *   Compute the final color and alpha contribution:
            $$FragColor = Color \cdot e^{-d^2}$$
            (Note: The peak opacity $\sigma$ is implicitly handled by the quad size and alpha blending [S1, unofficial]).
    *   **For 3D Splatting:**
        *   Receive `gPosition` from the geometry shader [S2, unofficial].
        *   Compute squared distance: $r^2 = \text{dot}(gPosition, gPosition)$ [S2, unofficial].
        *   Compute output color:
            $$FragColor = Color \cdot NormalizationConstant \cdot \exp(r^2 \cdot InverseVariance)$$
        *   `InverseVariance` controls the spread of the Gaussian [S2, unofficial].

**Pseudocode (2D Splatting - WebSplatter Style):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def compute_cutoff_radius(sigma):
    # Cull if below threshold
    if sigma < 1.0 / 255.0:
        return None
    return math.sqrt(math.log(255.0 * sigma))

def vertex_shader_2d(instance_sigma, quad_vertex_local_coord):
    r = compute_cutoff_radius(instance_sigma)
    if r is None:
        return None
    scaled_coord = quad_vertex_local_coord * r
    return scaled_coord

def fragment_shader_2d(interpolated_local_coord_p, color):
    d2 = dot(interpolated_local_coord_p, interpolated_local_coord_p)
    return color * math.exp(-d2)

def render_gaussians_2d(gaussians, quad_vertices):
    valid_gaussians = [g for g in gaussians if g.sigma >= 1.0 / 255.0]
    for g in valid_gaussians:
        for v in quad_vertices:
            local_coord = vertex_shader_2d(g.sigma, v)
            if local_coord is None: continue
            # Hardware interpolation happens here; simulated as direct pass for single vertex
            p = local_coord 
            frag_color = fragment_shader_2d(p, g.color)
            # Write to screen buffer

def render_gaussians_3d(gaussians, volume, inverse_size, inverse_variance, normalization_constant):
    # Bind FBO to 3D texture
    # glFramebufferTexture(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, volume.TextureHandle, 0)
    for g in gaussians:
        v_instance = g.instance_id
        v_position = g.position.xy
        
        gl_layer = v_instance
        g_position_z = 1.0 - 2.0 * v_instance * inverse_size
        g_position_xy = v_position
        
        g_position = [g_position_xy[0], g_position_xy[1], g_position_z]
        frag_color = fragment_shader_3d(g_position, g.color, inverse_variance, normalization_constant)
        # Write to volume slice gl_layer

def fragment_shader_3d(g_position, color, inverse_variance, normalization_constant):
    r2 = dot(g_position, g_position)
    return color * normalization_constant * math.exp(r2 * inverse_variance)
```

**Parameters:**
*   `InverseSize`: Used in 3D splatting to map instance ID to Z-coordinate. Value: `1.0 / depth` [S2, unofficial].
*   `InverseVariance`: Controls the spread of the Gaussian in 3D splatting. Value: Not given in the sources.
*   `NormalizationConstant`: Scaling factor for the Gaussian output in 3D splatting. Value: Not given in the sources.
*   `Threshold`: Minimum opacity for culling. Value: `1/255` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The cutoff radius calculation `r = sqrt(ln(255 * sigma))` requires a positive argument. If `sigma` is very small, `ln(255 * sigma)` may underflow or become negative, leading to NaN or culling. The pre-processing culling step handles `sigma < 1/255` [S1, unofficial].
*   **Boundaries:** The 3D texture uses `GL_CLAMP_TO_EDGE` to handle boundary conditions [S2, unofficial].
*   **Degenerate Input:** Gaussians with `sigma < 1/255` are culled, so they do not contribute to the output [S1, unofficial].

## Subtopics

### [Flowline Extrusion](1621-flowline-extrusion.md)
A technique to extrude a path of positions into a velocity field by looping over positions, computing velocity vectors, and splatting them using instanced rendering.

### [Volumetric Spot Noise Rendering](2360-volumetric-spot-noise-rendering.md)
Volumetric Spot Noise Rendering is an interactive method for synthesizing procedural 3D shell textures (volumetric surface details like hair, grass, or fur) by projecting random impulse distributions onto arbitrary surfaces and rendering them as sums of anisotropic Gaussian kernels. It solves the problem of efficiently rendering millions of semi-transparent, oriented primitives without the performance penalty of per-frame depth sorting or the aliasing artifacts of simple geometry instancing. It sets itself apart from its parent [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md) by using a hybrid pipeline that combines geometry instancing for depth/opacity with EWA splatting and Order Independent Transparency (OIT) for color, specifically addressing the "popping" and "halo" artifacts common in pure OIT or sorting-based approaches.

## Operating characteristics
*   **Time Complexity:** Linear in the number of Gaussians $O(N)$ for the rasterization stage, as each Gaussian is processed in a single draw call [S1, unofficial].
*   **Memory Complexity:** $O(N/WG)$ auxiliary storage, where $N$ is the number of Gaussians and $WG$ is the workgroup size, avoiding the large buffers required by tile-based rasterizers [S1, unofficial].
*   **Measured Performance:** WebSplatter achieves better scalability in web environments on heterogeneous devices (laptops, mobile phones) by avoiding the heavy memory traffic associated with generating and sorting tile-to-splat buffers [S1, unofficial].
*   **Strengths:** Avoids tile-based rasterizer overhead; dynamically sizes quads for tight bounds; efficient evaluation of Gaussian function via hardware interpolators [S1, unofficial].
*   **Limitations:** Relies on standard hardware pipeline, which may not be as optimized as custom CUDA tile-based rasterizers for specific hardware; requires pre-processing culling [S1, unofficial].
*   **When to use:** When rendering millions of Gaussians in a web environment or on heterogeneous devices where memory bandwidth is a bottleneck [S1, unofficial].
*   **When not to use:** When maximum performance on specialized hardware (e.g., high-end desktop GPUs with custom rasterizers) is required and memory traffic is not a concern [S1, unofficial].

## Background
*   **Origin:** The 2D splatting approach was introduced in "WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU" [S1, unofficial]. The 3D volume splatting approach was described in "Volumetric Splatting at The Little Grasshopper" [S2, unofficial].
*   **History:** The technique evolved from traditional volume rendering methods, adapting instancing and procedural generation to handle the specific requirements of Gaussian splatting, such as dynamic sizing and opacity-based culling [S1, S2, unofficial].
*   **Usage:** Used in web-based Gaussian splatting viewers (WebSplatter) and for generating volumetric data like velocity fields [S1, S2, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
[S2] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
