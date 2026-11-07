# Frequency-Domain Methods
Also called: Fourier Series Volume Representation, 4D Frequency Decomposition

## Overview
Frequency-Domain Methods represent volumetric data (including dynamic 4D spacetime volumes) as a superposition of basis functions in the frequency domain, often utilizing tensor decomposition to reduce memory usage. Unlike spatial-domain methods that store values at grid points, this approach stores spectral coefficients. It sets itself apart from sibling methods like [Fourier Projection-Slice Volume Rendering](824-fourier-projection-slice-volume-rendering.md) by focusing on explicit spectral representation and decomposition for compact storage rather than projection-slice extraction for rendering, and from [DCT-based Volume Rendering](5396-dct-based-volume-rendering.md) by using complex exponential Fourier bases instead of discrete cosine bases.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D or 4D scalar field $f(x, y, z)$ or $\mathbf{D}(x, y, z, t)$ defined on a discrete grid.
*   **Output:** A set of decomposed vectors (or tensors) representing the frequency coefficients, allowing reconstruction of the field value at any coordinate.

**Data Structures:**
The method replaces the dense 4D frequency volume $\tilde{\mathbf{D}}(u, v, w, k)$ with a set of low-rank factors using CANDECOMP/PARAFAC (CP) decomposition.
*   **Decomposed Vectors:** Four sets of vectors $\tilde{\mathbf{v}}^U, \tilde{\mathbf{v}}^V, \tilde{\mathbf{v}}^W, \tilde{\mathbf{v}}^K$.
    *   $\tilde{\mathbf{v}}^U \in \mathbb{C}^{U \times R}$: Vectors corresponding to the $x$-axis frequency bins $u=1 \dots U$.
    *   $\tilde{\mathbf{v}}^V \in \mathbb{C}^{V \times R}$: Vectors corresponding to the $y$-axis frequency bins $v=1 \dots V$.
    *   $\tilde{\mathbf{v}}^W \in \mathbb{C}^{W \times R}$: Vectors corresponding to the $z$-axis frequency bins $w=1 \dots W$.
    *   $\tilde{\mathbf{v}}^K \in \mathbb{C}^{K \times R}$: Vectors corresponding to the $t$-axis (time) frequency bins $k=1 \dots K$.
    *   $R$: The rank of the decomposition (number of components).
*   **Positional Encoding (Optional):** A function $f(t)$ used to parameterize time-dependent features, defined as $f(t) = [1, \sin(t), \cos(t), \sin(2t), \cos(2t), \dots]$.

**Numbered Steps:**
1.  **Global Transformation (Implicit):** The input volume is conceptually transformed into the frequency domain. The full 4D frequency tensor $\tilde{\mathbf{D}}(u, v, w, k)$ is factorized into the product of vectors along each axis using CP decomposition [S1].
2.  **Decomposition:** The 4D volume is represented as:
    $$ \mathbf{D}(x, y, z, t) = \sum_{u=1}^U \sum_{v=1}^V \sum_{w=1}^W \sum_{k=1}^K \tilde{\mathbf{D}}(u, v, w, k) \cdot e^{-j2\pi\left(\frac{ux}{U} + \frac{vy}{V} + \frac{wz}{W} + \frac{kt}{K}\right)} $$
    This is approximated by the rank-$R$ decomposition:
    $$ \mathbf{D}(x, y, z, t) = \sum_{r=1}^R \left( \sum_{u=1}^U \tilde{\mathbf{v}}^U(u)_r \cdot e^{-j2\pi\frac{ux}{U}} \right) \cdot \left( \sum_{v=1}^V \tilde{\mathbf{v}}^V(v)_r \cdot e^{-j2\pi\frac{vy}{V}} \right) \cdot \left( \sum_{w=1}^W \tilde{\mathbf{v}}^W(w)_r e^{-j2\pi\frac{wz}{W}} \right) \cdot \left( \sum_{k=1}^K \tilde{\mathbf{v}}^K(k)_r e^{-j2\pi\frac{kt}{K}} \right) $$
    where $\tilde{\mathbf{v}}^U(u)_r$ is the $u$-th element of the $r$-th vector in the $U$-axis factor [S1].
3.  **Evaluation:** To compute the value at a specific spacetime point $(x, y, z, t)$, the inner products of the positional encodings (if used) and the decomposed vectors are calculated. This replaces direct grid sampling with inner products [S1].

**Parameters:**
*   $U, V, W, K$: Number of frequency bins along each axis.
*   $R$: Decomposition rank. Higher $R$ improves accuracy but increases memory and computation [S1].
*   **Training Tricks:** Weighted L1 loss is applied to regularize the decomposed vectors, assigning higher weights to high-frequency coefficients to maintain smoothness. Smoothly bandwidth annealing is used to gradually introduce high-frequency components during training [S1].

**Edge Cases and Pitfalls:**
*   **Color Jittering and Distortion:** The method can suffer from severe color jittering and degraded details compared to spatial methods [S1].
*   **Performance:** Replacing grid sampling with inner products is less efficient, leading to slower rendering speeds [S1].
*   **Memory:** While decomposition reduces memory compared to storing the full 4D frequency volume, it still requires storing $R \times (U+V+W+K)$ vectors [S1].

## Operating characteristics
*   **Time Complexity:** Evaluation requires $O(R \cdot (U + V + W + K))$ operations per point, which is slower than direct grid lookup [S1].
*   **Memory Complexity:** $O(R \cdot (U + V + W + K))$ for the decomposed vectors, significantly less than $O(U \cdot V \cdot W \cdot K)$ for the full tensor [S1].
*   **Strengths:** Compact representation for dynamic 4D scenes; enables differentiable rendering via spectral coefficients.
*   **Limitations:** Degraded visual quality (color jittering); slower evaluation speed; sensitive to rank selection [S1].
*   **When to use:** When memory is constrained for 4D dynamic volumes and high-frequency detail is less critical than compact storage.
*   **When not to use:** When real-time interactive performance or high-fidelity static rendering is required.

## Background
This method originates from recent research on fast representations for dynamic scenes, specifically as a comparison point to spatial methods like HexPlane [S1]. It was explored as an orthogonal approach to spatial decomposition, leveraging Fourier Theory to represent spacetime values. The specific implementation using CP decomposition for 4D volumes and weighted L1 regularization for training was detailed in the HexPlane paper [S1].

## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
