# Exponential Depth Cueing in Frequency Domain

Also called: Phase-shift depth cueing

## Overview
Exponential depth cueing in the frequency domain is a method to apply constant exponential decay to volume rendering intensities based on distance, implemented within Fourier Volume Rendering (FVR). It solves the problem of efficiently computing non-linear depth weighting without recomputing the 3-D forward transform for every view. It sets itself apart from linear depth cueing in FVR by introducing an imaginary frequency phase shift in the Fourier domain, which allows the use of fewer textures during accumulation compared to linear cueing or no cueing [S1, unofficial].

## How it works
The method computes exponentially depth-cued footprints in two steps: first, by applying a phase shift in the frequency domain using the Fourier Projection Slicing (FPS) theorem, and second, by weighting the resulting textures according to the distance between the projection plane and the basis functions [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volume data decomposed into B-spline scaling functions and wavelets of order $j$.
    *   Cueing vector $\mathbf{L}$, defined by the extinction coefficient magnitude $|\mathbf{k}|$ and viewing angles $(\alpha_l, \beta_l)$.
    *   Volume midpoint $\mathbf{V_M}$.
    *   Basis function centers $\mathbf{P}$ (relative to $\mathbf{L}_{init}$).
*   **Outputs:**
    *   Depth-cued 2-D footprints (textures) ready for accumulation.

### Data Structures
The method operates on the frequency-domain representations of the basis functions.
*   **Scaling Function Transform:** $\tilde{\Phi}^{[j]}(f)$, a complex-valued function of frequency $f$.
*   **Wavelet Transform:** $\tilde{\Psi}^{[j]}(f)$, a complex-valued function of frequency $f$.
*   **Cueing Vector:** $\mathbf{L} = [k_x, k_y, k_z]^T$.

### Parameters
*   $k$: Constant exponential extinction coefficient.
*   $j$: Order of the B-spline scaling functions and wavelets.
*   $\alpha_l, \beta_l$: Viewing angles defining the cueing direction.
*   $f_0$: Imaginary frequency shift parameter.

### Step-by-Step Method

1.  **Calculate the Imaginary Frequency Shift ($f_0$):**
    Compute the shift value based on the extinction coefficient $k$:
    $$ f_0 = \frac{k}{-2\pi i} = \frac{ki}{2\pi} $$
    [S1, unofficial]

2.  **Compute Cued Footprints via Phase Shift:**
    Apply the phase shift to the standard B-spline and wavelet transforms. The Fourier transform of the exponentially depth-cued function is obtained by shifting the spectrum by $f_0$:
    $$ \tilde{H}(f) = H(f - f_0) $$
    [S1, unofficial]

    For B-spline scaling functions of order $j$:
    $$ \tilde{\Phi}^{[j]}(f) = \left( \frac{1 - e^{-i2\pi(f-f_0)}}{i2\pi(f-f_0)} \right)^j $$
    [S1, unofficial]

    For B-spline wavelets of order $j$:
    $$ \tilde{\Psi}^{[j]}(f) = R^{[j]}\left( e^{-i\pi(f-f_0)} \right) \tilde{\Phi}^{[j]}\left( \frac{f}{2} \right) $$
    where the modulation function $R^{[j]}$ is:
    $$ R^{[j]}\left( e^{-i\pi(f-f_0)} \right) = \frac{1}{2} \sum_{n=-\infty}^{\infty} q_{j,n} \cdot \left( e^{-i\pi(f-f_0)} \right)^n $$
    [S1, unofficial]

    *Note:* In 3D, the direction of this shift operation in the spectrum is chosen independently from the viewing direction; it corresponds to the depth-cueing direction in the spatial domain [S1, unofficial].

3.  **Define the Cueing Vector ($\mathbf{L}$):**
    Calculate the cueing vector for the given viewing angles $(\alpha_l, \beta_l)$:
    $$ \mathbf{L}(\alpha_l, \beta_l) = |\mathbf{k}| \begin{bmatrix} -\cos \alpha_l \cos \beta_l \\ \sin \alpha_l \cos \beta_l \\ \sin \beta_l \end{bmatrix} = \begin{bmatrix} k_x \\ k_y \\ k_z \end{bmatrix} $$
    [S1, unofficial]

4.  **Distance Weighting:**
    Weight the footprints during accumulation to ensure correct exponential decay relative to a reference plane (planar light source) perpendicular to $\mathbf{L}$ pointing towards the volume midpoint $\mathbf{V_M}$.
    *   Let $\mathbf{L}_{init}$ be the intersection point of the volume's bounding sphere and its tangent plane.
    *   Let $\mathbf{P}$ be the position of the basis function center $\mathbf{B_M}$ relative to $\mathbf{L}_{init}$.
    *   Compute the distance $dist$:
        $$ dist = \frac{\mathbf{L} \cdot \mathbf{P}}{|\mathbf{L}|} $$
        [S1, unofficial]
    *   Compute the weighting factor $\omega_d$:
        $$ \omega_d = e^{-dist} $$
        [S1, unofficial]
    *   Multiply the wavelet coefficient by $\omega_d$ during the accumulation step.

### Edge Cases and Pitfalls
*   **Shift Direction:** The direction of the frequency shift is independent of the viewing direction. It must be aligned with the desired depth-cueing direction in the spatial domain [S1, unofficial].
*   **Missing Weighting:** Without the distance weighting step ($\omega_d$), splats are not correctly depth-cued, resulting in incorrect visual output [S1, unofficial].

## Subtopics

### Distance Weighting for Depth Cueing
A specific weighting step applied during accumulation to ensure correct exponential depth cueing. It weights footprints based on their distance from a reference plane perpendicular to the depth-cueing vector $\mathbf{L}$ pointing towards the volume midpoint $\mathbf{V_M}$. It is chosen when correct exponential decay is required for the volume. Without it, smooth and correctly depth-cued volumes are not produced [S1, unofficial].

## Operating characteristics
*   **Performance:** Exponential depth cueing increases rendering performance because fewer textures have to be accumulated compared to linear cueing or no cueing [S1, unofficial].
*   **Visual Quality:** Interactive manipulation of the cueing direction allows for simulating simple shading operations in the wavelet domain, enhancing visual quality significantly [S1, unofficial].
*   **Limitations:** Requires the additional complexity of calculating the imaginary frequency shift $f_0$ and the distance weighting factor $\omega_d$ for each basis function during accumulation.

## Background
This method was proposed to extend previous work on linear depth cueing in the frequency domain [22]. It was introduced in the context of compression domain volume rendering for distributed applications, specifically targeting networked environments where efficient transmission and rendering of volume data (such as Visible Human CT data sets) are required [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering for Distributed ..., https://cgl.ethz.ch/Downloads/Publications/Tutorials/1998/tu_Gro98-4.pdf, fetched 2026-10-26 (unofficial)
