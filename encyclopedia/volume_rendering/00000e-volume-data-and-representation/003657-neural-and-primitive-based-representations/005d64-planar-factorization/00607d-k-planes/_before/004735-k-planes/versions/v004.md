# K-planes
Also called: hex-planes, k-planes factorization, tri-planes

## Overview
K-planes is a white-box, hybrid representation for volumetric rendering in arbitrary dimensions $d$ that factorizes the scene function into $k = \binom{d}{2}$ learned 2D feature planes [S2]. It solves the scalability and black-box nature of implicit neural fields by using explicit planar grids combined with a decoder, achieving high compression (e.g., 1000x over 4D grids) and fast optimization [S2]. Unlike sibling methods that add features from planes [S2], K-planes multiplies them element-wise, which enables spatial localization and allows the use of a linear decoder with a learned color basis instead of a non-linear MLP [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{q} \in \mathbb{R}^d$ (typically spatial coordinates $\mathbf{x}$ and optionally viewing direction $\mathbf{d}$ or time $t$).
*   **Output:** Density $\sigma \in \mathbb{R}$ and color $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
The core data structure is a set of $k = \binom{d}{2}$ 2D feature planes $\mathbf{P} = \{ \mathbf{P}_c \}_{c \in C}$.
*   **Plane Layout:** Each plane $\mathbf{P}_c$ has shape $N \times N \times M$.
    *   $N$: Symmetric spatial (and temporal, for space-time planes) resolution.
    *   $M$: Feature dimension size, capturing density and view-dependent color information.
*   **Multiscale Extension:** To capture multi-resolution details, the model maintains $S$ copies of the planes at different resolutions (e.g., $N \in \{64, 128, 256, 512\}$) [S2]. The features from all scales are concatenated before decoding.
*   **Plane Selection:**
    *   For $d=3$ (static 3D), $k=3$ planes: $xy, xz, yz$ (tri-planes) [S2].
    *   For $d=4$ (dynamic 4D), $k=6$ planes: $xy, xz, yz, xt, yt, zt$ (hex-planes) [S2].
    *   For general $d$, planes correspond to every unique pair of dimensions [S2].

**Parameters:**
*   $d$: Dimensionality of the scene space (typically 3 or 4) [S2].
*   $N$: Resolution of the planes [S2].
*   $M$: Feature vector size [S2].
*   $S$: Number of multiscale resolutions [S2].

**Algorithm Steps:**

1.  **Coordinate Normalization and Projection:**
    Normalize the query point $\mathbf{q}$ entries to the range $[0, N)$. For each plane $c \in C$, project $\mathbf{q}$ onto the 2D axes corresponding to that plane to obtain 2D coordinates $\pi_c(\mathbf{q})$ [S2].
    *   Example for 4D $\mathbf{q}=(x,y,z,t)$: For plane $P_{xt}$, coordinates are $(x, t)$ [S2].

2.  **Multiscale Bilinear Interpolation:**
    For each plane $c$ and each scale $s \in \{1, \dots, S\}$, perform bilinear interpolation $\psi$ on the plane grid $\mathbf{P}_c^{(s)}$ at the projected coordinates to retrieve a feature vector $\mathbf{f}(\mathbf{q})_c^{(s)}$ [S2]:
    $$ \mathbf{f}(\mathbf{q})_c^{(s)} = \psi(\mathbf{P}_c^{(s)}, \pi_c(\mathbf{q})) $$
    where $\psi$ denotes bilinear interpolation of a point into a regularly spaced 2D grid [S2].

3.  **Feature Combination (Hadamard Product):**
    Combine the features from all $k$ planes for a specific scale by multiplying them element-wise (Hadamard product) [S2]:
    $$ \mathbf{f}_{scale}(\mathbf{q}) = \prod_{c \in C} \mathbf{f}(\mathbf{q})_c^{(s)} $$
    This results in a feature vector of length $M$ for that scale [S2].

4.  **Concatenation:**
    Concatenate the feature vectors from all $S$ scales to form the final input feature vector $\mathbf{f}(\mathbf{q})$ for the decoder [S2].

5.  **Decoding:**
    Decode $\mathbf{f}(\mathbf{q})$ into density $\sigma$ and color $\mathbf{c}$ using one of two methods:

    *   **Option A: Linear Decoder (Explicit Model)**
        Uses a learned color basis. Density is predicted linearly, and color is predicted using a basis dependent on view direction $\mathbf{d}$ [S2].
        $$ \sigma(\mathbf{q}) = \mathbf{f}(\mathbf{q}) \cdot \mathbf{b}_\sigma $$
        $$ \mathbf{c}(\mathbf{q}, \mathbf{d}) = \sum_{j=1}^{M} \mathbf{f}(\mathbf{q})_j \cdot \mathbf{B}_{RGB}(\mathbf{d})_j $$
        where $\mathbf{b}_\sigma$ is a learned density basis vector and $\mathbf{B}_{RGB}(\mathbf{d})$ is a view-dependent color basis [S2].
        Apply sigmoid to $\mathbf{c}$ and exponential (with truncated gradient) to $\sigma$ to ensure valid ranges [S2].

    *   **Option B: MLP Decoder (Hybrid Model)**
        Uses two small MLPs: $g_\sigma$ and $g_{RGB}$ [S2].
        $$ \hat{\mathbf{f}}(\mathbf{q}) = g_\sigma(\mathbf{f}(\mathbf{q})) $$
        $$ \sigma(\mathbf{q}) = \exp(\hat{\mathbf{f}}_\sigma) $$
        $$ \mathbf{c}(\mathbf{q}, \mathbf{d}) = \text{sigmoid}(g_{RGB}(\hat{\mathbf{f}}_{RGB}, \gamma(\mathbf{d}))) $$
        where $\gamma(\mathbf{d})$ is an embedding of the view direction [S2].

6.  **Rendering:**
    Render the ray color and density using the standard volumetric rendering formula [S2].

**Regularization Losses:**
*   **Spatial Total Variation ($\mathcal{L}_{TV}$):** Encourages sparse or smooth gradients in space [S2].
    $$ \mathcal{L}_{TV}(\mathbf{P}) = \frac{1}{|C|n^2} \sum_{c,i,j} \left( \|\mathbf{P}_c^{i,j} - \mathbf{P}_c^{i-1,j}\|_2^2 + \|\mathbf{P}_c^{i,j} - \mathbf{P}_c^{i,j-1}\|_2^2 \right) $$
    Applied to space-only planes and spatial dimensions of space-time planes [S2].
*   **Temporal Smoothness ($\mathcal{L}_{smooth}$):** Penalizes sharp acceleration in time using a 1D Laplacian filter [S2].
    $$ \mathcal{L}_{smooth}(\mathbf{P}) = \frac{1}{|C|n^2} \sum_{c,i,t} \|\mathbf{P}_c^{i,t-1} - 2\mathbf{P}_c^{i,t} + \mathbf{P}_c^{i,t+1}\|_2^2 $$
    Applied only to the time dimension of space-time planes ($xt, yt, zt$) [S2].
*   **Sparse Transients ($\mathcal{L}_{sep}$):** Encourages space-time planes to remain at the multiplicative identity (1) for static regions [S2].
    $$ \mathcal{L}_{sep}(\mathbf{P}) = \sum_{c \in \{xt, yt, zt\}} \|\mathbf{1} - \mathbf{P}_c\|_1 $$
    Space-time planes are initialized to 1 [S2].

**Sampling Strategies:**
*   **Proposal Sampling:** Uses a variant of Mip-NeRF 360 proposal sampling with a small instance of K-planes as a density model. A two-stage sampler refines density estimates iteratively [S2].
*   **Importance Sampling (IST):** For multiview dynamic scenes, after static convergence, sample training rays proportionally to the maximum color variation within 25 frames before or after [S2].

**Edge Cases and Pitfalls:**
*   **Unbounded Scenes:** For unbounded scenes, implement an $l_\infty$ version of scene contraction [S2].
*   **Forward-Facing Scenes:** Apply Normalized Device Coordinates (NDC) [S2].
*   **Monocular Dynamic Scenes:** Importance sampling (IST) cannot be used for monocular dynamic scenes [S2].

## Subtopics

### [TriPlane NeRF](6932-triplane-nerf.md)
TriPlane NeRF is a hybrid 3D representation that aligns explicit features along three axis-aligned orthogonal feature planes to solve the scaling problems of voxel-based representations while maintaining efficiency and expressiveness. It differs from its parent [K-planes](3317-k-planes.md) by restricting the factorization to the three primary spatial coordinate planes ($XY, XZ, YZ$) rather than all $d$-choose-2 pairs, and from [Voxel NeRF](52-name.md) by using explicit feature planes instead of a dense 3D grid, which allows the decoder to remain small and enables the planes to be optimized via gradients or generated by CNNs.

### K-planes MLP Decoder
A hybrid model where K-planes features are decoded by small MLPs, similar to Instant-NGP and DVGO [S2]. Features are decoded by two small MLPs: $g_\sigma$ maps features $\mathbf{f}(\mathbf{q})$ into density $\sigma$ and additional features $\hat{\mathbf{f}}$; $g_{RGB}$ maps $\hat{\mathbf{f}}$ and embedded view direction $\gamma(\mathbf{d})$ into RGB color [S2]. Predicted density and color values are normalized via exponential and sigmoid, respectively [S2].

### Appearance Code Augmentation
An extension to handle scenes with varying appearance (e.g., changing illumination) by adding a global appearance code per frame to disentangle global appearance from scene geometry [S2]. The K-planes model (either linear or MLP decoder) is augmented with a learned M-dimensional vector for each frame, passed as an additional input to the color decoder/basis so it affects color but not geometry [S2].

## Operating characteristics
*   **Memory:** Achieves ~1000x compression over a full 4D grid. A 4D volume represented by K-planes uses ~200 MB, whereas a direct representation at the same resolution requires >300 GB [S2].
*   **Speed:** Trains orders of magnitude faster than prior implicit models and on par with concurrent hybrid models [S2].
*   **Scalability:** Scales gracefully with dimension $d$ in both optimization time and model size, using $k = \binom{d}{2}$ planes [S2].
*   **Strengths:** White-box/interpretable, low memory, fast optimization, supports arbitrary dimensions (static 3D, dynamic 4D, varying appearance) [S2].
*   **Limitations:** Requires careful regularization for dynamic scenes (temporal smoothness, sparse transients) [S2]. Multiplication-based combination is critical for linear decoder performance [S2].

## Background
K-planes was introduced in January 2023 [S1, S2]. It was developed as a white-box alternative to black-box implicit neural fields, inspired by explicit modeling approaches like Plenoxels and spatial decompositions like Triplane (EG3D) and Tensor Decomposition (TensoRF) [S2]. It extends the concept of tri-planes to arbitrary dimensions, including hex-planes for 4D dynamic scenes [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] K-Planes: Explicit Radiance Fields in Space, Time, and Appearance, https://doi.org/10.1109/cvpr52729.2023.01201, fetched 2026-10-26
