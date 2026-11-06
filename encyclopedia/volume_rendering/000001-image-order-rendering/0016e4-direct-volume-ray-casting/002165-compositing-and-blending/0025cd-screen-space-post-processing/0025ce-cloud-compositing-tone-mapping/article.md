# Cloud Compositing & Tone Mapping

Also called: ACES tone mapping, sky/ground/cloud blending

## Overview
This method composites pre-resolved volumetric cloud data with a procedural sky and ground background, then applies tone mapping and gamma correction to produce a final displayable image. It solves the problem of integrating discrete cloud render passes into a coherent scene by reconstructing 3D ray directions from screen-space UVs to determine appropriate background environments (sky vs. ground) and blending cloud radiance against them. It differs from [HDR Post-Processing Pipeline](4633-hdr-post-processing-pipeline.md) by handling the specific spatial logic of cloud-background interaction (horizon culling, procedural sky/ground generation) rather than just global image adjustments, and from [Bloom](4906-bloom.md) by focusing on radiance compositing rather than luminance-based glow.

## How it works
**Inputs:**
*   `uClouds`: `sampler2D` containing resolved cloud radiance (RGB) and transmittance (Alpha).
*   `uCloudDepth`: `sampler2D` containing depth information from the cloud pass (used for ray reconstruction context, though exact usage in compositing is implicit in the ray direction step).
*   `uInvViewProj`: `mat4` inverse view-projection matrix.
*   `uCamPos`: `vec3` camera position (used in debug mode).
*   `uSunDir`: `vec3` sun direction vector.
*   `uShape`: `sampler3D` (debug only).
*   `uShowNoise`: `int` (debug flag).
*   `vUv`: `vec2` fragment texture coordinates (0 to 1).

**Outputs:**
*   `outColor`: `vec4` final color output at location 0.

**Data Structures:**
*   The cloud texture `uClouds` is assumed to be a 2D texture where RGB channels hold the accumulated cloud radiance and the Alpha channel holds the accumulated transmittance (or opacity, depending on the specific compositing convention; the source code `background * clouds.a + clouds.rgb` implies `clouds.a` is the opacity/coverage factor and `clouds.rgb` is the emitted/scattered radiance).

**Steps:**

1.  **Ray Direction Reconstruction:**
    Compute the 3D ray direction for the current fragment using the inverse view-projection matrix and the fragment's NDC coordinates.
    *   Convert UV to NDC: $\text{ndc} = 2.0 \cdot \text{vUv} - 1.0$.
    *   Project near and far points in clip space:
        $$ p_{\text{near}} = uInvViewProj \cdot \begin{bmatrix} \text{ndc}_x \\ \text{ndc}_y \\ -1.0 \\ 1.0 \end{bmatrix}, \quad p_{\text{far}} = uInvViewProj \cdot \begin{bmatrix} \text{ndc}_x \\ \text{ndc}_y \\ 1.0 \\ 1.0 \end{bmatrix} $$
    *   Transform to view space and normalize:
        $$ \mathbf{rd} = \text{normalize}\left( \frac{p_{\text{far}}.xyz}{p_{\text{far}}.w} - \frac{p_{\text{near}}.xyz}{p_{\text{near}}.w} \right) $$

2.  **Background Selection and Computation:**
    Determine if the ray points towards the sky or the ground.
    *   **Condition:** If the ray's vertical component (Y-axis in view space, assuming standard OpenGL coordinate system where Y is up) is below the horizon and the camera is above the ground plane, select the ground background. Otherwise, select the sky background.
    *   **Ground Background:** Compute ground color using a checkerboard pattern modulated by exponential fog. The fog density and color are typically derived from weather parameters (not explicitly detailed in the compositing shader code but implied by the context of `Weather` struct in the source).
    *   **Sky Background:** Compute sky color using a sunset gradient function dependent on the sun's elevation and direction (`uSunDir`). This includes the sun disc and sun glow effects.

3.  **Cloud Compositing:**
    Sample the resolved cloud texture and blend it with the computed background.
    *   Sample clouds: $\mathbf{C} = \text{texture}(uClouds, \text{vUv})$.
    *   Composite:
        $$ \mathbf{Color} = \mathbf{Background} \cdot \mathbf{C}.a + \mathbf{C}.rgb $$
    *   Note: This assumes `C.a` represents the opacity (coverage) of the clouds and `C.rgb` represents the cloud radiance. The background is multiplied by the alpha, implying the background is visible through the transparent parts of the cloud volume, and the cloud radiance is added on top.

4.  **Tone Mapping and Gamma Correction:**
    Apply ACES tone mapping (as referenced by the alias and standard practice in this renderer's pipeline, though the specific ACES formula is not in the compositing shader code snippet, the alias "ACES tone mapping" suggests it is applied here or in a preceding step) and gamma correction.
    *   The source code explicitly shows gamma correction:
        $$ \mathbf{outColor} = \text{pow}(\mathbf{Color}, \frac{1.0}{2.2}) $$
    *   The alpha is set to 1.0.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: vUv (vec2), uClouds (sampler2D), uInvViewProj (mat4), uCamPos (vec3), uShowNoise (int), uShape (sampler3D)
# Outputs: outColor (vec4)

def main(vUv, uClouds, uInvViewProj, uCamPos, uShowNoise, uShape):
    # 1. Ray Direction Reconstruction
    ndc = vUv * 2.0 - 1.0
    p_near = uInvViewProj * vec4(ndc.x, ndc.y, -1.0, 1.0)
    p_far = uInvViewProj * vec4(ndc.x, ndc.y, 1.0, 1.0)
    
    # Transform to view space
    p_near_view = p_near.xyz / p_near.w
    p_far_view = p_far.xyz / p_far.w
    
    rd = normalize(p_far_view - p_near_view)
    
    # 2. Background Selection
    # Assuming standard OpenGL Y-up view space. 
    # Horizon check: if rd.y < 0, it points below the horizon.
    # Ground check: camera is above ground (assumed true for this method's context).
    
    if rd.y < 0.0:
        # Ground Background
        # Compute checkerboard and exponential fog
        # Note: Exact checkerboard/fog parameters are not in the compositing shader snippet
        # but are part of the renderer's weather/ground logic.
        background = compute_ground_background(rd)
    else:
        # Sky Background
        # Compute sunset gradient and sun disc/glow
        # Note: Sun direction and elevation are required but not passed as uniforms in this specific snippet
        # They are likely global or passed via other means in the full renderer.
        background = compute_sky_background(rd)
        
    # 3. Cloud Compositing
    clouds = texture(uClouds, vUv)
    # clouds.rgb is radiance, clouds.a is alpha/opacity
    color = background * clouds.a + clouds.rgb
    
    # Debug Mode: Slice through 3D shape texture
    if uShowNoise == 1:
        # Slice using camera X position as Z-slice
        z_slice = fract(uCamPos.x * 0.0001)
        noise_sample = texture(uShape, vec3(vUv, z_slice))
        # The source code does not specify how noise_sample is blended, 
        # but typically it might replace or modulate the color. 
        # Based on "debug only", we assume it's an overlay or replacement.
        # Since the source code snippet doesn't show the blend, we leave it as a comment.
        # color = noise_sample.rgb # Hypothetical debug output
        pass
        
    # 4. Tone Mapping and Gamma Correction
    # ACES Tone Mapping is implied by the alias and pipeline context.
    # The source code explicitly applies gamma correction.
    # If ACES is applied, it would happen before gamma.
    # Since the source code only shows pow(color, 1/2.2), we follow that.
    # [G] ACES tone mapping is typically applied to linear radiance before gamma.
    # If the source implies ACES is part of this step but doesn't show the formula,
    # we note that the explicit code is gamma correction.
    
    final_color = pow(color, vec3(1.0 / 2.2))
    
    outColor = vec4(final_color, 1.0)
    return outColor

def compute_ground_background(rd):
    # Placeholder for ground logic
    # Checkerboard pattern based on world position projected from ray
    # Exponential fog based on distance
    return vec3(0.0) # Not given in the sources.

def compute_sky_background(rd):
    # Placeholder for sky logic
    # Sunset gradient based on sun elevation
    # Sun disc and glow
    return vec3(0.0) # Not given in the sources.
```

## Subtopics

### [ACES Filmic Tone Mapping](4228-aces-filmic-tone-mapping.md)
A filmic tone mapping function that compresses HDR color into $[0, 1]$ using a characteristic shoulder curve to prevent highlight clipping. It differs from simple Reinhard tonemapping by providing a non-linear highlight roll-off rather than a basic division. It builds on [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md).

### [Tonemapping and Gamma Correction](5076-tonemapping-and-gamma-correction.md)
This method processes linear radiance values into a displayable sRGB image by applying exposure control, chroma preservation, black point correction, and gamma encoding. It solves the problem of high dynamic range (HDR) data exceeding display limits and the visual artifacts introduced by non-linear gamma encoding, such as shadow haze and desaturation of bright colors. It differs from [ACES Filmic Tone Mapping](4228-aces-filmic-tone-mapping.md) by using a simpler exponential compression and explicit post-processing steps for chroma and black level rather than a single parametric curve.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment, as it involves a constant number of texture samples, matrix multiplications, and arithmetic operations.
*   **Memory Complexity:** $O(1)$ additional memory, relying on the pre-resolved cloud texture and uniforms.
*   **Performance:** The method is lightweight and suitable for real-time rendering. The source code indicates a "resolve" pass time, suggesting this compositing step is part of a larger pipeline where cloud resolution is the heavier step.
*   **Strengths:** Simple integration of cloud data with background; handles horizon culling and procedural sky/ground generation efficiently.
*   **Limitations:** Relies on pre-resolved cloud data; debug mode requires 3D texture sampling which may be slower; gamma correction is hardcoded to 2.2.
*   **When to use:** When integrating pre-computed or pre-resolved volumetric clouds into a scene with a dynamic sky/ground background.
*   **When not to use:** When real-time cloud resolution is required (use the cloud resolution pass instead) or when more sophisticated tone mapping is needed (use ACES Filmic Tone Mapping).

## Background
This method is part of the Volumetric Cloud Renderer project by Aryan Harkawat [S1, unofficial]. It is designed to composite volumetric cloud data with a procedural environment, providing a realistic sky and ground background that interacts with the cloud layer. The method is used in the interactive rendering loop of the project, allowing users to adjust weather parameters and view the results in real-time.

## Sources
[S1] harkawataryan/Volumetric-Cloud-Renderer, https://github.com/harkawataryan/Volumetric-Cloud-Renderer, fetched 2026-10-27 (unofficial)
