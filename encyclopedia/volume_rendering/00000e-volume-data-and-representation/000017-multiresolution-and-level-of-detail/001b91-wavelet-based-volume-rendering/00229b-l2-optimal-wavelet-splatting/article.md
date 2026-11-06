# L2-optimal wavelet splatting

Also called: Wavelet Projections for Volume Rendering

## Overview
L2-optimal wavelet splatting is a volume rendering technique that decomposes volume data into a wavelet pyramid representation in the spline domain and computes splats via an orthogonal projection operator using least-squares approximation on an adaptive multiresolution grid [S1]. It sets itself apart from standard interpolation-based wavelet splatting (such as [Wavelet-based Multiresolution Rendering](1588-wavelet-based-multiresolution-rendering.md)) by ensuring the smallest possible approximation error for a given sampling step size and by adapting the grid resolution to the basis function size, which yields significant performance gains at coarse scales [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data $f(\mathbf{x})$ of size $M^3$ voxels, represented as a linear combination of basis functions $\psi_k(\mathbf{x})$ with coefficients $c(k)$: $f(\mathbf{x}) = \sum_k c(k)\psi_k(\mathbf{x})$ [S1].
*   **Input:** Viewing direction $\theta$ (unit vector $\vec{\theta}$) and screen dimensions $N \times N$ [S1].
*   **Input:** Scale level $j$ and base sampling step $h$ [S1].
*   **Output:** Rendered image on the screen of size $N^2$ pixels [S1].

**Data Structures:**
*   **Wavelet Coefficients:** Array of coefficients $c(k)$ corresponding to the wavelet decomposition of the volume [S1].
*   **Kernel Table:** A precomputed table for the kernel function $\xi_{h;\theta}(\mathbf{s})$, calculated once per viewing direction at fine resolution in shift parameter $\mathbf{s}$ [S1].
*   **Grid:** An adaptive reconstruction grid with sampling step $h_j = 2^j h$ [S1].

**Parameters:**
*   $M$: Volume resolution dimension (e.g., 32, 64, 128, 256, 512, 1024, 2048) [S1].
*   $N$: Rendering screen size dimension [S1].
*   $h$: Base sampling step size. Typical value: $h=1$ (no oversampling) [S1].
*   $h_j$: Adaptive sampling step size at scale $j$, defined as $h_j = 2^j h$ [S1].
*   $j$: Scale level. Complexity decays with a factor of $8^j$ per scale $j$ at small scales [S1].
*   $n$: Degree of the B-spline basis function. Values used: $n=0$ (Haar), $n=1$ (Linear), $n=3$ (Cubic) [S1].
*   $k_1, k_3, k_7$: Constant factors in the operation count formulas [S1].
*   $s$: Splat support size, defined as $s(\Psi_n) = 2n+1$ for B-splines [S1].

**Steps:**

1.  **Wavelet Decomposition:** Decompose the volume data $f(\mathbf{x})$ into a linear combination of basis functions $\psi_k(\mathbf{x})$ with coefficients $c(k)$ [S1].
    $$ f(\mathbf{x}) = \sum_k c(k)\psi_k(\mathbf{x}) $$

2.  **Wavelet Splat Definition:** The volume rendering integral (X-ray transform) for a direction $\theta$ becomes a linear combination of wavelet splats $\chi_{\theta;k}(\mathbf{y})$:
    $$ p_\theta f(\mathbf{y}) = \sum_k c(k) \chi_{\theta;k}(\mathbf{y}) $$
    where $\mathbf{y} = (u, v)$ are coordinates on the projection plane perpendicular to $\vec{\theta}$, and $\chi_{\theta;k}(\mathbf{y}) = p_\theta \psi_k(\mathbf{y})$ is the projection of the basis function [S1].

3.  **Adaptive Grid Approximation:** For a fixed viewing direction $\theta$ and scale $j$, approximate the continuous wavelet splat $\chi_{\theta;j}(\mathbf{y})$ onto a reconstruction grid with sampling step $h_j = 2^j h$ using basis functions $\phi(\mathbf{y}/h_j \mathbf{l})$ [S1]. The approximation operator $P_{h_j}$ yields discrete grid coefficients $c_{h_j}(\mathbf{l})$ [S1].

4.  **Least-Squares Coefficient Calculation:** Calculate the discrete grid coefficients $c_{h_j}(\mathbf{l})$ using least-squares approximation via the scalar product with the dual function $\tilde{\phi}(\mathbf{y})$:
    $$ c_{h_j}(\mathbf{l}) = \langle \chi_{\theta;j}(\mathbf{y}), \tilde{\phi}(\mathbf{y}/h_j \mathbf{l}) \rangle h_j $$
    where $\tilde{\phi}(\mathbf{y})$ is the dual function of the basis function $\phi(\mathbf{y})$ [S1].

5.  **Kernel Lookup:** Express the scalar product as a kernel function $\xi_{h;\theta}(\mathbf{s})$ with a single shift parameter $\mathbf{s} = h_j \mathbf{l} - \mathbf{z}$ (where $\mathbf{z}$ relates to the wavelet shift):
    $$ c_{h_j}(\mathbf{l}) = \xi_{h;\theta}(\mathbf{s}) = \left\langle \tilde{\phi}\left(\frac{\mathbf{y}}{h_j}\right), \chi_{\theta}(\mathbf{y} + \mathbf{s}) \right\rangle $$
    The kernel $\xi_{h;\theta}(\mathbf{s})$ is precomputed for the current viewing direction at fine resolution in $\mathbf{s}$ and stored in a table [S1]. Retrieving grid coefficients requires a single table lookup [S1].

6.  **Full-Screen Interpolation:** Interpolate the projected data from the dual space grid to full-screen resolution $N^2$ in cardinal space using three steps [S1]:
    *   **Dual-to-Spline Change:** Convert from dual space to spline space [S1].
    *   **Upsampling:** Use the shortest upsampling interpolation filters available in spline space to reach full resolution [S1].
    *   **Spline-to-Cardinal Change:** Convert from spline space to cardinal space for visualization [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def l2_optimal_wavelet_splatting(volume_coeffs, viewing_direction, volume_size_M, screen_size_N, scale_j, base_step_h, spline_degree_n):
    # 1. Calculate adaptive grid step
    h_j = (2 ** scale_j) * base_step_h

    # 2. Precompute kernel table for the current viewing direction
    # The kernel xi_h_theta(s) is calculated once at fine resolution in s.
    # [Source does not specify the exact numerical integration method for the kernel.]
    kernel_table = {}

    # 3. Initialize adaptive grid
    grid_dim = int(volume_size_M / base_step_h)
    grid = [[0.0 for _ in range(grid_dim)] for _ in range(grid_dim)]

    # 4. Project wavelet splats onto the adaptive grid
    for k, c_k in enumerate(volume_coeffs):
        # [Source does not specify the exact mapping from coefficient index k to spatial shift z and scale.]
        z = k % volume_size_M
        wavelet_scale = k // volume_size_M

        # Determine relevant grid indices l where the kernel support overlaps
        # s = h_j * l - z
        for l in range(grid_dim):
            s = h_j * l - z
            # [Source does not specify exact kernel support bounds; typically compact for B-splines]
            if abs(s) <= (2 * spline_degree_n + 1):
                # Single table lookup
                kernel_value = kernel_table.get((s, viewing_direction), 0.0)
                grid[l] += c_k * kernel_value

    # 5. Interpolate from adaptive grid to full screen resolution N^2
    # Step 1: Dual-to-spline space change
    spline_grid = [[0.0 for _ in range(grid_dim)] for _ in range(grid_dim)]
    for i in range(grid_dim):
        for j in range(grid_dim):
            spline_grid[i][j] = grid[i][j] # [Source does not specify exact dual-to-spline filter coefficients]

    # Step 2: Upsampling to full resolution using shortest filters in spline space
    upscaled_image = [[0.0 for _ in range(screen_size_N)] for _ in range(screen_size_N)]
    for i in range(screen_size_N):
        for j in range(screen_size_N):
            upscaled_image[i][j] = spline_grid[i // 2][j // 2] # [Source does not specify exact upsampling filter]

    # Step 3: Spline-to-cardinal space change
    rendered_image = [[0.0 for _ in range(screen_size_N)] for _ in range(screen_size_N)]
    for i in range(screen_size_N):
        for j in range(screen_size_N):
            rendered_image[i][j] = upscaled_image[i][j] # [Source does not specify exact spline-to-cardinal filter]

    return rendered_image
```

**Edge Cases and Pitfalls:**
*   **Full Resolution Breakdown:** At full resolution ($j=0$ or small $j$), the speed-up of Method 2 breaks down because the adaptive grid step $h_j$ becomes small, increasing computation [S1]. At this point, the complexity of Method 2 is 50% higher than Method 1, although it yields higher quality [S1].
*   **Precision:** The method relies on accurate precomputation of the kernel table [S1]. Errors in the kernel calculation can propagate to the final image [G].
*   **Degenerate Input:** The method assumes the volume data can be decomposed into the specified wavelet basis [G]. If the volume has significant high-frequency content not captured by the basis, the approximation error may be higher [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Method 1 (Gross):** Complexity $Op_1 \approx k_1 (M 2^j)^3 (s + h - 1)^2 \approx 2^j M^3$ [S1]. The decay is only $2^j$ per scale [S1].
*   **Method 2 (L2-optimal):** Complexity $Op_2 \approx k_3 2^{-3j} M^3 + k_7 N^2$ [S1]. The term dependent on $M$ decays as $2^{-3j}$ (or $8^{-j}$) per scale [S1].
*   **Scaling:** The complexity $Op_2$ decays with a factor of $8^j$ per scale $j$ at small scales, until it reaches a constant dependent on screen size $N^2$ [S1]. This is the theoretical limit as complexity cannot decrease faster than the number of voxels decreases per scale $j$ [S1].

**Measured Performance:**
*   Method 2 is up to two orders of magnitude faster than Method 1 at scale $j$ around 4-5 for large volumes ($M=512, 1024, 2048$) [S1].
*   At full resolution, Method 2 complexity is 50% higher than Method 1, but yields higher quality [S1].

**Strengths and Limitations:**
*   **Strengths:** Higher performance at coarse resolutions; better quality for a given resolution; full control over approximation error; adaptive grid reduces computation at coarse scales [S1].
*   **Limitations:** Slower at full resolution compared to direct splatting; requires precomputation of kernel table per view; more complex implementation due to dual space and interpolation steps [S1].

**When to Use:**
*   Interactive volume rendering where fast feedback is needed while browsing around and through a volume [S1].
*   Progressive rendering where resolution increases once a viewpoint is chosen [S1].
*   Large volumes where coarse-scale rendering is dominant [S1].

**When Not to Use:**
*   When full resolution rendering is the primary requirement, as Method 2 is slower than Method 1 at $j=0$ [S1].
*   When hardware does not support efficient table lookups or adaptive grid operations [G].

## Background
L2-optimal wavelet splatting was proposed by Stefan Horbelt, Michael Unser, and Martin Vetterli from the Swiss Federal Institute of Technology (EPFL) [S1]. It extends Gross's method of volume wavelet rendering by computing splats via an orthogonal projection operator [S1]. The method was published in the Eurographics Association proceedings [S1]. It builds on the theory of B-spline signal processing and wavelet approximation techniques [S1].

## Sources
[S1] Wavelet Projections for Volume Rendering, https://bigwww.epfl.ch/publications/horbelt9901.pdf, fetched 2026-10-28
