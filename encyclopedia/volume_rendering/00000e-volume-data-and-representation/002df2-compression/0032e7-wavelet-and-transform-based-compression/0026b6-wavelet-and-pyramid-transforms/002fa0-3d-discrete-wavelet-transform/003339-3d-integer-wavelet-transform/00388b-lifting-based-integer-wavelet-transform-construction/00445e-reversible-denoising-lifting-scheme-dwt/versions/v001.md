# Reversible Denoising Lifting Scheme DWT

Also called: RDLS-DWT, Reversible Denoising and Lifting Steps, Reversible Denoising and Lifting Steps (RDLS) with Step Skipping

## Overview
Reversible Denoising Lifting Scheme DWT (RDLS-DWT) is a wavelet transform variant that replaces standard prediction and update lifting steps with denoising-aware steps to prevent noise propagation between subbands while maintaining perfect reversibility [S1]. It sets itself apart from standard Lifting-Based Integer Wavelet Transforms by integrating irreversible denoising filters into the lifting process, allowing the transform to adapt to signal characteristics without losing invertibility [S1].

## How it works
**Inputs and Outputs:**
*   Input: A sequence of samples $s_i$ (typically from a subband or spatial domain).
*   Output: Transformed coefficients (detail and approximation) and the original samples (if reconstructed).

**Data Structures:**
*   Sample array $s$: 1D or multi-dimensional array of integer values.
*   Denoised sample $s_{dx}$: Intermediate value derived from $s$ using a denoising filter.

**Parameters:**
*   Denoising filter: A function mapping input samples to denoised samples $s_{dx}$.
    *   **None Filter:** $s_{dx} = s_i$. Turns RDLS into the unmodified transform [S1].
    *   **Null Filter:** $s_{dx} = 0$. Allows practically skipping the step [S1].
*   Rounding function $b(\cdot)$: Defined as $\lfloor \cdot \rfloor$ (floor) in the source formulas [S1].

**Formulas:**
The standard prediction and update lifting steps are replaced by the following RDLS steps [S1]:

1.  **Prediction Step:**
    $$s_x \leftarrow s_x - \left\lfloor \frac{b(s_{dx-1} + s_{dx+1})}{2} \right\rfloor$$
    Where $s_x$ is the current sample, and $s_{dx-1}, s_{dx+1}$ are the denoised neighbors [S1].

2.  **Update Step:**
    $$s_x \leftarrow s_x + \left\lfloor \frac{b(s_{dx-1} + s_{dx+1} + 2)}{4} \right\rfloor$$
    Where $s_x$ is the current sample, and $s_{dx-1}, s_{dx+1}$ are the denoised neighbors [S1].

**Algorithm Steps:**
1.  For each sample $s_x$ in the input sequence:
2.  Compute the denoised sample $s_{dx}$ using the selected denoising filter applied to the neighborhood of $s_x$ [S1].
3.  Apply the Prediction Step formula to update $s_x$ [S1].
4.  Apply the Update Step formula to update $s_x$ [S1].
5.  If the Null filter is used ($s_{dx}=0$), the operations become identity operations ($s_x \leftarrow s_x$), effectively skipping the computation [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The formulas reference neighbors $s_{dx-1}$ and $s_{dx+1}$. Implementation must handle boundaries (e.g., symmetric extension, zero-padding, or clamping) consistently with the parent DWT implementation [S1].
*   **Reversibility:** Despite using irreversible denoising filters, the transform remains perfectly reversible because the denoising is only used to compute the prediction/update offsets, not to alter the data directly before the lifting steps [S1].
*   **Step Skipping:** In standard DWT, the non-lifting reordering step limits the freedom to skip selected transform parts. RDLS-DWT combined with Step Skipping (RDLS-SS-DWT) removes this limitation by allowing any step, including reordering, to be skipped if the Null filter is used for both complementary subbands [S1].

## Subtopics

### Skippable Step DWT
A transform obtained from DWT by allowing skipping any transform computation step, including the non-lifting reordering step, which limits the freedom in the standard DWT [S1]. The non-lifting reordering step in standard DWT limits the freedom to skip selected transform parts; SS-DWT removes this limitation [S1]. SS-DWT attains similar bitrate improvements to RDLS-SS-DWT but at a smaller computational cost [S1].

### Reversible Denoising Lifting Scheme DWT — RDLS-SS-DWT
A variant of RDLS-DWT that incorporates step skipping [S1]. RDLS-SS-DWT results in greater bitrate improvements compared to RDLS-DWT [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** RDLS-DWT and its variants (RDLS-SS-DWT, SS-DWT) provide lossless compression bitrate improvements for 3D volumetric images (e.g., medical images) when used with JPEG 2000 (JP3D) [S1].
*   **Trade-offs:** RDLS-SS-DWT offers greater bitrate improvements than RDLS-DWT [S1]. SS-DWT offers similar improvements to RDLS-SS-DWT but at a smaller computational cost [S1].
*   **Limitations:** The effectiveness depends on the choice of denoising filter and which subbands are processed [S1].

## Background
The method was proposed in the context of improving the lossless compression of 3D volumetric medical images using the JP3D standard [S1]. It generalizes techniques previously found effective in 2D images to 3D [S1]. The concept of Reversible Denoising Lifting Steps (RDLS) and step skipping was previously applied to reversible color space transforms and 2D-DWT [S1].

## Sources
[S1] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
