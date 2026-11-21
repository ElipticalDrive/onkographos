# Composite-Only Splatting
Also called: Basic Splatting

## Overview
Composite-only splatting is the foundational object-order volume rendering technique where volumetric primitives are traversed in depth order, projected onto the screen, and immediately composited into the final image buffers [S1, unofficial][S2, book]. It differs from sibling methods like [Stop-the-pop](1472-stop-the-pop.md) or [Sorting Algorithms](14243-sorting-algorithms.md) by performing per-primitive compositing rather than accumulating splats into intermediate buffers or sorting all splats globally before a single pass, making it the simplest but most artifact-prone approach [S1, unofficial][S2, book].

## How it works
**Inputs:**
*   Volumetric dataset $V$ with scalar values.
*   Transfer functions mapping scalar values to color $\mathbf{c}$ and opacity $\alpha$.
*   Camera parameters (view matrix, projection matrix).
*   Screen resolution $(W, H)$.

**Outputs:**
*   Final color buffer $C_{final}[x,y]$.
*   Final opacity buffer $A_{final}[x,y]$.

**Data Structures:**
*   **Color Buffer:** 2D array of size $W \times H$, storing RGB values.
*   **Opacity Buffer:** 2D array of size $W \times H$, storing alpha values.
*   **Primitive List:** A list of volume samples (voxels or points) sorted by depth relative to the camera.

**Steps:**
1.  **Sorting:** Sort all volume primitives (voxels/points) in front-to-back or back-to-front order relative to the camera [S1, unofficial][S2, book].
2.  **Traversal:** Iterate through the sorted primitives $k = 1 \dots N$.
3.  **Shading:** For each primitive $k$, determine its color $\mathbf{c}_k$ and opacity $\alpha_k$ using the transfer functions and shading equation (e.g., Equation 9 in [S2, book]) [S1, unofficial][S2, book].
4.  **Splatting:** Project the primitive's footprint onto the screen. For each pixel $(x,y)$ covered by the footprint:
    *   Calculate the footprint weight $w(x,y)$ based on the kernel function (e.g., Gaussian) [S1, unofficial][S2, book].
    *   Compute the contribution: $\Delta \mathbf{C} = w(x,y) \cdot \mathbf{c}_k \cdot \alpha_k$ and $\Delta A = w(x,y) \cdot \alpha_k$ [S1, unofficial][S2, book].
5.  **Compositing:** Update the screen buffers using standard alpha blending. If traversing front-to-back:
    $$ \mathbf{C}_{new}[x,y] = \mathbf{C}_{old}[x,y] \cdot (1 - \Delta A) + \Delta \mathbf{C} $$
    $$ A_{new}[x,y] = A_{old}[x,y] \cdot (1 - \Delta A) + \Delta A $$
    (Note: Exact blending equation may vary slightly based on specific implementation of Equation 13 in [S2, book], but standard pre-multiplied alpha blending is implied) [S1, unofficial][S2, book].
6.  **Repeat:** Continue until all primitives are processed.

**Pitfalls:**
*   **Color Bleeding:** Because interpolation (splatting) and compositing are coupled per primitive, color bleeding artifacts can occur [S1, unofficial][S2, book].
*   **Sparkling:** Slight sparkling artifacts may be noticeable in animated viewing due to the inability to separate interpolation and compositing operations caused by the pre-integration of the basis kernel [S1, unofficial][S2, book].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of primitives and $K$ is the average number of pixels covered by a splat. Sorting adds $O(N \log N)$.
*   **Memory Complexity:** $O(W \cdot H)$ for the color and opacity buffers.
*   **Strengths:** Simple implementation; low memory overhead compared to sheet-buffered methods; suitable for real-time if sorting is efficient.
*   **Limitations:** Prone to visual artifacts (bleeding, sparkling); requires strict depth sorting; performance degrades with high overlap or large kernels.
*   **When to use:** Simple volumes, educational purposes, or when implementation simplicity is prioritized over visual quality.
*   **When not to use:** High-quality medical visualization, complex lighting, or when artifacts are unacceptable.

## Background
Composite-only splatting was the first proposed splatting method for volume rendering [S1, unofficial][S2, book]. It was introduced as the most basic object-order technique, establishing the fundamental paradigm of projecting primitives and compositing them directly into the image buffer [S1, unofficial][S2, book]. It served as the baseline against which more advanced techniques like axis-aligned sheet-buffered splatting were developed to mitigate its inherent artifacts [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
