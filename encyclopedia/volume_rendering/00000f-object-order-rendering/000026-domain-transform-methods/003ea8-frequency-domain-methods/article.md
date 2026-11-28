# Frequency-Domain Methods
## Overview
Frequency-Domain Methods represent 4D spacetime volumetric data as a superposition of Fourier basis functions, utilizing CANDECOMP/PARAFAC (CP) tensor decomposition to reduce memory and computation [S1]. Unlike spatial decomposition approaches, this method operates orthogonally by transforming the representation into the frequency domain [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D spacetime scalar field $\mathbf{D}(x, y, z, t)$ defined on a discrete grid with dimensions $U, V, W, K$ along the spatial and temporal axes [S1].
*   **Output:** A set of decomposed vectors representing the frequency coefficients, enabling reconstruction of the field value at any coordinate [S1].

**Data Structures:**
The method replaces the dense 4D frequency volume $\tilde{\mathbf{D}}(u, v, w, k)$ with low-rank factors via CP decomposition [S1].
*   **Decomposed Vectors:** Four sets of vectors $\tilde{\mathbf{v}}^U, \tilde{\mathbf{v}}^V, \tilde{\mathbf{v}}^W, \tilde{\mathbf{v}}^K$, each of shape $\mathbb{C}^{N \times R}$ where $N \in \{U, V, W, K\}$ and $R$ is the decomposition rank [S1].
*   **Positional Encoding (Optional):** A function $f(t)$ parameterizing time-dependent features, defined as $f(t) = [1, \sin(t), \cos(t), \sin(2t), \cos(2t), \dots]$ or using Legendre/Chebyshev polynomials [S1].

**Numbered Steps:**
1.  **Global Transformation:** The input volume is represented in the frequency domain using a 4D Fourier series [S1].
2.  **Decomposition:** The full frequency tensor is factorized into the product of vectors along each axis using CP decomposition [S1]. The value at any spacetime point is computed as:
    $$ \mathbf{D}(x, y, z, t) = \sum_{r=1}^R \left( \sum_{u=1}^U \tilde{\mathbf{v}}^U(u)_r \cdot e^{-j2\pi\frac{ux}{U}} \right) \cdot \left( \sum_{v=1}^V \tilde{\mathbf{v}}^V(v)_r \cdot e^{-j2\pi\frac{vy}{V}} \right) \cdot \left( \sum_{w=1}^W \tilde{\mathbf{v}}^W(w)_r e^{-j2\pi\frac{wz}{W}} \right) \cdot \left( \sum_{k=1}^K \tilde{\mathbf{v}}^K(k)_r e^{-j2\pi\frac{kt}{K}} \right) $$
    where $\tilde{\mathbf{v}}^U(u)_r$ denotes the $u$-th element of the $r$-th vector in the $U$-axis factor [S1].
3.  **Evaluation:** The method replaces grid sampling with inner products of the positional encodings (if used) and the decomposed vectors [S1].

**Parameters:**
*   $U, V, W, K$: Number of frequency bins along each axis [S1].
*   $R$: Decomposition rank [S1].
*   **Training Tricks:** Weighted L1 loss regularizes the decomposed vectors, assigning higher weights to high-frequency coefficients to maintain smoothness [S1]. Smoothly bandwidth annealing gradually introduces high-frequency components during training [S1].

**Edge Cases and Pitfalls:**
*   **Visual Degradation:** The method suffers from severe color jittering, distortions, less-saturated colors, and degraded details compared to spatial methods [S1].
*   **Performance:** Replacing grid sampling with inner products is less efficient, leading to slower evaluation speeds [S1].
*   **Memory:** While decomposition reduces memory compared to storing the full 4D frequency volume, it still requires storing $R \times (U+V+W+K)$ vectors [S1].

## Operating characteristics
*   **Time Complexity:** Evaluation requires computing inner products across $R$ components, which is slower than direct grid lookup [S1].
*   **Memory Complexity:** Requires storing $R \times (U+V+W+K)$ elements for the decomposed vectors, reducing memory compared to the full tensor [S1].
*   **Strengths:** Compact representation for dynamic 4D scenes [S1].
*   **Limitations:** Severe color jittering and distortions; slower evaluation speed; additional matrix-vector production reduces overall speeds compared to spatial alternatives [S1].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
This method originates from research on fast representations for dynamic scenes, specifically explored as an orthogonal approach to spatial decomposition methods like HexPlane [S1]. It leverages Fourier Theory to represent 4D spacetime values in the frequency domain and applies CANDECOMP decomposition to reduce computation and memory usage [S1]. The specific implementation using CP decomposition for 4D volumes and weighted L1 regularization was detailed in the HexPlane paper [S1].

## Sources
[S1] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
