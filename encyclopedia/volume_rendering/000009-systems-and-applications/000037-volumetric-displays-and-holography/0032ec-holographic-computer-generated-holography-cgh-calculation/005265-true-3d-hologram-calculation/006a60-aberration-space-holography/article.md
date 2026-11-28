# Aberration-Space Holography

Also called: Aberration-space correction, Full-volume aberration-space holography

## Overview
Aberration-space holography is a method for generating computer-generated holograms (CGH) that simultaneously corrects spatially varying (anisoplanatic) aberrations across an entire Nyquist-limited volume or field of view [S1, unofficial]. Unlike isoplanatic correction, which optimizes phase for a single local patch and fails outside it, this technique combines individual propagation kernels—each tailored to negate local aberration at a specific target point—into a single spatial light modulator (SLM) hologram [S1, unofficial]. This enables diffraction-limited control of multiple optical beams in parallel across 2D fields and 3D volumes, overcoming the anisoplanatic limit that restricts parallel steering to small regions [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $N$: The set of $N$ target points of interest in the farfield (or nearfield, depending on propagation model), defined by their coordinates (e.g., lateral angles $k_x, k_y$ and depth $z$).
    *   $F_n$: The desired complex amplitude (magnitude and phase) for each target spot $n$.
    *   $D$: The order of the Zernike polynomial basis used to model aberrations.
    *   $w_d^{(n)}$: The Zernike coefficients representing the local wavefront aberration at target point $n$.
    *   $P(x,y)$: The pupil plane of the optical system (typically circular).
*   **Outputs:**
    *   $f(x,y)$: The complex nearfield pattern (SLM phase mask) to be displayed.

**Data Structures**
*   **Zernike Polynomials:** Indexed by a singleton ANSI index $d$, corresponding to radial order $n$ and azimuthal order $l$ ($Z_d = Z_{nl}$). The basis set includes terms for lateral steering ($Z_1, Z_2$), defocus ($Z_4$), and higher-order aberrations up to order $D$.
*   **Kernel Representation:** Each kernel $K_n(x,y)$ is represented as a complex exponential of a phase sum:
    $$ K_n(x, y) = \exp\left(-i \left[ k_x^{(n)} x + k_y^{(n)} y + \sum_{d=1}^{D} w_d^{(n)} Z_d(x, y) \right] \right) $$
    where the linear terms $k_x^{(n)} x + k_y^{(n)} y$ handle lateral steering (shearing), and the Zernike sum handles aberration correction and depth focusing.

**Numbered Steps**
1.  **Define the Shearing Kernel:** Establish the base linear steering kernel for lateral displacement in Fourier holography:
    $$ K_{\text{shear}}(k_x, k_y, x, y) = \exp(-i[k_x x + k_y y]) $$
    This kernel steers the nearfield pattern $f(x,y)$ to the farfield angle $(k_x, k_y)$ [S1, unofficial].
2.  **Expand Pupil Wavefront:** Represent the local wavefront aberration at each target point $n$ as a summation of Zernike polynomials $Z_d(x,y)$ weighted by coefficients $w_d^{(n)}$:
    $$ \Phi_n(x, y) = \sum_{d=1}^{D} w_d^{(n)} Z_d(x, y) $$
    The coefficients $w_d^{(n)}$ are determined by measuring or estimating the anisoplanatic aberration at the location of target $n$ [S1, unofficial].
3.  **Tailor Kernels:** Construct a specific "Zernike kernel" $K_n(x,y)$ for each target point $n$ by combining the shearing term with the aberration correction term. The kernel is designed to negate the local aberration, effectively creating a diffraction-limited focus at that specific site despite the distorted medium:
    $$ K_n(x, y) = \exp\left(-i \left[ k_x^{(n)} x + k_y^{(n)} y + \sum_{d=1}^{D} w_d^{(n)} Z_d(x, y) \right] \right) $$
    Note: The sign of the Zernike sum is negative in the exponent to provide subtractive (corrective) phase [S1, unofficial].
4.  **Combine Kernels:** Sum the complex conjugates of all $N$ tailored kernels, weighted by their respective target complex amplitudes $F_n$, to form the hologram:
    $$ f(x, y) \propto \sum_{n=1}^{N} K_n^*(x, y) \cdot F_n $$
    This step reshapes power within a $D$-dimensional aberration space rather than just 2D/3D spatial space, enabling full-field anisoplanatic control [S1, unofficial].
5.  **Compute Nearfield:** Calculate the final SLM phase mask $f(x,y)$ using the compressed sum. This is equivalent to an inverse compressed Fourier transform where the "basis" functions are the aberration-corrected kernels [S1, unofficial].

**Formulas**
*   **Forward Transform (Verification):**
    $$ F_n \propto \sum_{x,y} K_n(x, y) \cdot f(x, y) $$
*   **Inverse Transform (Hologram Generation):**
    $$ f(x, y) \propto \sum_{n=1}^{N} K_n^*(x, y) \cdot F_n $$
    Where $K_n^*$ is the complex conjugate of the $n$-th kernel [S1, unofficial].

**Parameters**
*   **$N$ (Number of spots):** Typically small relative to the megapixel-scale SLM resolution. This sparsity allows for compressive sensing approaches where only a subset of farfield pixels are targeted [S1, unofficial].
*   **$D$ (Zernike Order):** The number of Zernike modes used for correction.
    *   Typical value for 2D arrays: $D=44$ [S1, unofficial].
    *   Typical value for volumetric displays: $D=27$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Anisoplanatism:** Previous techniques fail outside a small isoplanatic patch because the point spread function is not invariant. Aberration-space holography addresses this by treating aberration as a dimension in the hologram calculation [S1, unofficial].
*   **Precision:** The method requires accurate characterization of the local wavefront error $w_d^{(n)}$ for each target point. Errors in aberration measurement will lead to residual aberration in the corrected spots [S1, unofficial].
*   **Computational Load:** While compressive sensing reduces the number of terms compared to a full DFT, summing $N$ kernels with $D$ modes each can be computationally intensive for large $N$ and $D$. However, it avoids the memory overhead of zero-padding required for high-precision DFT-based steering [S1, unofficial].

## Subtopics

### Isoplanatic Correction
Optimizes phase for a single local neighborhood (isoplanatic patch). Leaves residual aberration outside this patch. Choose when the field of view is small or aberrations are uniform. [Link](15055-holographic-defocus-blur-matching.md) (Sibling: Holographic Defocus Blur Matching).

### Metasurface Holography
Uses longitudinally engineered metasurfaces for 3D vectorial holography. Different physical implementation (static metasurface vs. dynamic SLM). [Link](18693-longitudinally-engineered-metasurface-for-3d-vectorial-holog.md) (Sibling: Longitudinally Engineered Metasurface for 3D Vectorial Holography).

## Operating characteristics
*   **Time Complexity:** Linear in the number of target spots $N$ and the number of Zernike modes $D$. The computation is $O(N \cdot D)$ per pixel if evaluated naively, or optimized via FFT-based convolution if the kernels share structure. The sources note that the method offers "fully parallelized performance approaching that of a single Dth order correction" [S1, unofficial].
*   **Memory Complexity:** Proportional to the SLM resolution (megapixel scale) and the storage of $N$ kernels. Compressive sensing reduces the effective number of farfield pixels computed, lowering memory compared to full DFT methods [S1, unofficial].
*   **Measured Performance:**
    *   **2D Tweezer Array:** Corrected 50 isoplanatic patches with 8 principal aberration modes. Achieved an 8× larger field of view than the best isoplanatic correction. Uncorrected holography yielded average relative Strehl ratio $\langle \tilde{S} \rangle = 0.15$. Isoplanatic correction improved local performance but only ~2% of the field achieved diffraction-limited performance ($\tilde{S} > 0.8$). Aberration-space holography achieved $\langle \tilde{S} \rangle = 0.78$, with 43% of the field corrected to $\tilde{S} > 0.8$ [S1, unofficial].
    *   **3D Volumetric Display:** Increased the corrected volume by 12× compared to isoplanatic correction in two-photon excitation experiments [S1, unofficial].
*   **Strengths:** Enables full-field, parallel beamforming; corrects anisoplanatic aberrations; works across 2D fields and 3D volumes; compatible with iterative algorithms like Gerchberg-Saxton for amplitude optimization [S1, unofficial].
*   **Limitations:** Requires accurate wavefront sensing/calibration for each target point; performance is limited by the order $D$ of the Zernike basis (higher order requires more modes); SLM temporal response limits speed of serial correction if not fully parallelized [S1, unofficial].
*   **When to Use:** When parallel steering of multiple beams is required across a large field of view or deep volume in the presence of significant spatially varying aberrations (e.g., through tissue, vacuum windows, or turbulent media).
*   **When Not to Use:** When the field of view is small enough that a single isoplanatic patch covers the entire area, or when real-time correction is needed but wavefront sensing is too slow [S1, unofficial].

## Background
Aberration-space holography was introduced to overcome the limitations of isoplanatic correction in adaptive optics and holography. Traditional adaptive optics often corrects for a single "isoplanatic patch," leaving residual aberrations elsewhere. This technique generalizes the concept by treating aberration as an additional dimension in the hologram calculation, allowing for simultaneous correction across the entire Nyquist-limited volume [S1, unofficial]. It builds on the principles of Fourier holography and compressive sensing spot holography, extending the shearing kernel to include modal expansions of the pupil wavefront (Zernike polynomials) [S1, unofficial]. The method has applications in neutral atom quantum computing, holographic displays, high-speed 3D laser printing, multifocal nonlinear microscopy, optogenetics, and laser fabrication [S1, unofficial].

## Sources
[S1] Full-volume aberration-space holography, https://arxiv.org/abs/2505.08777v1, fetched 2026-10-26 (unofficial)
