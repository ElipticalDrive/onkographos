# Deferred Reflection in 3D Gaussian Splatting

Also called: 3D Gaussian Splatting with Deferred Reflection, Deferred 3D Gaussian Splatting for Reflection, Deferred Reflection, Deferred Shading with 3DGS, Ours, deferred, Per-pixel Reflection in 3DGS

## Overview
Deferred Reflection is a two-pass rendering technique for 3D Gaussian Splatting that computes specular reflections per-pixel in screen space rather than per-Gaussian [S1]. It solves the problem of high-frequency radiance fields being difficult to fit stably and the bottleneck of normal estimation caused by discontinuous gradients in forward shading [S1]. Unlike forward shading methods that compute reflection colors on individual Gaussians, this method separates Gaussian splatting from reflection shading, allowing pixel-precise normal maps to guide gradient propagation and environment map optimization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian parameters $\Theta_i$ (position, covariance, opacity), view-dependent spherical harmonic (SH) coefficients for base color, an initial environment map, and a spherical domain $\Omega$ covering the foreground object [S1].
*   **Outputs:** Final rendered image color $C_{final}$, optimized environment map, and optimized Gaussian parameters including normals and reflection strength [S1].

**Data Structures:**
The method uses viewport-aligned textures (screenspace maps) generated in Pass 1. These buffers store:
1.  **Base Color ($C_{base}$):** The diffuse/base color of the Gaussian soup.
2.  **Normal ($N$):** The surface normal vector.
3.  **Reflection Strength ($R$):** A scalar parameter weighting the contribution of the reflection [S1].

**Parameters:**
*   **Reflection Strength ($R_i$):** A scalar associated with each Gaussian $i$, learned during training. It weights the base and reflection colors in the final composition [S1].
*   **Normal Vector:** Regarded as the shortest axis of each Gaussian ellipsoid [S1].
*   **Environment Map:** A high-frequency lookup table representing lighting, learned during training [S1].
*   **Spherical Domain $\Omega$:** A constraint restricting the deferred reflection stage to Gaussians inside this domain to reduce interference from background objects during training [S1].

**Algorithm Steps:**

1.  **Pass 1: Gaussian Splatting Pass (G-Buffer Generation)**
    For each Gaussian $i$, compute its screen-space projection and alpha weight $w_i$ relative to the view direction $\mathbf{v}$, following the standard 3DGS splatting formulation [S1]:
    $$ C(\mathbf{v}) = \sum_i c_i(\mathbf{v}) G(\Theta_i, \mathbf{v}) $$
    where $c_i(\mathbf{v})$ is the base color and $G(\Theta_i, \mathbf{v})$ is the Gaussian weight [S1].
    Instead of computing the final color, blend the Gaussian properties into the screenspace maps:
    *   Accumulate base color $C_{base}$ weighted by opacity and Gaussian weight.
    *   Accumulate normal $N$ weighted by opacity and Gaussian weight.
    *   Accumulate reflection strength $R$ weighted by opacity and Gaussian weight.
    *   Normalize the accumulated normal $N$ to unit length [S1].

2.  **Pass 2: Pixel Shading Pass (Deferred Reflection)**
    For each pixel $(u, v)$ in the image:
    *   Retrieve the blended base color $C_{base}(u,v)$, normal $N(u,v)$, and reflection strength $R(u,v)$ from the G-buffers.
    *   Compute the reflection direction $\mathbf{r}$ using the view direction $\mathbf{v}$ and the normal $N(u,v)$:
        $$ \mathbf{r} = 2(\mathbf{v} \cdot N(u,v))N(u,v) - \mathbf{v} $$
        (Assuming standard reflection geometry where $\mathbf{v}$ is the vector from the camera to the pixel, or its negative depending on coordinate system convention; the source implies standard reflection direction calculation) [S1].
    *   Query the environment map $E(\mathbf{r})$ using the reflection direction $\mathbf{r}$ to acquire the specular reflection color $C_{refl}$ [S1].
    *   Compute the final color $C_{final}$ as the sum of the base and reflection colors weighted by the reflection strength:
        $$ C_{final}(u,v) = C_{base}(u,v) + R(u,v) \cdot C_{refl} $$
        [S1].

3.  **Training: Normal Propagation**
    *   **Observation:** Gaussians with large reflective strength values $R_i$ have near-correct normal vectors [S1].
    *   **Expansion:** These reflective Gaussians are expanded to propagate their normal vectors to nearby Gaussians [S1].
    *   **Overlap Mechanism:** When a Gaussian with a near-correct normal overlaps a different Gaussian, shared pixels receive near-correct normals. These pixels get meaningful normal gradients from the image color loss, helping to optimize the overlapping Gaussian's normal [S1].
    *   **Gradient Flow:** The deferred shading model blends Gaussian properties into viewport-aligned textures. The blended input values evaluate reflection and compose base/reflection colors into the final color. Image color loss feeds gradient back to input values of the same pixel, creating a gradient channel from color to blended normal to individual Gaussian normal. This facilitates information flow between normals of overlapping Gaussians [S1].

4.  **Domain Restriction**
    *   Restrict the deferred reflection stage to Gaussians inside the spherical domain $\Omega$ covering the foreground object. This reduces interference from background objects during training [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Gaussian Splatting (G-Buffer Generation)
def pass1_gaussian_splatting(gaussians, camera, image_width, image_height):
    # Initialize G-buffers
    base_color_buffer = zeros((image_height, image_width, 3))
    normal_buffer = zeros((image_height, image_width, 3))
    reflection_strength_buffer = zeros((image_height, image_width, 1))
    alpha_buffer = zeros((image_height, image_width, 1))

    for gaussian in gaussians:
        # Compute screen-space projection and weight w_i
        # This follows standard 3DGS splatting logic [S1]
        screen_coords, weight, alpha = project_gaussian(gaussian, camera)
        
        # Get base color c_i(v) from SH coefficients [S1]
        c_base = get_sh_color(gaussian.sh_coeffs, camera.view_direction)
        
        # Get normal N_i (shortest axis of ellipsoid) [S1]
        n_i = get_shortest_axis(gaussian.covariance)
        
        # Get reflection strength R_i [S1]
        r_i = gaussian.reflection_strength

        # Check if inside spherical domain Omega [S1]
        if not is_inside_domain(gaussian.position, domain_Omega):
            continue

        # Blend into buffers
        # Accumulate weighted values
        base_color_buffer[screen_coords] += c_base * weight * alpha
        normal_buffer[screen_coords] += n_i * weight * alpha
        reflection_strength_buffer[screen_coords] += r_i * weight * alpha
        alpha_buffer[screen_coords] += weight * alpha

    # Normalize normals in the buffer
    for y in range(image_height):
        for x in range(image_width):
            if alpha_buffer[y, x] > 0:
                normal_buffer[y, x] = normalize(normal_buffer[y, x])
    
    return base_color_buffer, normal_buffer, reflection_strength_buffer, alpha_buffer

# Pass 2: Pixel Shading (Deferred Reflection)
def pass2_pixel_shading(base_color_buffer, normal_buffer, reflection_strength_buffer, env_map, camera):
    image_height, image_width, _ = base_color_buffer.shape
    final_image = zeros((image_height, image_width, 3))

    for y in range(image_height):
        for x in range(image_width):
            c_base = base_color_buffer[y, x]
            n = normal_buffer[y, x]
            r = reflection_strength_buffer[y, x]
            
            # Compute reflection direction r_dir
            # v is view direction from camera to pixel
            v = camera.get_view_direction_at_pixel(x, y)
            r_dir = 2 * dot(v, n) * n - v
            
            # Query environment map
            c_refl = sample_env_map(env_map, r_dir)
            
            # Compose final color
            final_image[y, x] = c_base + r * c_refl
            
    return final_image

# Training Step
def train_step(gaussians, camera, env_map, gt_image, domain_Omega):
    # Pass 1
    base_color, normal, refl_strength, alpha = pass1_gaussian_splatting(gaussians, camera, ...)
    
    # Pass 2
    pred_image = pass2_pixel_shading(base_color, normal, refl_strength, env_map, camera)
    
    # Compute loss (e.g., L1 + SSIM) between pred_image and gt_image [S1]
    loss = compute_loss(pred_image, gt_image)
    
    # Backpropagate loss
    # Gradients flow from color -> blended normal -> individual Gaussian normal [S1]
    # Gradients flow to env_map, reflection_strength, and gaussian parameters [S1]
    loss.backward()
    
    # Normal Propagation Logic (Implicit in gradient flow, but explicitly mentioned as a mechanism)
    # Gaussians with large reflection strength have near-correct normals.
    # These propagate to overlapping Gaussians via shared pixels [S1].
    pass
```

## Operating characteristics
*   **Performance:** Runs at a frame rate almost identical to vanilla Gaussian Splatting [S1]. On the Shiny Blender Dataset, training time is 16 minutes, and rendering frame rate is 251 FPS. On real-world scenes, training time is 47 minutes, and rendering frame rate is 80 FPS [S1].
*   **Quality:** Demonstrates consistent improvement in PSNR for both synthetic and real-world scenes compared to forward shading and other baselines [S1]. On the Shiny Blender Dataset, it achieves PSNR 33.66, SSIM 0.979, and LPIPS 0.098, compared to forward shading which has lower metrics (Table 1 in [S1]) [S1].
*   **Efficiency:** Uses significantly fewer Gaussians than forward shading or other baselines. For example, on the 'ball' scene, it uses 27.2k Gaussians vs 76.4k (forward) vs 76.5k (GShader) vs 199k (3DGS) [S1].
*   **Strengths:** Produces sharp reflections with pixel-level details. Eliminates interpolation artifacts caused by discontinuous color changes at Gaussian boundaries. Stabilizes reflection strength and environment map training by smoothing gradients through extra per-pixel samples [S1].
*   **Limitations:** Environment map queries demand high precision on normal vectors but provide scarcely useful gradients to refine them, making normal estimation difficult [S1]. The semi-transparent Gaussian soup has a loosely-defined surface, complicating accurate normal estimation [S1].
*   **When to use:** For novel view synthesis and real-time rendering of specular reflection effects where high-quality reflections and efficient Gaussian count are desired [S1].
*   **When not to use:** For non-specular scenes, the method provides negligible benefit over vanilla 3DGS (Table 6 in [S1] shows similar metrics for non-specular scenes like drums, ficus, etc.) [S1].

## Background
*   **Origin:** Introduced by Keyang Ye, Qiming Hou, and Kun Zhou at Zhejiang University [S1].
*   **Publication:** Published in SIGGRAPH Conference Papers '24, July 27-August 1, 2024 [S1].
*   **History:** Deferred shading [Deering et al. 1988] is a classical real-time rendering technique that computes high-frequency shading effects like specularity per-pixel in screen space after baking scene properties like normals and colors into buffers [S1]. This method combines deferred shading with 3D Gaussian Splatting to effectively render specular reflection, producing accurate estimation of normal and environment maps [S1]. It addresses the limitations of per-Gaussian shading (e.g., GaussianShader [Jiang et al. 2023]) which computes reflection on individual Gaussians, leading to independent normal gradients that cannot influence each other and interpolation artifacts [S1].

## Sources
[S1] 3D Gaussian Splatting with Deferred Reflection, https://doi.org/10.1145/3641519.3657456, fetched 2026-10-26
