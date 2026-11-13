# Noise Splats

Also called: Textured Splats with Noise

## Overview
Noise Splats is a multivariate volume rendering technique that extends textured splatting by embedding controlled noise or discrete dots within the reconstruction texture [S1, unofficial]. It solves the problem of representing nonhomogeneous materials (e.g., translucent granite) or synthesizing realistic clouds and fog by using the local amount of noise as a second independent scalar variable [S1, unofficial]. Unlike standard textured splatting which uses anisotropic textures for vector fields, Noise Splats embeds noise directly into the scalar splat footprint, allowing the variance of the rendered image to encode additional data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A volume dataset defined on an integer lattice with two scalar fields per voxel: the primary density/intensity and a secondary noise amplitude $w \in [0, 1]$ [S1, unofficial]. A precomputed table of noise textures (splats) with increasing noise amplitude [S1, unofficial].
*   **Outputs:** A 2D image where the mean intensity represents the primary variable and the local variance represents the secondary variable [S1, unofficial].

**Data Structures:**
*   **Noise Texture Table:** A set of 2D textures $T_k$, where $k$ indexes the noise amplitude level [S1, unofficial]. Each texture represents a single splat footprint with embedded noise [S1, unofficial]. The size of the table is user-specified [S1, unofficial]. Textures are computed on the fly when data or settings change [S1, unofficial].
*   **Splat Point:** Defined by position $\mathbf{p}$, primary color/intensity $C$, and noise amplitude $w$ [S1, unofficial].

**Algorithm Steps:**

1.  **Precompute Noise Textures:** Generate a table of splat textures $T_k$ for $k=0 \dots N-1$ [S1, unofficial]. Each texture represents the reconstruction kernel $h(r)$ modulated by a specific noise amplitude [S1, unofficial]. The number of textures $N$ is user-defined [S1, unofficial]. Three distinct random variants are calculated for each number of points to avoid regular patterns, and one is chosen randomly at runtime [S1, unofficial].
2.  **Select Splat for Rendering:** For each voxel/splat point with noise amplitude $w$, map $w$ to an index in the texture table [S1, unofficial]. If $w=0$, select the noise-free splat. If $w=1$, select the highest amplitude splat [S1, unofficial].
3.  **Render:** Transform the splat to screen space [S1, unofficial]. Rasterize the selected texture $T_k$ onto the frame buffer using the **modulate** blend function in OpenGL (or equivalent hardware blending) [S1, unofficial]. The splat color is modulated by the texture's noise pattern [S1, unofficial].
4.  **Mip-mapping (Optional):** Apply mip-mapping to the noise textures [S1, unofficial]. This ensures that as a splat's size approaches a single pixel, the noise fades into the reconstruction function's average, or remains visible depending on user preference [S1, unofficial].

**Formulas:**

*   **Noisy Splat Function (White Noise Variant):**
    $$ h_{\text{noisy}}(r) = h(r) (1 + w \cdot \text{Random}) - w $$
    Where:
    *   $h(r)$ is the base reconstruction footprint function [S1, unofficial].
    *   $w$ is the noise amplitude parameter ($0 \le w \le 1$) [S1, unofficial].
    *   $\text{Random}$ is a white noise value with mean zero [S1, unofficial].
    *   The term $-w$ shifts the baseline, while the noise is windowed by $h(r)$ to avoid harsh edge effects and ensure correct amplitude when neighboring splats composite [S1, unofficial].
    *   If $w$ is large, the footprint may need scaling by $(1-w)$ to maintain a valid color range, though this is usually unnecessary for semi-transparent splats with moderate $w$ [S1, unofficial].

*   **Potential for Jittered Dithering (Point Distribution Variant):**
    $$ D_{i,j} = \frac{4D_{i-1,j} + D_{i-1,j-1} + 2D_{i,j-1} + D_{i+1,j-1}}{8} + R $$
    Where:
    *   $D_{i,j}$ is the potential at grid cell $(i,j)$ [S1, unofficial].
    *   $R$ is a random value between 0 and 1 [S1, unofficial].
    *   A point is selected at $(i,j)$ if $D_{i,j} > 0.5$, and $D_{i,j}$ is reduced by 1 [S1, unofficial].

*   **Weighted Point Density (Poisson Disc Variant):**
    The threshold for point selection is inversely proportional to the splat footprint $h(r)$ [S1, unofficial]. This creates a higher point density near the center of the splat that gradually fades to zero at the edges, ensuring the summation of expected densities from composite splats is fairly constant [S1, unofficial].

**Parameters:**
*   **Noise Amplitude ($w$):** Varies between 0 and 1. A value of 0 selects a splat with no noise, while a value of 1 selects the splat with the highest noise amplitude [S1, unofficial].
*   **Texture Table Size ($N$):** Number of splats in the precomputed table [S1, unofficial].
*   **Maximum Amplitude:** User-specified limit for the noise in the table [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overlap Artifacts:** If the overlap of reconstruction kernels is ignored when placing dots, bands with a sharp increase or decrease in the density of dots will appear at the seams of composite splats [S1, unofficial].
*   **Clumping:** Strictly random or pseudo-random point distributions cause clumping and make data interpretation inaccurate [S1, unofficial]. Poisson disc or jittered dithering must be used to ensure a uniform distribution [S1, unofficial].
*   **Precision:** The noise must have a mean of zero to preserve the overall intensity of the volume rendering [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import numpy as np

def generate_footprint(size):
    """Generates a 2D Gaussian reconstruction kernel h(r)."""
    y, x = np.ogrid[:size, :size]
    cx, cy = size / 2, size / 2
    r = np.sqrt((x - cx)**2 + (y - cy)**2)
    return np.exp(-r**2 / (2 * (size/4)**2))

def generate_white_noise(size):
    """Generates a 2D array of white noise with mean zero."""
    return np.random.uniform(-1.0, 1.0, size=(size, size))

def apply_mipmap(texture, target_size):
    """Downsamples texture to target_size using average pooling."""
    if texture.shape[0] <= target_size:
        return texture
    factor = texture.shape[0] // target_size
    return np.mean(texture[:target_size*factor, :target_size*factor].reshape(target_size, factor, target_size, factor), axis=(1, 3))

def rasterize_texture(texture, position, color):
    """Simulates OpenGL modulate blend: framebuffer *= texture * color."""
    # In a real renderer, this writes to the frame buffer at `position`.
    # Here we simulate the multiplication step.
    modulated = texture * color
    return modulated

def precompute_noise_textures(num_textures, texture_size, max_amplitude):
    """
    Generates a table of noise textures with 3 random variants per index.
    """
    texture_table = []
    for k in range(num_textures):
        w = (k + 1) / num_textures * max_amplitude
        h = generate_footprint(texture_size)
        variants = []
        for _ in range(3):
            noise_map = generate_white_noise(texture_size)
            # Formula: h(r) (1 + w * Random) - w
            noisy_h = h * (1 + w * noise_map) - w
            variants.append(noisy_h)
        texture_table.append(variants)
    return texture_table

def render_noise_splat(splat_point, texture_table, mip_map_enabled):
    """
    Renders a single splat point using the noise splat technique.
    """
    w = splat_point.noise_amplitude  # 0 <= w <= 1
    # Map w to index in texture table
    index = int(w * (len(texture_table) - 1))
    # Randomly select one of 3 variants to avoid regular patterns
    variant = random.randint(0, 2)
    texture = texture_table[index][variant]
    # Apply mip-mapping if enabled and splat is small
    if mip_map_enabled and splat_point.screen_size < texture.shape[0]:
        texture = apply_mipmap(texture, splat_point.screen_size)
    # Rasterize using modulate blend
    return rasterize_texture(texture, splat_point.screen_position, splat_point.color)
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Real-time interactive rendering is possible due to hardware acceleration via OpenGL rasterization [S1, unofficial]. Poisson disc generation is computationally expensive, so jittered dithering or white noise are preferred for performance [S1, unofficial].
*   **Strengths:** Allows visualization of multivariate data (e.g., velocity magnitude and variance) in a single pass. Creates realistic textures for clouds and fog [S1, unofficial].
*   **Limitations:** Requires precomputation of textures. Overlap artifacts can occur if noise distribution is not carefully weighted [S1, unofficial].

## Background
Noise Splats was introduced to extend volume rendering to include nonhomogeneous volume representations, such as translucent granite or realistic clouds and fog [S1, unofficial]. It builds on the Textured Splats technique by Crawfis and Max [S1, unofficial]. The method uses Poisson disc distribution, jittered dithering (Mitchell), and white noise modulation to embed noise into the reconstruction kernel [S1, unofficial].

## Sources
[S1] Multivariate volume rendering, https://www.osti.gov/biblio/495676, fetched 2026-10-26 (unofficial)
