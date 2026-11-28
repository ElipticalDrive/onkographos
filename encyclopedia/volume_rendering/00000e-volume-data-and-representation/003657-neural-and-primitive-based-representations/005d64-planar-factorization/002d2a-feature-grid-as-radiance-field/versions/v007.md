# Feature Grid as Radiance Field

Also called: TensoRF, Chen et al. 2022, EgoNeRF representation, Tensor Factorization

## Overview
Feature Grid as Radiance Field represents a 3D scene by decomposing the high-dimensional radiance field tensor into low-rank components (vectors and matrices), significantly reducing memory usage compared to dense voxel grids. It sets itself apart from [K-planes](3317-k-planes.md) by using a full 3D tensor decomposition (CP or vector-matrix) rather than restricting factors to planar slices, and from [Forplane](5177-forplane.md) by maintaining a unified tensor structure rather than separate planar factorizations for different modalities.

## How it works
The method defines a scene function $F_\Theta: \mathbb{R}^3 \rightarrow \mathbb{R}^k$ (where $k$ includes density and color channels) using explicit feature grids that are factorized into lower-dimensional tensors. Unlike the parent [Planar Factorization](17382-planar-factorization.md) which projects points onto 2D planes, this method treats the scene as a 3D tensor $\mathcal{G}$ and decomposes it into a sum of outer products of vectors and matrices.

### Inputs and Outputs
*   **Input:** A set of posed images $\{I_i\}$ with corresponding camera parameters. A query point $\mathbf{x} \in \mathbb{R}^3$ (spatial coordinates) and viewing direction $\mathbf{d} \in \mathbb{R}^3$.
*   **Output:** Volume density $\sigma(\mathbf{x})$ and color $\mathbf{c}(\mathbf{x}, \mathbf{d})$.

### Data Structures
The radiance field is represented by two primary feature grids:
1.  **Density Grid ($\mathcal{G}_\sigma$):** Stores volume density.
2.  **Appearance Grid ($\mathcal{G}_a$):** Stores neural appearance features.

Both grids are decomposed into low-rank components. The decomposition uses the outer product ($\otimes$) of vectors and matrices.

**Density Grid Decomposition:**
The density grid $\mathcal{G}_\sigma$ is decomposed into a sum of terms involving vectors $\mathbf{v}$ and matrices $\mathbf{M}$. For a specific component $y$ (e.g., Yin/Yang in EgoNeRF, or just the global grid in TensoRF), the decomposition is:
$$ \mathcal{G}_\sigma^y = \sum_{n=1}^{N_\sigma} \left( \mathbf{v}_{\sigma,n}^{y,R} \otimes \mathbf{M}_{\sigma,n}^{y,\Theta\Phi} + \mathbf{v}_{\sigma,n}^{y,\Theta} \otimes \mathbf{M}_{\sigma,n}^{y,\Phi R} + \mathbf{v}_{\sigma,n}^{y,\Phi} \otimes \mathbf{M}_{\sigma,n}^{y,R\Theta} \right) $$
Where:
*   $N_\sigma$ is the rank of the decomposition.
*   $\mathbf{v}_{\sigma,n}^{y,R}, \mathbf{v}_{\sigma,n}^{y,\Theta}, \mathbf{v}_{\sigma,n}^{y,\Phi}$ are vectors corresponding to the radial ($R$), polar ($\Theta$), and azimuthal ($\Phi$) dimensions (or Cartesian $X,Y,Z$ in standard TensoRF).
*   $\mathbf{M}_{\sigma,n}^{y,\Theta\Phi}$ is a matrix formed by the outer product of the $\Theta$ and $\Phi$ vectors, or a learned matrix factor.
*   The notation $\mathcal{A}_{\sigma,n}^{y,m}$ in [S2] refers to the combined term $\mathbf{v} \otimes \mathbf{M}$.

**Appearance Grid Decomposition:**
The appearance grid $\mathcal{G}_a$ is decomposed similarly, but typically uses vectors for all three dimensions in a CP-like structure or mixed vector-matrix forms:
$$ \mathcal{G}_a^y = \sum_{n=1}^{N_a} \left( \mathcal{A}_{a,n}^{y,R} \otimes \mathbf{b}_{3n-2}^y + \mathcal{A}_{a,n}^{y,\Theta} \otimes \mathbf{b}_{3n-1}^y + \mathcal{A}_{a,n}^{y,\Phi} \otimes \mathbf{b}_{3n}^y \right) $$
Where $\mathcal{A}$ and $\mathbf{b}$ are vector/matrix factors.

**Union of Components:**
If the scene is split into components $Y$ (e.g., Yin and Yang for handling sparse/dense regions separately as in EgoNeRF):
$$ \mathcal{G}_\sigma = \bigcup_{y \in Y} \mathcal{G}_\sigma^y, \quad \mathcal{G}_a = \bigcup_{y \in Y} \mathcal{G}_a^y $$

### Query Process
1.  **Interpolation:** For a query point $\mathbf{x}$, features are retrieved from the factorized grids. In EgoNeRF, this uses trilinear interpolation $\mathcal{T}$ on the balanced spherical grids [S2]. In standard TensoRF, the query is computed by evaluating the tensor decomposition directly:
    $$ \mathbf{f}_\sigma(\mathbf{x}) = \mathcal{G}_\sigma(\mathbf{x}) $$
    $$ \mathbf{f}_a(\mathbf{x}) = \mathcal{G}_a(\mathbf{x}) $$
    The evaluation of a decomposed tensor $\mathcal{G} = \sum_n \mathbf{v}_n \otimes \mathbf{M}_n$ at point $\mathbf{x}$ involves projecting $\mathbf{x}$ onto the relevant dimensions and summing the results.

2.  **Decoding:**
    *   **Density:** $\sigma(\mathbf{x}) = \mathcal{T}(\mathcal{G}_\sigma, \mathbf{x})$ [S2].
    *   **Color:** A small MLP $f_{\text{MLP}}$ decodes the appearance features and viewing direction:
        $$ \mathbf{c}(\mathbf{x}, \mathbf{d}) = f_{\text{MLP}}(\mathcal{T}(\mathcal{G}_a, \mathbf{x}), \mathbf{d}) $$ [S2].

### Parameters
*   **Resolution:** Grid resolution (e.g., $N_y^r, N_y^\theta, N_y^\phi$ in EgoNeRF) [S2].
*   **Rank ($N_\sigma, N_a$):** The number of terms in the decomposition. Higher rank increases accuracy but also memory.
*   **MLP Size:** Tiny MLP for color decoding [S2].

### Edge Cases and Pitfalls
*   **Precision:** Trilinear interpolation on factorized grids may introduce smoothing artifacts if the rank is too low.
*   **Boundaries:** Handling points outside the grid bounds requires clamping or extrapolation.
*   **Sparse Regions:** Standard dense grids waste memory on empty space. EgoNeRF addresses this with Yin/Yang grids [S2].

## Subtopics

### [Masked Wavelet Representation for Compact Neural Radiance Fields](9619-masked-wavelet-representation-for-compact-neural-radiance-fi.md)
Reduces memory by applying DWT and masking to grid coefficients.

### [3D Bilateral Grid Slicing](10529-3d-bilateral-grid-slicing.md)
Approximates view-dependent color transformations using affine matrices in a 3D grid.

## Operating characteristics
*   **Memory Complexity:** Reduces space complexity from $O(n^3)$ for dense grids to $O(n^2)$ for low-rank tensor factorization [S2].
*   **Training Time:** Memory-efficient; trains in approximately 30 minutes for comparable quality to NeRF [S1, unofficial].
*   **Strengths:** High memory efficiency, fast training, comparable quality to dense NeRF.
*   **Limitations:** Lower rank may lose high-frequency details; interpolation on factorized grids can be slower than direct lookup if not optimized.
*   **When to use:** When memory is constrained or training speed is critical.
*   **When not to use:** When extremely high-frequency detail is required and memory is abundant.

## Background
Originated with TensoRF (Chen et al. 2022) [S1, unofficial], which decomposed the radiance field grid into low-rank tensor components. EgoNeRF (2023) [S2] adapted this using balanced spherical grids and Yin/Yang decomposition for egocentric view synthesis.

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
