# Polynomial Depth Cueing

Also called: Specular Reflections via Polynomial Depth Cueing, Specular Shading, Fourier Projection-Slice with Polynomial Depth Cueing

## Overview
Polynomial Depth Cueing is an extension of the Fourier projection-slice volume rendering method that replaces linear attenuation with non-linear depth cues, such as inverse square law falloff, or models specular reflections using the Phong illumination model [S1, S2, unofficial]. It sets itself apart from standard linear attenuation by allowing steeper intensity falloff to disambiguate depth in high-complexity scenes, but at the cost of requiring a large number of precomputed spectra determined by the multinomial theorem [S1, unofficial].

## How it works
**Inputs:**
*   Volume dataset $V(\mathbf{x}, \mathbf{y}, \mathbf{z})$ containing scalar values (e.g., density, opacity).
*   View direction vector $\mathbf{V}$.
*   Light direction vector $\mathbf{L}$.
*   Surface normal field $\mathbf{N}(\mathbf{x}, \mathbf{y}, \mathbf{z})$ (required for specular reflections; optional or replaced for depth cueing).
*   Specular exponent $n$ (integer $\ge 1$).
*   Source intensity $I_s$ and opacity function $\Omega$.

**Outputs:**
*   Rendered image intensity $I$.

**Data Structures:**
*   **Precomputed Spectra:** A set of $K$ 2D Fourier spectra, where $K$ is the number of terms in the polynomial expansion. For specular reflections, $K$ is determined by the multinomial theorem applied to $(\mathbf{N} \cdot \mathbf{H})^n$. For depth cueing, $K$ is determined by the degree of the polynomial depth function.
*   **Weighting Vectors:** $\mathbf{H}_n$ and $\mathbf{N}_n$ (or $\mathbf{D}_n$) containing the coefficients and vector components derived from the polynomial expansion.

**Steps:**

1.  **Determine the Polynomial Expansion:**
    *   For **Specular Reflections**: Expand the term $(\mathbf{N} \cdot \mathbf{H})^n$ into a polynomial using the multinomial theorem, where $\mathbf{H}$ is the halfway vector between $\mathbf{L}$ and $\mathbf{V}$ ($\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|}$) [S1, unofficial].
    *   For **Polynomial Depth Cueing**: Replace $\mathbf{N}$ with the viewing ray direction $\mathbf{D}$ (or position vector relative to viewer) and $\mathbf{H}$ with $\mathbf{V}$ to model non-linear attenuation, such as inverse square law $(\mathbf{D} \cdot \mathbf{V})^2$ [S1, S2, unofficial].

2.  **Precompute Spectra:**
    *   Compute the 2D Fourier spectra for each term in the expanded polynomial.
    *   The number of spectra $K$ corresponds to the number of terms. For example, if $n=10$ for specular reflections, $K=286$ [S1, unofficial]. If using inverse square law falloff with degree 2, $K=10$ spectra are required [S2, unofficial].

3.  **Apply Fourier Projection-Slice Theorem:**
    *   For each precomputed spectrum, apply the Fourier projection operator $\Pi_V$ along the view direction $\mathbf{V}$ [S1, unofficial].
    *   The output image $I$ is computed as a weighted sum of these projections.

4.  **Compute Final Intensity:**
    *   For **Specular Reflections**:
        $$I = \mathbf{H}_n \cdot \Pi_V\left(\frac{\pi}{2} + \frac{\pi}{2} \mathbf{N}_n(\mathbf{x}, \mathbf{y}, \mathbf{z})\right) I_s \Omega(\mathbf{x}, \mathbf{y}, \mathbf{z}) + c_4 \Pi_V[I_s \Omega(\mathbf{x}, \mathbf{y}, \mathbf{z})]$$
        where $\mathbf{H}_n$ and $\mathbf{N}_n$ contain the factors from the polynomial expansion of $(\mathbf{N} \cdot \mathbf{H})^n$, and $c_4$ is a suitable constant [S1, unofficial].
    *   For **Depth Cueing** (e.g., inverse square law):
        Replace $\mathbf{N}$ and $\mathbf{H}$ with $\mathbf{D}$ and $\mathbf{V}$ in the weighting equations and adjust constants accordingly [S1, S2, unofficial].

**Parameters:**
*   $n$: Specular exponent or polynomial degree. Typical value $n=10$ [S1, unofficial].
*   $I_s$: Source intensity.
*   $\Omega$: Opacity function.

**Pitfalls:**
*   **Performance:** The number of required spectra grows rapidly with $n$. For $n=10$, 286 spectra are needed, making the method significantly slower than conventional volume rendering, even for large volumes [S1, unofficial].
*   **Precision:** High-degree polynomials may suffer from numerical instability if not carefully implemented [G].
*   **Boundary Conditions:** Ensure that vectors $\mathbf{N}, \mathbf{H}, \mathbf{D}, \mathbf{V}$ are properly normalized before dot products to avoid incorrect intensity scaling [G].

## Operating characteristics
*   **Time Complexity:** $O(K \cdot N \log N)$, where $K$ is the number of spectra (dependent on $n$) and $N$ is the number of voxels. The FFT dominates the cost per spectrum.
*   **Memory Complexity:** $O(K \cdot N)$ to store the precomputed spectra and intermediate projections.
*   **Strengths:** Provides realistic specular highlights and steeper depth cues than linear attenuation, improving depth perception in complex scenes [S1, S2, unofficial].
*   **Limitations:** High computational cost due to the large number of required spectra for high-degree polynomials [S1, unofficial].
*   **When to Use:** When visual quality of specular reflections or depth cueing is critical and performance is not the primary constraint.
*   **When Not to Use:** For real-time applications or scenes with limited computational resources, especially if high $n$ values are required.

## Background
Polynomial Depth Cueing was introduced by Marc Levoy in his 1992 paper "Volume Rendering using the Fourier Projection-Slice Theorem" as an extension to handle specular reflections and non-linear depth attenuation [S1, S2, unofficial]. It builds on the Fourier projection-slice method by incorporating the Phong specular model and multinomial expansion [S1, unofficial]. The method was demonstrated on CT scan data of a human skull [S1, S2, unofficial].

## Sources
[S1] Volume Rendering using the Fourier Projection-Slice Theorem, https://www.graphicsinterface.org/wp-content/uploads/gi1992-8.pdf, fetched 2026-10-27 (unofficial)
[S2] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
