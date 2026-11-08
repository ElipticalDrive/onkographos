# Feature Grid as Radiance Field

Also called: EgoNeRF representation, Tensor Factorization

## Overview
Feature Grid as Radiance Field represents a scene using explicit, low-rank tensor-decomposed feature grids for density and appearance, replacing the implicit coordinate-to-color mapping of standard MLP-based NeRFs [S1]. It solves the problem of high memory consumption and slow training associated with dense 3D voxel grids by factorizing the feature tensors into lower-dimensional vector and matrix components, reducing space complexity from $O(n^3)$ to $O(n^2)$ [S1]. Unlike sibling methods that rely on hash grids or Gaussian primitives, this approach maintains a regular, structured grid layout (specifically balanced spherical grids for egocentric scenes) while achieving compactness through tensor factorization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of egocentric images with corresponding camera parameters (position and viewing direction).
*   **Output:** A 3D scene representation allowing novel view synthesis. The scene function $F: \mathbb{R}^3 \times \mathbb{R}^3 \rightarrow \mathbb{R}^4$ maps a 3D spatial position $\mathbf{x}$ and a 3D viewing direction $\mathbf{d}$ to volume density $\sigma$ and RGB color $\mathbf{c}$.

**Data Structures:**
The representation consists of two primary grids, $\mathcal{G}_\sigma$ and $\mathcal{G}_a$, defined over a "balanced spherical grid" parameterization. This grid is composed of Yin and Yang components ($Y = \{\text{Yin, Yang}\}$) to handle the singularity at the poles of spherical coordinates [S1].

1.  **Density Grid ($\mathcal{G}_\sigma$):**
    *   Structure: Union of Yin and Yang components: $\mathcal{G}_\sigma = \bigcup_{y \in Y} \mathcal{G}_\sigma^y$.
    *   Dimensions: Each component $\mathcal{G}_\sigma^y$ has resolution $2N_y^r \times N_y^\theta \times N_y^\phi$.
    *   Content: Single channel storing explicit volume density values.
    *   Storage: Decomposed via low-rank tensor factorization into vectors and matrices (see below).

2.  **Appearance Grid ($\mathcal{G}_a$):**
    *   Structure: Union of Yin and Yang components: $\mathcal{G}_a = \bigcup_{y \in Y} \mathcal{G}_a^y$.
    *   Dimensions: Each component $\mathcal{G}_a^y$ has resolution $2N_y^r \times N_y^\theta \times N_y^\phi \times C$, where $C$ is the feature dimension.
    *   Content: $C$-dimensional neural appearance features.
    *   Storage: Decomposed via low-rank tensor factorization into vectors and matrices (see below).

**Tensor Factorization Details:**
The dense 3D grids are factorized to reduce memory. Let $N_\sigma$ and $N_a$ be the rank parameters for density and appearance, respectively. The outer product is denoted by $\otimes$.

*   **Density Factorization ($\mathcal{G}_\sigma^y$):**
    $$ \mathcal{G}_\sigma^y = \sum_{n=1}^{N_\sigma} \mathbf{v}_{\sigma,n}^{y,R} \otimes \mathbf{M}_{\sigma,n}^{y,\Theta\Phi} + \mathbf{v}_{\sigma,n}^{y,\Theta} \otimes \mathbf{M}_{\sigma,n}^{y,\Phi R} + \mathbf{v}_{\sigma,n}^{y,\Phi} \otimes \mathbf{M}_{\sigma,n}^{y,R\Theta} $$
    Here, $\mathbf{v}$ represents vectors along one spherical axis (Radius $R$, Theta $\Theta$, Phi $\Phi$), and $\mathbf{M}$ represents matrices spanning the other two axes. For example, $\mathbf{v}_{\sigma,n}^{y,R}$ is a vector of size $2N_y^r$, and $\mathbf{M}_{\sigma,n}^{y,\Theta\Phi}$ is a matrix of size $N_y^\theta \times N_y^\phi$.

*   **Appearance Factorization ($\mathcal{G}_a^y$):**
    $$ \mathcal{G}_a^y = \sum_{n=1}^{N_a} \mathcal{A}_{a,n}^{y,R} \otimes \mathbf{b}_{3n-2}^y + \mathcal{A}_{a,n}^{y,\Theta} \otimes \mathbf{b}_{3n-1}^y + \mathcal{A}_{a,n}^{y,\Phi} \otimes \mathbf{b}_{3n}^y $$
    Here, $\mathcal{A}$ represents matrices spanning two spherical axes, and $\mathbf{b}$ represents vectors along the third. The indices $3n-2, 3n-1, 3n$ group the three factors for each rank $n$.

**Reconstruction Steps:**
1.  **Interpolation:** Given a query point $\mathbf{x}$ (mapped to spherical coordinates $r, \theta, \phi$) and a viewing direction $\mathbf{d}$:
    *   Compute the density feature vector $\mathbf{f}_\sigma$ by trilinearly interpolating the reconstructed density grid $\mathcal{G}_\sigma$ at $\mathbf{x}$. Since $\mathcal{G}_\sigma$ is factorized, this involves summing the interpolated contributions from the vector/matrix factors.
    *   Compute the appearance feature vector $\mathbf{f}_a$ by trilinearly interpolating the reconstructed appearance grid $\mathcal{G}_a$ at $\mathbf{x}$.
2.  **Density Output:**
    $$ \sigma(\mathbf{x}) = \mathcal{T}(\mathcal{G}_\sigma, \mathbf{x}) $$
    Where $\mathcal{T}$ denotes the trilinear interpolation operation on the factorized grid.
3.  **Color Output:**
    $$ \mathbf{c}(\mathbf{x}, \mathbf{d}) = f_{\text{MLP}}(\mathbf{f}_a, \mathbf{d}) $$
    Where $f_{\text{MLP}}$ is a small Multi-Layer Perceptron that decodes the appearance features $\mathbf{f}_a$ and the viewing direction $\mathbf{d}$ into the final RGB color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_feature_grid(x, d, grids, mlp):
    """
    Query the Feature Grid representation for density and color.
    
    Args:
        x: 3D spatial position (x, y, z).
        d: 3D viewing direction (dx, dy, dz).
        grids: Dictionary containing 'sigma' and 'appearance' grids.
               Each grid is a list of components (Yin, Yang).
               Each component contains factorized tensors (vectors/matrices).
        mlp: A small neural network (MLP) for color decoding.
        
    Returns:
        sigma: Volume density.
        color: RGB color vector.
    """
    # 1. Convert Cartesian position x to spherical coordinates (r, theta, phi)
    # Note: Implementation depends on the specific spherical parameterization
    # used for the balanced grid (Yin/Yang split).
    r, theta, phi = cartesian_to_spherical(x)
    
    # 2. Interpolate Density
    sigma = 0.0
    for y in ['Yin', 'Yang']:
        # Reconstruct density value from factorized grid G_sigma^y
        # This involves interpolating the vectors and matrices and summing them
        # according to the formula: sum_n (v_R @ M_ThetaPhi + v_Theta @ M_PhiR + v_Phi @ M_RTheta)
        sigma_y = interpolate_factorized_density_grid(grids['sigma'][y], r, theta, phi)
        sigma += sigma_y
        
    # 3. Interpolate Appearance Features
    # Initialize feature vector of size C
    appearance_features = zeros(C)
    for y in ['Yin', 'Yang']:
        # Reconstruct appearance features from factorized grid G_a^y
        # Formula: sum_n (A_R @ b_3n-2 + A_Theta @ b_3n-1 + A_Phi @ b_3n)
        features_y = interpolate_factorized_appearance_grid(grids['appearance'][y], r, theta, phi)
        appearance_features += features_y
        
    # 4. Decode Color
    color = mlp(appearance_features, d)
    
    return sigma, color

def interpolate_factorized_density_grid(grid_component, r, theta, phi):
    """
    Interpolate the density grid component using its factorized representation.
    """
    # grid_component contains:
    # v_sigma_n^R (vector), M_sigma_n^ThetaPhi (matrix)
    # v_sigma_n^Theta (vector), M_sigma_n^PhiR (matrix)
    # v_sigma_n^Phi (vector), M_sigma_n^RTheta (matrix)
    # for n in 1..N_sigma
    
    density = 0.0
    for n in range(1, N_sigma + 1):
        # Interpolate vector v_R at r
        v_r = trilinear_interpolate_vector(grid_component.v_sigma_n_R[n], r)
        # Interpolate matrix M_ThetaPhi at (theta, phi)
        m_theta_phi = trilinear_interpolate_matrix(grid_component.M_sigma_n_ThetaPhi[n], theta, phi)
        density += dot(v_r, m_theta_phi) # Outer product contraction
        
        # Interpolate vector v_Theta at theta
        v_theta = trilinear_interpolate_vector(grid_component.v_sigma_n_Theta[n], theta)
        # Interpolate matrix M_PhiR at (phi, r)
        m_phi_r = trilinear_interpolate_matrix(grid_component.M_sigma_n_PhiR[n], phi, r)
        density += dot(v_theta, m_phi_r)
        
        # Interpolate vector v_Phi at phi
        v_phi = trilinear_interpolate_vector(grid_component.v_sigma_n_Phi[n], phi)
        # Interpolate matrix M_RTheta at (r, theta)
        m_r_theta = trilinear_interpolate_matrix(grid_component.M_sigma_n_RTheta[n], r, theta)
        density += dot(v_phi, m_r_theta)
        
    return density

def interpolate_factorized_appearance_grid(grid_component, r, theta, phi):
    """
    Interpolate the appearance grid component using its factorized representation.
    Returns a vector of size C.
    """
    features = zeros(C)
    for n in range(1, N_a + 1):
        # Interpolate matrix A_R at (theta, phi)
        a_r = trilinear_interpolate_matrix(grid_component.A_a_n_R[n], theta, phi)
        # Interpolate vector b_3n-2 at r
        b_1 = trilinear_interpolate_vector(grid_component.b_3n_minus_2[n], r)
        features += outer_product(a_r, b_1) # Adds matrix-vector outer product to features
        
        # Interpolate matrix A_Theta at (phi, r)
        a_theta = trilinear_interpolate_matrix(grid_component.A_a_n_Theta[n], phi, r)
        # Interpolate vector b_3n-1 at theta
        b_2 = trilinear_interpolate_vector(grid_component.b_3n_minus_1[n], theta)
        features += outer_product(a_theta, b_2)
        
        # Interpolate matrix A_Phi at (r, theta)
        a_phi = trilinear_interpolate_matrix(grid_component.A_a_n_Phi[n], r, theta)
        # Interpolate vector b_3n at phi
        b_3 = trilinear_interpolate_vector(grid_component.b_3n[n], phi)
        features += outer_product(a_phi, b_3)
        
    return features
```

**Parameters:**
*   $N_y^r, N_y^\theta, N_y^\phi$: Grid resolutions for radius, theta, and phi. Typical values are not specified in the sources, but the grid is described as "balanced spherical" [S1].
*   $C$: Dimension of the neural appearance features. Not given in the sources.
*   $N_\sigma, N_a$: Ranks of the tensor factorization for density and appearance grids. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Singularities:** The use of spherical coordinates introduces singularities at the poles ($\theta=0, \pi$). The "balanced spherical grid" with Yin and Yang components is used to mitigate this, but implementation must correctly handle the coordinate mapping and interpolation near these boundaries [S1].
*   **Precision:** Trilinear interpolation on factorized grids may introduce approximation errors compared to dense grid lookups. The rank $N_\sigma, N_a$ controls this trade-off.
*   **Boundary Conditions:** The grid is defined over a specific spherical volume. Queries outside this volume require clamping or out-of-bounds handling.

## Subtopics

### [Masked Wavelet Representation for Compact Neural Radiance Fields](9619-masked-wavelet-representation-for-compact-neural-radiance-fi.md)
Masked Wavelet Representation (MW-NeRF) reduces the memory footprint of grid-based neural radiance fields by transforming spatial grid coefficients into the wavelet domain, applying a trainable binary mask to induce sparsity, and compressing the result. It sets itself apart from standard grid-based methods like [Feature Grid as Radiance Field](5508-feature-grid-as-radiance-field.md) by leveraging the compactness of wavelet coefficients for smooth signals and per-scene optimized masking, achieving state-of-the-art performance within a 2 MB memory budget.

### [3D Bilateral Grid Slicing](10529-3d-bilateral-grid-slicing.md)
3D Bilateral Grid Slicing is a method for approximating continuous, view-dependent color transformations (such as Image Signal Processing or ISP) by storing affine transformation matrices in a 3D grid indexed by 2D spatial coordinates $(x, y)$ and a 1D color guidance dimension (grayscale intensity). It solves the problem of learning a continuous function that maps pixel coordinates and colors to affine matrices, enabling efficient, differentiable color correction within radiance field representations. It differs from sibling methods like Masked Wavelet Representation by focusing on explicit color-space transformations via grid sampling rather than compact spectral decomposition of the radiance field itself.

## Operating characteristics
*   **Time Complexity:** Query time is dominated by the interpolation of the factorized grids. For a grid of resolution $n$, dense interpolation is $O(1)$, but factorized interpolation requires summing $N$ terms, each involving vector/matrix operations. The exact time complexity per query is not explicitly stated, but it is faster than dense grid lookup for high resolutions due to reduced memory bandwidth, though potentially slower than hash grids for very low resolutions.
*   **Memory Complexity:** The low-rank tensor factorization reduces space complexity from $O(n^3)$ for a dense 3D grid to $O(n^2)$ [S1]. This is a significant reduction for high-resolution grids.
*   **Measured Performance:** The method is designed for egocentric view synthesis [S1]. Specific performance metrics (FPS, memory usage in MB) are not provided in the sources.
*   **Strengths:**
    *   Reduced memory footprint compared to dense voxel grids.
    *   Regular grid structure allows for efficient trilinear interpolation.
    *   Suitable for egocentric scenes with spherical symmetry.
*   **Limitations:**
    *   Limited to scenes that can be reasonably represented in spherical coordinates (egocentric).
    *   Tensor factorization rank must be tuned; too low leads to loss of detail, too high reduces memory benefits.
    *   Handling of non-spherical scenes or large-scale environments may require complex partitioning or multiple grids.
*   **When to Use:** Ideal for egocentric view synthesis where memory is constrained and the scene has a natural spherical structure.
*   **When Not to Use:** For large-scale outdoor scenes or scenes with complex, non-spherical geometry where hash grids or Gaussian splatting might be more appropriate.

## Background
The Feature Grid as Radiance Field, specifically in the form of EgoNeRF, was introduced in the paper "Balanced Spherical Grid for Egocentric View Synthesis" [S1]. It addresses the limitations of MLP-based NeRFs, which suffer from slow training and high memory usage for dense representations, by using explicit feature grids. The key innovation is the application of low-rank tensor factorization to these grids, inspired by previous work on tensor decomposition [S1, citing [7]]. The method is tailored for egocentric (first-person) view synthesis, utilizing a balanced spherical grid to handle the coordinate system's singularities.

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
