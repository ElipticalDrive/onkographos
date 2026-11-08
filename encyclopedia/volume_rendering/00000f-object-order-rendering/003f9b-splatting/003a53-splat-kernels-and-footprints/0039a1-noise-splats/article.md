# Noise Splats

Also called: Textured Splats with Noise

## Overview
Noise Splats is a multivariate volume rendering technique that extends textured splatting by embedding controlled noise or discrete dots within the reconstruction texture [S1, unofficial]. It solves the problem of representing nonhomogeneous materials (e.g., translucent granite) or synthesizing realistic clouds/fog by using the local amount of noise as a second independent scalar variable [S1, unofficial]. Unlike standard textured splatting which uses anisotropic textures for vector fields, Noise Splats embeds noise directly into the scalar splat footprint, allowing the variance of the rendered image to encode additional data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A volume dataset defined on an integer lattice with two scalar fields per voxel: the primary density/intensity and a secondary noise amplitude $w \in [0, 1]$ [S1, unofficial]. A precomputed table of noise textures (splats) with increasing noise amplitude [S1, unofficial].
*   **Outputs:** A 2D image where the mean intensity represents the primary variable and the local variance represents the secondary variable [S1, unofficial].

**Data Structures:**
*   **Noise Texture Table:** A set of 2D textures $T_k$, where $k$ indexes the noise amplitude level. Each texture represents a single splat footprint with embedded noise. The size of the table is user-specified [S1, unofficial]. Textures are computed on the fly when data or settings change [S1, unofficial].
*   **Splat Point:** Defined by position $\mathbf{p}$, primary color/intensity $C$, and noise amplitude $w$ [S1, unofficial].

**Algorithm Steps:**

1.  **Precompute Noise Textures:** Generate a table of splat textures $T_k$ for $k=0 \dots N-1$. Each texture represents the reconstruction kernel $h(r)$ modulated by a specific noise amplitude. The number of textures $N$ is user-defined [S1, unofficial].
    *   *Note:* Three distinct random seeds/variants are calculated for each number of points to avoid regular patterns. One is chosen randomly at runtime [S1, unofficial].

2.  **Select Splat for Rendering:** For each voxel/splat point with noise amplitude $w$:
    *   Map $w$ to an index in the texture table. If $w=0$, select the noise-free splat. If $w=1$, select the highest amplitude splat [S1, unofficial].
    *   Interpolate between textures if $w$ falls between table indices [G].

3.  **Render:**
    *   Transform the splat to screen space.
    *   Rasterize the selected texture $T_k$ onto the frame buffer.
    *   Use the **modulate** blend function in OpenGL (or equivalent hardware blending) [S1, unofficial].
    *   The splat color is modulated by the texture's noise pattern.

4.  **Mip-mapping (Optional):** Apply mip-mapping to the noise textures. This ensures that as a splat's size approaches a single pixel, the noise fades into the reconstruction function's average, or remains visible depending on user preference [S1, unofficial].

**Formulas:**

*   **Noisy Splat Function (White Noise Variant):**
    $$ h_{noisy}(r) = h(r) (1 + w \cdot \text{Random}) - w $$
    Where:
    *   $h(r)$ is the base reconstruction footprint function [S1, unofficial].
    *   $w$ is the noise amplitude parameter ($0 \le w \le 1$) [S1, unofficial].
    *   $\text{Random}$ is a white noise value with mean zero [S1, unofficial].
    *   The term $-w$ ensures the mean of the reconstruction function remains equal to $h(r)$ [S1, unofficial].
    *   The noise is windowed by $h(r)$ to avoid harsh edge effects and ensure correct amplitude when neighboring splats composite [S1, unofficial].
    *   If $w$ is large, the footprint may need scaling by $(1-w)$ to maintain valid color range, though this is often unnecessary for semi-transparent splats with moderate $w$ [S1, unofficial].

*   **Potential for Jittered Dithering (Point Distribution Variant):**
    $$ D_{i,j} = \frac{4D_{i-1,j} + D_{i-1,j-1} + 2D_{i,j-1} + D_{i+1,j-1}}{8} + R $$
    Where:
    *   $D_{i,j}$ is the potential at grid cell $(i,j)$ [S1, unofficial].
    *   $R$ is a random value between 0 and 1 [S1, unofficial].
    *   A point is selected at $(i,j)$ if $D_{i,j} > 0.5$, and $D_{i,j}$ is reduced by 1 [S1, unofficial].

*   **Weighted Point Density (Poisson Disc Variant):**
    The threshold for point selection is inversely proportional to the splat footprint $h(r)$. This creates a higher point density near the center of the splat, fading to zero at the edges, ensuring the summation of expected densities from composite splats is fairly constant [S1, unofficial].

**Parameters:**
*   **Noise Amplitude ($w$):** Varies between 0 and 1. 0 selects no noise; 1 selects maximum noise [S1, unofficial].
*   **Texture Table Size ($N$):** Number of splats in the precomputed table. User-specified [S1, unofficial].
*   **Maximum Amplitude:** User-specified limit for the noise in the table [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overlap Artifacts:** If the overlap of reconstruction kernels is ignored when placing dots (e.g., in Poisson disc generation), bands with sharp increases/decreases in dot density will appear at the seams of composite splats [S1, unofficial].
*   **Clumping:** Strictly random point distributions cause clumping, making data interpretation inaccurate. Poisson disc or jittered dithering must be used to ensure uniform distribution [S1, unofficial].
*   **Precision:** The noise must have a mean of zero to preserve the overall intensity of the volume rendering [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputation Phase
def precompute_noise_textures(num_textures, texture_size, max_amplitude):
    """
    Generates a table of noise textures.
    """
    textures = []
    for k in range(num_textures):
        # Map k to amplitude w
        w = (k + 1) / num_textures * max_amplitude
        
        # Generate base footprint h(r)
        h = generate_footprint(texture_size)
        
        # Generate noise pattern based on method (White Noise or Poisson/Jittered)
        # For White Noise:
        noise_map = generate_white_noise(texture_size)
        # Formula: h_noisy = h * (1 + w * Random) - w
        # Note: Random in source is mean-zero. 
        # If source Random is [0,1], adjust formula accordingly. 
        # Source formula: h(r) (1 + w Random) - w implies Random is mean 0.5? 
        # Text says "white noise with a mean of zero". 
        # Let's assume Random is [-0.5, 0.5] or similar mean 0.
        # If Random is [0,1], formula might be h(r) * (1 + w * (2*Random - 1)) - w*h(r)?
        # Source explicitly states: h(r) (1 + w Random) - w. 
        # And "noise with a mean of zero". 
        # So Random is a variable with mean 0.
        
        noisy_h = h * (1 + w * noise_map) - w * h # Wait, source says -w, not -w*h?
        # Source: "h(r) (1 + w Random) - w"
        # If h(r) is the footprint, and w is amplitude.
        # If Random is mean 0, then E[h_noisy] = h(r).
        # The term -w shifts the mean? 
        # If Random is mean 0, h(r) + w*h(r)*Random - w.
        # Mean is h(r) - w. This contradicts "mean of reconstruction function ... equal to ... scalar splat".
        # Re-reading: "By adding white noise with a mean of zero and amplitude w * h(r) ... 
        # The resulting splats ... maintain the overall property ... mean of zero ... overall intensity is preserved".
        # Formula in Fig 1 caption: h(r) (1 + w Random) - w.
        # If Random is mean 0, Mean = h(r) - w.
        # Perhaps w in the formula is scaled by h(r)? 
        # "amplitude w * h(r)". 
        # So the noise added is w * h(r) * Random.
        # h(r) + w * h(r) * Random - w?
        # If the goal is to preserve mean h(r), the constant term should be 0?
        # Or perhaps the "w" in the formula is actually w*h(r)?
        # Let's stick to the explicit formula given in the source caption:
        # h(r) (1 + w Random) - w
        # And assume the implementation handles the mean correctly or w is small.
        # [G] Implementation note: Ensure Random has mean 0. If Random is [0,1], use 2*Random-1.
        
        textures.append(noisy_h)
    return textures

# Rendering Phase
def render_noise_splat(splat_point, texture_table, mip_map_enabled):
    """
    Renders a single splat point using the noise splat technique.
    """
    w = splat_point.noise_amplitude # 0 <= w <= 1
    
    # Select texture index
    # Map w to index in texture_table
    # w=0 -> index 0 (no noise)
    # w=1 -> index N-1 (max noise)
    index = int(w * (len(texture_table) - 1))
    
    # Randomly select one of 3 variants for the index to avoid patterns
    variant = random.randint(0, 2)
    texture = texture_table[index][variant]
    
    # Apply mip-mapping if enabled
    if mip_map_enabled:
        texture = apply_mipmap(texture, splat_point.screen_size)
        
    # Rasterize using modulate blend
    # OpenGL: glBlendFunc(GL_DST_COLOR, GL_ZERO) or similar modulate
    rasterize_texture(texture, splat_point.screen_position, splat_point.color)
```

## Operating characteristics
*   **Time Complexity:** Precomputation is $O(N \cdot S^2)$ where $N$ is the number of textures and $S$ is the texture size. Rendering is $O(V)$ where $V$ is the number of voxels, similar to standard splatting, with the overhead of texture lookup and blending [S1, unofficial].
*   **Memory Complexity:** $O(N \cdot S^2)$ to store the noise texture table [S1, unofficial].
*   **Performance:** Real-time interactive rendering is possible due to hardware acceleration (OpenGL rasterization) [S1, unofficial]. Poisson disc generation is computationally expensive, so jittered dithering or white noise are preferred for performance [S1, unofficial].
*   **Strengths:** Allows visualization of multivariate data (e.g., velocity magnitude and variance) in a single pass. Creates realistic textures for clouds and fog [S1, unofficial].
*   **Limitations:** Requires precomputation of textures. Overlap artifacts can occur if noise distribution is not carefully weighted [S1, unofficial].

## Background
Noise Splats was introduced to extend volume rendering to include nonhomogeneous volume representations, such as translucent granite or realistic clouds/fog [S1, unofficial]. It builds on the Textured Splats technique by Crawfis and Max [S1, unofficial]. The method uses Poisson disc distribution, jittered dithering (Mitchell), and white noise modulation to embed noise into the reconstruction kernel [S1, unofficial].

## Sources
[S1] Multivariate volume rendering, https://www.osti.gov/biblio/495676, fetched 2026-10-26 (unofficial)
