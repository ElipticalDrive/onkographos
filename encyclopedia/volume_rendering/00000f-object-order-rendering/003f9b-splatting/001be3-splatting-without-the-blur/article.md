# Splatting Without the Blur

Also called: Gradient splats, Gradient splatting, Post-Classified Splatting, Post-projection classification splatting, Post-shaded splatting, Splatting Without The Blur [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]

## Overview
Splatting Without the Blur is a volume rendering technique that performs classification and shading in screen space, after voxels have been projected onto the image plane, to avoid the edge blurring inherent in traditional pre-shaded splatting [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]. Unlike [Pre-Classified Splatting](5410-pre-classified-splatting.md), which applies optical properties before projection and results in fuzzy edges due to kernel smoothing, this method preserves crisp object boundaries by deferring classification until after the density footprint is rasterized [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]. It distinguishes itself from [Edge Preservation in Volume Rendering Using Splatting](6573-edge-preservation-in-volume-rendering-using-splatting.md) by manipulating the rendering pipeline stages rather than warping the reconstruction kernels themselves [S1, unofficial].

## How it works
The method modifies the standard splatting pipeline by separating the projection of raw density from the computation of color and opacity [S1, unofficial][S2, abstract only, unofficial]. It requires an image-aligned sheet buffer to store intermediate results, as axis-aligned buffers cannot support the per-pixel screen-space operations required for post-classification [S3, unofficial].

### Inputs and Outputs
*   **Input:** A raw scalar density volume $V(x,y,z)$ [S1, unofficial].
*   **Output:** A final 2D image with crisp edges and shaded surfaces [S1, unofficial][S2, abstract only, unofficial].

### Data Structures
1.  **Sheet Buffer:** An image-aligned buffer storing the accumulated density contributions [S3, unofficial].
2.  **Lookup Tables (LUTs):** Pre-computed tables for shading functions to mitigate the overhead of per-pixel gradient calculations [S1, unofficial][S2, abstract only, unofficial].

### Steps
1.  **Project Density:** Splat the raw density volume $V$ onto the image-aligned sheet buffer using standard radially symmetric interpolation kernels (regular splats) [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]. This creates a fuzzy density field on the screen [S1, unofficial].
2.  **Compute Screen Gradients:** Calculate the gradient of the projected density field on the screen [S1, unofficial][S2, abstract only, unofficial]. Two methods are described:
    *   **Gradient Splats:** Project a pre-computed gradient volume using specialized gradient splats [S1, unofficial][S2, abstract only, unofficial][S4, unofficial].
    *   **Central Differencing:** Compute gradients directly on the projected density data in screen space using central differencing [S1, unofficial][S2, abstract only, unofficial][S3, S4, unofficial]. This approach reduces the number of footprint rasterizations by a factor of four compared to using gradient splats, since only the density volume needs to be projected [S1, unofficial][S2, abstract only, unofficial][S4, unofficial].
3.  **Classify and Shade:** Perform per-pixel classification and shading using the computed screen gradients [S1, unofficial][S2, abstract only, unofficial][S3, S4, unofficial]. Pixels are classified based on their density against an iso-threshold [S2, abstract only, unofficial].
4.  **Composite:** Composite the sheet-buffer to form the final image [S3, unofficial].

### Formulas
Let $D(x,y)$ be the density value at screen coordinate $(x,y)$ accumulated in the sheet buffer [S1, unofficial]. The gradient $\nabla D(x,y)$ is computed via central differencing:
$$ \frac{\partial D}{\partial x} \approx \frac{D(x+1, y) - D(x-1, y)}{2} $$
$$ \frac{\partial D}{\partial y} \approx \frac{D(x, y+1) - D(x, y-1)}{2} $$
The magnitude $|\nabla D|$ determines the edge strength [S1, unofficial]. Classification $C(x,y)$ is applied:
$$ C(x,y) = \begin{cases} \text{Shaded Color} & \text{if } D(x,y) > \tau \\ 0 & \text{otherwise} \end{cases} $$
where $\tau$ is the iso-threshold [S2, abstract only, unofficial].

### Parameters
*   **Iso-threshold ($\tau$):** The density value below which pixels are set to zero during post-classification [S2, abstract only, unofficial].
*   **Lookup Tables:** Used to map gradient magnitude and density to color/opacity efficiently [S1, unofficial][S2, abstract only, unofficial].

### Edge Cases and Pitfalls
*   **Pre-shaded + Post-classification Artifacts:** Combining pre-shaded voxel projection with post-classification (setting pixels to zero if density is below threshold) causes severe artifacts [S2, abstract only, unofficial].
*   **Sheet Buffer Alignment:** Post-classified rendering requires image-aligned sheet buffers [S3, unofficial].
*   **Performance Overhead:** The calculation of screen gradients and per-pixel shading adds overhead compared to pre-shaded splatting [S1, unofficial][S2, abstract only, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatting_without_blur(volume, mvp_matrix, W, H, iso_threshold, kernel_radius, kernel_func):
    # Initialize image-aligned sheet buffer with zeros
    sheet_buffer = [[0.0] * W for _ in range(H)]

    # Step 1: Project raw density volume using regular splats
    for voxel in volume:
        # Transform voxel to screen space using MVP matrix
        sx, sy, sz = apply_mvp(mvp_matrix, voxel.position)
        # Splat density value into sheet buffer
        accumulate_splat(sheet_buffer, sx, sy, voxel.density, kernel_radius, kernel_func)

    # Step 2: Compute screen gradients using central differencing
    gradient_x = [[0.0] * W for _ in range(H)]
    gradient_y = [[0.0] * W for _ in range(H)]

    for y in range(1, H - 1):
        for x in range(1, W - 1):
            gradient_x[y][x] = (sheet_buffer[y][x+1] - sheet_buffer[y][x-1]) / 2.0
            gradient_y[y][x] = (sheet_buffer[y+1][x] - sheet_buffer[y-1][x]) / 2.0

    # Step 3: Classify and shade per-pixel
    final_image = [[0.0, 0.0, 0.0] for _ in range(H)]

    for y in range(H):
        for x in range(W):
            density = sheet_buffer[y][x]
            grad_mag = (gradient_x[y][x]**2 + gradient_y[y][x]**2)**0.5

            # Post-classification: discard pixels below iso-threshold
            if density < iso_threshold:
                continue

            # Compute shading based on gradient and density
            color = compute_shading(density, grad_mag)
            final_image[y][x] = color

    return final_image

def apply_mvp(matrix, point):
    # Standard 4x4 matrix multiplication for perspective projection
    x, y, z = point
    w = matrix[3][0]*x + matrix[3][1]*y + matrix[3][2]*z + matrix[3][3]
    sx = (matrix[0][0]*x + matrix[0][1]*y + matrix[0][2]*z + matrix[0][3]) / w
    sy = (matrix[1][0]*x + matrix[1][1]*y + matrix[1][2]*z + matrix[1][3]) / w
    sz = (matrix[2][0]*x + matrix[2][1]*y + matrix[2][2]*z + matrix[2][3]) / w
    return sx, sy, sz

def accumulate_splat(buffer, cx, cy, value, radius, kernel):
    for dy in range(-radius, radius + 1):
        for dx in range(-radius, radius + 1):
            x, y = int(cx + dx), int(cy + dy)
            if 0 <= x < len(buffer[0]) and 0 <= y < len(buffer):
                weight = kernel(dx, dy)
                buffer[y][x] += value * weight

def compute_shading(density, grad_mag):
    # Simplified shading: map density and gradient magnitude to RGB
    # In practice, this uses lookup tables [S1, unofficial][S2, abstract only, unofficial]
    r = min(1.0, density * 0.5)
    g = min(1.0, grad_mag * 0.5)
    b = 0.0
    return [r, g, b]
```

## Subtopics

### [Pre-Classified Splatting](5410-pre-classified-splatting.md)
Pre-classified splatting is a volume rendering strategy where classification and shading are applied to the raw density volume prior to the splatting process. This approach results in a blurred edge image upon reconstruction due to the subsequent smoothing kernel. It is distinguished from post-classified splatting, which performs classification after projection to preserve crisp edges, by its reliance on pre-computed optical properties.

### [Edge Preservation in Volume Rendering Using Splatting](6573-edge-preservation-in-volume-rendering-using-splatting.md)
Edge Preservation in Volume Rendering Using Splatting is a method to preserve sharp edge details in splatting for volume rendering by applying a non-linear warping to the footprints of conventional splats and building a table of footprints for different possible edge positions and edge strengths. Conventional splatting algorithms produce fuzzy images for views close to the volume model because overlapping interpolation kernels ensure a smooth transition from one voxel to another, preventing the preservation of edges. This method sets itself apart from siblings like [Splatting Without the Blur](1157-splatting-without-the-blur.md) and [Pre-Classified Splatting](5410-pre-classified-splatting.md) by manipulating the underlying reconstruction kernels and resulting footprints directly, rather than deferring classification or shading to post-projection stages. It allows for the embedding of non-linear and anisotropic interpolation functions into the footprint tables to maintain sharpness while zooming in.

### Post-classification and Shading in Screen Space
Post-classification and Shading in Screen Space is a method proposed by Mueller et al. [Mueller et al. 1999a] for performing classification and shading in screen space, contrasting with typical pre-classification/pre-shading where voxels store RGBα values [S4, unofficial]. The gradients are either projected to screen space using so-called gradient splats, or they are computed in screen space using central differencing [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The central differencing scheme reduces the number of footprint rasterizations by a factor of four compared to using gradient splats, as only voxel densities need to be projected [S2, abstract only, unofficial][S4, unofficial]. However, the per-pixel gradient calculation and shading add overhead compared to pre-shaded splatting [S1, unofficial][S2, abstract only, unofficial].
*   **Memory Complexity:** Requires an image-aligned sheet buffer [S3, unofficial].
*   **Measured Performance:** Rendered at 512 x 512 resolution for the full UNC head dataset [S2, abstract only, unofficial].
*   **Strengths:** Produces crisp edges and well-preserved surface detail, especially in zoomed views [S1, unofficial][S2, abstract only, unofficial]. Gradient splats provide even crisper detail compared to central differencing [S2, abstract only, unofficial].
*   **Limitations:** Cannot be used with axis-aligned sheet buffers [S3, unofficial]. Combining pre-shaded projection with post-classification causes severe artifacts [S2, abstract only, unofficial].
*   **When to use:** When edge sharpness is critical and the hardware supports image-aligned buffers and efficient per-pixel shading (e.g., via lookup tables) [S1, unofficial][S2, abstract only, unofficial].
*   **When not to use:** When performance is paramount and edge blurring is acceptable, or when using axis-aligned sheet buffers [S3, unofficial].

## Background
Splatting Without the Blur, also known as post-shaded splatting, was introduced by Mueller et al. in 1999 [S1, unofficial][S2, abstract only, unofficial][S4, unofficial]. It was developed to address the limitation of traditional splatting methods, which classify and shade voxels before projection, leading to blurry object edges due to the fuzzy nature of the splat footprints [S1, unofficial][S2, abstract only, unofficial]. The method manipulates the rendering pipeline to perform classification and shading after projection, ensuring that volume contributions outside the iso-range do not affect the final image [S1, unofficial][S2, abstract only, unofficial]. It is a key variant in the splatting family, offering a trade-off between image quality (crisp edges) and computational overhead (screen-space gradient calculation) [S1, unofficial][S2, abstract only, unofficial][S3, S4, unofficial].

## Sources
[S1] EWA volume splatting - ResearchGate, https://www.researchgate.net/publication/48546966_EWA_volume_splatting, fetched 2026-10-28 (unofficial)
[S2] Splatting Without The Blur - Stony Brook University, https://www3.cs.stonybrook.edu/~mueller/research/blur/, fetched 2026-10-28 (abstract only, unofficial)
[S3] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
[S4] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
