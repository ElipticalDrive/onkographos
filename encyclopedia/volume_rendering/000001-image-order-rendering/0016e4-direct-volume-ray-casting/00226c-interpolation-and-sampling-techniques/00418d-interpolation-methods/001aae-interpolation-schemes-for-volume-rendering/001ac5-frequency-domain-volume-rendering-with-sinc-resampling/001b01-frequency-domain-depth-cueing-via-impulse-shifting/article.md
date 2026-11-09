# Frequency Domain Depth Cueing via Impulse Shifting

Also called: Analytic signal depth cueing, Cosine Delta Method, Frequency Domain Depth Cueing, Shifted H depth cueing, Shifted spectrum depth cueing

## Overview
Frequency Domain Depth Cueing via Impulse Shifting is a technique for applying distance-based attenuation (depth cueing) to volume rendering entirely within the frequency domain, avoiding expensive spatial-domain convolution or repeated 3D FFTs per view [S1, S2, unofficial]. It modifies the interpolation filter $H$ to a shifted filter $H'$ by convolving it with the Fourier transform of the depth cueing function, allowing the cueing to be applied as a simple 2D operation on the slice plane during extraction [S1, S2, unofficial]. Unlike naive spatial approaches or hybrid methods that require multiple volume copies and per-view 3D transforms, this method leverages the precomputed 3D spectrum and small filter support to make the depth cueing cost negligible compared to interpolation [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The 3D Fourier transform of the volume data $F(s)$, the original interpolation filter $H(s)$, the view direction vector $V$, and depth cueing parameters (strength, offset, period) [S1, S2, unofficial].
*   **Output:** The modified interpolation filter $H'(s)$ and the final projected image values computed by interpolating $F(s)$ with $H'(s)$ [S2, unofficial].

**Data Structures:**
*   $F(s)$: 3D spectrum of the volume data [S1, S2, unofficial].
*   $H(s)$: 3D interpolation kernel. Support is small, typically $3^3$ to $5^3$ [S1, S2, unofficial].
*   $H'(s)$: 2D complex array on the slice plane, representing the modified filter [S2, unofficial].
*   $D(s)$: 2D complex array representing the Fourier transform of the depth cueing function $d(x)$ evaluated on the slice plane [S2, unofficial].

**Method Steps:**

1.  **Define Depth Cueing Function:**
    Choose a depth cueing function $d(x)$ that weights voxels by their distance from the observer [S1, unofficial].
    *   *Linear Cueing:* $d_l(x) = C_{cue}(V \cdot x) + C_{avg}$, where $V$ is the view vector, $x$ is the position, $C_{cue}$ is the strength, and $C_{avg}$ is a constant offset [S2, S3, unofficial].
    *   *Sine Wave Cueing:* Model $d(x)$ as a half-period sine wave to simplify its transform [S3, unofficial]. Adjust the origin, amplitude, and period such that the value is zero at the farthest voxel and unity at the closest voxel to eliminate the need for a DC term [S1, S3, unofficial]. The period is typically made long enough so the cueing appears almost linear [S1, S3, unofficial].

2.  **Compute Fourier Transform of Cueing Function $D(s)$:**
    *   *For Linear Cueing:* The transform $D_l(s)$ is computed using a precomputed differential operator convolution $VH(s)$ [S2, S3, unofficial].
    *   *For Sine Wave Cueing:* The transform consists of two impulses:
        $$D(s) = C_1 \delta(s - s_w) + C_2 \delta(s + s_w)$$
        where $C_1$ and $C_2$ are complex constants determined by amplitude and shift, and $s_w$ is determined by the period [S1, S3, unofficial].

3.  **Compute Modified Filter $H'(s)$:** Convolve the original filter with the cueing transform:
    $$H'(s) = H(s) * D(s)$$
    *   *Implementation via Impulse Shifting:* Since $D(s)$ consists of impulses (for sine wave cueing), the convolution becomes a sum of shifted copies of $H(s)$ with complex weights:
        $$H'(s) = C_1 H(s - s_w) + C_2 H(s + s_w)$$
        [S1, S3, unofficial].
    *   *Optimization:* To reduce kernel size, one of the impulses can be removed by doubling the weight of the remaining impulse, utilizing the analytic signal property [S1, S3, unofficial].

4.  **Interpolate and Project:**
    Use $H'(s)$ instead of $H(s)$ to interpolate $F(s)$ during slice extraction [S2, unofficial]. This operation is 2D, evaluated only on the plane of the slice being extracted [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shifted_filter(H, s_w, C1, C2, remove_negative=False):
    """
    Computes the modified interpolation filter H' for depth cueing.
    Args:
        H: 3D complex array, the original interpolation filter on a grid.
        s_w: 3D float array, frequency shift vector.
        C1, C2: complex numbers, impulse weights.
        remove_negative: bool, if True, removes the negative frequency impulse.
    Returns:
        H_prime: 2D complex array on the slice plane, the modified filter.
    """
    # [G] Standard trilinear interpolation of 3D grid H at shifted coordinates (s_w and -s_w),
    # extracting the 2D slice plane. Returns 2D complex arrays.
    H_pos = interpolate_3d_to_2d(H, s_w)
    H_neg = interpolate_3d_to_2d(H, -s_w)
    if remove_negative:
        return 2.0 * C1 * H_pos
    return C1 * H_pos + C2 * H_neg

def interpolate_3d_to_2d(grid_3d, shift_vec):
    # [G] Standard trilinear interpolation of 3D grid at shifted coordinates, extracting 2D slice.
    # For each 2D coordinate on the slice plane, compute the weighted sum of the 8 neighboring 3D grid values
    # based on fractional distances, handling boundaries by zero-padding. Returns 2D complex array.
    # Sources do not specify the exact interpolation routine.

def render_slice(F, H_prime, slice_plane):
    """
    Renders a slice using the modified filter.
    Args:
        F: 3D complex array, volume spectrum.
        H_prime: 2D complex array, modified filter.
        slice_plane: 2D array of coordinates defining the extraction plane.
    Returns:
        projection: 2D complex array, the projected slice.
    """
    # [G] Standard slice extraction: samples F at the given 2D plane coordinates using trilinear interpolation.
    F_slice = extract_slice(F, slice_plane)
    # [G] Standard 2D convolution: for each output pixel, compute the weighted sum of neighboring input pixels using kernel H_prime.
    projection = convolve_2d(F_slice, H_prime)
    return projection.real

def extract_slice(F, plane_coords):
    # [G] Standard slice extraction: samples F at the given 2D plane coordinates.
    # For each 2D coordinate, compute trilinear interpolation from the 3D spectrum F. Returns 2D complex array.
    # Sources do not specify the exact interpolation routine.

def convolve_2d(A, B):
    # [G] Standard 2D convolution of array A with kernel B.
    # For each output pixel, compute the weighted sum of neighboring input pixels using kernel B, handling boundaries by zero-padding.
    # Sources do not specify the exact convolution routine.
```

**Edge Cases and Pitfalls:**
*   **Complex Output:** Removing one impulse (to reduce kernel size) or using a sine wave with two impulses makes the projection image complex (non-real) [S1, S3, unofficial]. The imaginary part is a cosine wave (analytic signal artifact) [S3, unofficial]. However, the real part of the result still contains the correct projection image [S1, S3, unofficial]. Only the real part should be displayed.
*   **Kernel Size:** Using a sine wave with two impulses increases the filter kernel size [S3, unofficial]. Removing one impulse mitigates this but introduces complex output [S1, S3, unofficial].
*   **Precomputation:** $VH$ (differential operator convolution) can be precomputed and stored in a table, making the computation of $H'$ for linear cueing of insignificant cost [S2, S3, unofficial].
*   **DC Term:** Adjusting the sine wave parameters (origin, amplitude, period) to be zero at the farthest voxel and unity at the closest eliminates the DC term, simplifying $D(s)$ to just two impulses [S1, S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The 3D forward FFT is performed only once [S1, S2, unofficial]. For each view, the modification of $H$ to $H'$ is a 2D operation on the slice plane. The cost of recomputing $H'$ is negligible compared to the cost of interpolation itself, because the support of $H$ is small ($3^3$ to $5^3$) and $D(s)$ is simple [S1, S2, unofficial].
*   **Memory Complexity:** Requires storage for the 3D spectrum $F(s)$ and the filter $H(s)$. No additional volume copies are needed [S1, S2, unofficial].
*   **Performance:** The depth cueing is "practically free" compared to spatial domain convolution [S1, S2, unofficial]. While asymptotically faster than conventional spatial domain volume rendering, implementation considerations make it barely superior in practice [S2, unofficial].
*   **Strengths:** Avoids per-view 3D FFTs; avoids multiple volume copies; high quality due to sinc resampling (inherited from parent method); flexible depth cueing functions [S1, S2, unofficial].
*   **Limitations:** Produces complex output (requires taking real part); kernel size can increase with complex cueing functions; requires precomputation of differential operators for linear cueing [S1, S2, S3, unofficial].

## Background
*   **Origin:** Developed as part of the frequency domain volume rendering work in 1993 [S2, S3, S4, unofficial].
*   **Development:** It extends the frequency domain volume rendering algorithm by incorporating depth cueing and directional shading entirely in the frequency domain, using properties of the Fourier transform (differentiation theorem) to avoid spatial domain multiplication [S1, S2, unofficial].
*   **Usage:** Used in fast volume rendering applications where depth cues are important for shape perception [S2, unofficial].

## Sources
[S1] PDF Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Frequency Domain Volume Rendering - Stanford University, https://graphics.stanford.edu/papers/fvr/totsuka-fvr-sig93.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S4] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
