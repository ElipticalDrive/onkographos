# L2-optimal wavelet splatting

Also called: FPS splatting, High-Quality Wavelet Splatting, Volume wavelet rendering, Wavelet Projections for Volume Rendering, Wavelet Splats, Wavelet Splatting, Wavelet-based splatting

## Overview
L2-optimal wavelet splatting is a volume rendering technique that projects volume data by accumulating scaled and translated wavelet basis functions (splats) onto a multiresolution grid [S1, unofficial][S2][S3, unofficial]. It solves the problem of rendering large volume datasets efficiently by performing computations in the wavelet domain without full decompression, while ensuring maximal image quality for a given resolution [S3, unofficial]. It is distinguished from standard wavelet splatting (Method 1) by using least-squares approximation on an adaptive grid, which provides a faster decay in computational complexity with respect to scale and higher quality at coarse resolutions [S2][S3, unofficial].

## How it works
**Inputs:** Volume data $f(\mathbf{x})$ of size $M^3$ [S2], viewing direction $\mathbf{s}$ (spherical coordinates $\alpha, \beta$) [S1, unofficial][S2], rendering screen size $N^2$ [S2], wavelet decomposition level $M$ and scale level $j$ [S1, unofficial][S2], B-spline degree $n$ (typically $n=3$) [S2], base sampling step $h$ (typically $h=1$) [S2].
**Outputs:** Accumulated framebuffer image [S1, unofficial][S2].
**Data Structures:**
- **Wavelet Coefficients:** $d_{Mpqr}^{\text{type}}$ for level $m$, position $p,q,r$, and type; $c_{Mpqr}$ for scaling coefficients at level $M$ [S1, unofficial].
- **Precomputed Kernel Table:** $\xi_{h;\theta}(\mathbf{s})$, storing scalar products between wavelet splats and dual function shifts for all possible shifts, calculated once per view direction [S2].
- **Splat Textures:** 2D arrays representing the inverse FFT of Fourier slices [S1, unofficial].

**Steps:**
1. Decompose volume data $f(\mathbf{x})$ into a linear combination of basis functions $\psi_k(\mathbf{x})$ with coefficients $c(k)$ [S2].
2. Compute wavelet splats $\chi_{\theta;j}(\mathbf{y})$ using Fourier Projection Slicing (FPS). The 2D Fourier transform of the projection equals a plane slicing the 3D Fourier transform parallel to the projection plane and intersecting the origin [S1, unofficial]. For B-splines, the Fourier transform has a closed form [S1, unofficial]. Apply inverse FFT to the 2D slices to obtain spatial splats [S1, S3, unofficial].
3. Approximate splats on a multiresolution grid. For scale $j$, use sampling step $h_j = 2^j h$ [S2]. Compute grid coefficients $c_{h_j}(\mathbf{l})$ via least-squares approximation using the dual function $\tilde{\phi}(\mathbf{y})$ [S2][S3, unofficial]. Precompute the kernel $\xi_{h;\theta}(\mathbf{s})$ by convolving the splat with grid basis shifts [S3, unofficial]. Retrieve coefficients via single table lookup [S2].
4. Accumulate into framebuffer. Map screen positions, weight by coefficients, and accumulate [S1, unofficial].
5. Interpolate to full resolution. Change from dual space to spline space, upsample to $N^2$, then change to cardinal space [S2][S3, unofficial].

**Formulas:**
$$ I(\infty, \mathbf{x}, \mathbf{s}) = \sum_m \sum_{\substack{m = 1 \\ \text{type} = 1}} d_{Mpqr}^{\text{type}} \int_{-\infty}^{\infty} \psi_{Mpqr}^{3, \text{type}}(\mathbf{x} + \mathbf{s}t) dt + \sum_{\substack{p, q, r \in \mathbb{Z}}} c_{Mpqr} \int_{-\infty}^{\infty} \phi_{Mpqr}^3(\mathbf{x} + \mathbf{s}t) dt $$
where $d_{Mpqr}^{\text{type}}$ are wavelet coefficients at level $m$, position $p,q,r$, and type; $c_{Mpqr}$ are scaling coefficients at level $M$; $\mathbf{x}$ is screen position; $\mathbf{s}$ is view direction; $t$ is integration parameter [S1, unofficial].
Grid coefficients:
$$ c_{h_j}(\mathbf{l}) = \langle \chi_{\theta;j}(\mathbf{y}), \tilde{\phi}(\mathbf{y}/h_j \mathbf{l}) \rangle h_j $$
where $\chi_{\theta;j}(\mathbf{y})$ is the continuous wavelet splat at direction $\theta$, scale $j$; $\tilde{\phi}$ is the dual function; $h_j = 2^j h$ is the adaptive sampling step; $\mathbf{l}$ is the grid index [S2].
B-spline Fourier transform:
$$ \hat{\beta}_n(\omega) = \left( \frac{\sin(\omega/2)}{\omega/2} \right)^{n+1} $$
where $n$ is the B-spline degree [S2].

**Parameters:**
- $M$: Volume resolution dimension (e.g., 32, 64, 128, 256, 512, 1024, 2048) [S2].
- $N$: Rendering screen size dimension [S2].
- $h$: Base sampling step size (typically $h=1$, no oversampling) [S2].
- $h_j$: Adaptive sampling step at scale $j$, $h_j = 2^j h$ [S2].
- $n$: B-spline degree ($n=0$ Haar, $n=1$ Linear, $n=3$ Cubic) [S2].

**Edge Cases and Pitfalls:**
- **Semiorthogonality:** For B-spline wavelets of order $j > 1$, the function system is semiorthogonal, requiring dual functions for decomposition. Implementations may require additional basis transforms [S1, unofficial].
- **Full Resolution Performance:** At full resolution ($j=0$ or small $j$), the speed-up breaks down because $h_j$ becomes small, increasing computation, though quality remains higher [S2].
- **Approximation Error:** The choice of $h$ and $n$ affects error. Cubic B-splines ($n=3$) perform significantly better than linear or Haar [S2].
- **Memory Complexity:** Requires storage for wavelet coefficients and the precomputed kernel table $\xi_{h;\theta}(\mathbf{s})$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def l2_optimal_wavelet_splatting(volume_data, view_direction, screen_size, M, N, h=1, n=3):
    # 1. Decompose volume into wavelet coefficients c(k)
    coeffs = wavelet_transform_3d(volume_data, levels=M)
    framebuffer = numpy.zeros(screen_size)

    # 2. Precompute wavelet splats in frequency domain (only 8 unique depths)
    # Basis functions differ by dilation; others derived by subsampling [S1, unofficial]
    unique_splats = {}
    for level in range(M):
        for wtype in ['x', 'y', 'z', 'xy', 'xz', 'yz', 'xyz', 'scaling']:
            basis_ft = tensor_product_fourier_basis(wtype, n, level)
            splat_2d_ft = extract_fourier_slice(basis_ft, view_direction)
            unique_splats[(level, wtype)] = inverse_fft2(splat_2d_ft)

    # 3. For each scale j, compute grid coefficients via L2-optimal projection
    for j in range(M):
        h_j = 2**j * h  # Adaptive sampling step [S2]
        kernel_table = precompute_kernel_table(unique_splats, j, view_direction, h_j, n)

        for (p, q, r), coeff in get_coefficients_at_scale(coeffs, j):
            shift = (p * h_j, q * h_j, r * h_j)
            grid_coeffs = lookup_table(kernel_table, shift)
            splat_image = interpolate_dual_to_cardinal(grid_coeffs, h_j, N)
            framebuffer += coeff * splat_image

    return framebuffer

# Standard math/vector operations assumed available:
# wavelet_transform_3d, tensor_product_fourier_basis, extract_fourier_slice,
# inverse_fft2, precompute_kernel_table, get_coefficients_at_scale,
# lookup_table, interpolate_dual_to_cardinal.
```

## Operating characteristics
- **Time Complexity:** The complexity of Method 2 (L2-optimal) decays with a factor of $8^j$ per scale $j$ at small scales, until it reaches a constant dependent on screen size $N^2$ [S2]. The complexity is approximately $Op2 \approx k_3 2^{3j} M^3 + k_7 N^2$ [S2]. In contrast, Method 1 (standard wavelet splatting) decays with only $2^j$ [S2].
- **Memory Complexity:** Requires storage for wavelet coefficients and the precomputed kernel table $\xi_{h;\theta}(\mathbf{s})$ [S2].
- **Performance:** Method 2 is up to two orders of magnitude faster than Method 1 at scale $j \approx 4-5$ for large volumes ($M=512, 1024, 2048$) [S2]. At full resolution, Method 2 complexity is 50% higher than Method 1, but yields higher quality [S2].
- **Strengths:** Higher quality at coarse resolutions, full control over approximation error, efficient interactive browsing [S2][S3, unofficial].
- **Limitations:** Higher complexity at full resolution compared to Method 1, requires precomputation of kernels and splats [S2].
- **When to use:** Interactive volume rendering where fast feedback at low resolution is needed, and high quality is required once a viewpoint is chosen [S2][S3, unofficial]. Suitable for medical imaging where physicians need high visualization quality [S3, unofficial].

## Background
L2-optimal wavelet splatting was developed by Stefan Horbelt, Michael Unser, and Martin Vetterli at the Swiss Federal Institute of Technology (EPFL) [S2][S3, unofficial]. It extends Gross's method of volume wavelet rendering by computing splats via an orthogonal projection operator using least-squares approximation [S2][S3, unofficial]. It is used in wavelet-based volume rendering, specifically compared against Method 1 in performance analysis [S2].

## Sources
[S1] Compression Domain Volume Rendering for Distributed ..., https://cgl.ethz.ch/Downloads/Publications/Tutorials/1998/tu_Gro98-4.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Wavelet Projections for Volume Rendering, https://bigwww.epfl.ch/publications/horbelt9901.pdf, fetched 2026-10-28
[S3] High-Quality Wavelet Splatting for Volume Rendering, https://bigwww.epfl.ch/publications/horbelt9801.pdf, fetched 2026-10-28 (unofficial)
