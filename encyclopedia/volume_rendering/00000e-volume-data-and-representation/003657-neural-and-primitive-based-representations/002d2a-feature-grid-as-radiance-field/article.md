# Feature Grid as Radiance Field

Also called: EgoNeRF representation, Tensor Factorization

## Overview
Feature Grid as Radiance Field represents a scene using explicit, low-rank tensor-decomposed feature grids for density and appearance, replacing the implicit multi-layer perceptron (MLP) regression used in standard Neural Radiance Fields. It solves the memory bottleneck of dense 3D grids by factorizing the feature tensors into lower-dimensional components, reducing space complexity from cubic to quadratic relative to resolution. Unlike sibling methods that rely on hash grids [Hash Grid Neural Fields](3855-hash-grid-neural-fields.md) or Gaussian primitives [Gaussian Head Avatar (GHA)](1478-gaussian-head-avatar-gha.md), this method maintains a regular grid structure optimized for spherical parameterization, specifically targeting egocentric view synthesis.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of egocentric images with corresponding camera parameters (position $\mathbf{x}$ and viewing direction $\mathbf{d}$).
*   **Output:** Volume density $\sigma(\mathbf{x})$ and color $c(\mathbf{x}, \mathbf{d})$ for any continuous 3D coordinate and direction.

**Data Structures:**
The representation uses two primary grids, $\mathcal{G}_\sigma$ and $\mathcal{G}_a$, defined on a "balanced spherical grid" with resolution $2N_y^r \times N_y^\theta \times N_y^\phi$.
1.  **Density Grid ($\mathcal{G}_\sigma$):** A 3D tensor of size $2N_y^r \times N_y^\theta \times N_y^\phi$ with a single channel storing explicit volume density values.
2.  **Appearance Grid ($\mathcal{G}_a$):** A 4D tensor of size $2N_y^r \times N_y^\theta \times N_y^\phi \times C$ storing $C$-dimensional neural appearance features.

Both grids are decomposed into "Yin" and "Yang" components ($y \in Y = \{\text{Yin, Yang}\}$), where the final grid is the union of these components: $\mathcal{G} = \bigcup_{y \in Y} \mathcal{G}^y$.

**Tensor Factorization:**
To reduce memory, the feature tensors are decomposed into vectors and matrices using low-rank tensor factorization.

For the density grid $\mathcal{G}_\sigma^y$, the decomposition is:
$$
\mathcal{G}_\sigma^y = \sum_{n=1}^{N_\sigma} \left( \mathbf{v}_{\sigma,n}^{y,R} \otimes \mathbf{M}_{\sigma,n}^{y,\Theta\Phi} + \mathbf{v}_{\sigma,n}^{y,\Theta} \otimes \mathbf{M}_{\sigma,n}^{y,\Phi R} + \mathbf{v}_{\sigma,n}^{y,\Phi} \otimes \mathbf{M}_{\sigma,n}^{y,R\Theta} \right)
$$
where:
*   $N_\sigma$ is the number of factorization terms.
*   $\otimes$ denotes the outer product.
*   $\mathbf{v}$ represents vector factors (e.g., $\mathbf{v}_{\sigma,n}^{y,R}$ is a vector along the radial axis $R$).
*   $\mathbf{M}$ represents matrix factors (e.g., $\mathbf{M}_{\sigma,n}^{y,\Theta\Phi}$ is a matrix spanning the $\Theta$ and $\Phi$ axes).
*   The indices $R, \Theta, \Phi$ correspond to the radial, polar, and azimuthal dimensions of the spherical grid.

For the appearance grid $\mathcal{G}_a^y$, the decomposition is:
$$
\mathcal{G}_a^y = \sum_{n=1}^{N_a} \left( \mathcal{A}_{a,n}^{y,R} \otimes \mathbf{b}_{3n-2}^y + \mathcal{A}_{a,n}^{y,\Theta} \otimes \mathbf{b}_{3n-1}^y + \mathcal{A}_{a,n}^{y,\Phi} \otimes \mathbf{b}_{3n}^y \right)
$$
where:
*   $N_a$ is the number of factorization terms.
*   $\mathcal{A}$ represents matrix factors (e.g., $\mathcal{A}_{a,n}^{y,R}$ spans the $\Theta$ and $\Phi$ axes).
*   $\mathbf{b}$ represents vector factors (e.g., $\mathbf{b}_{3n-2}^y$ is a vector along the radial axis $R$).

**Lookup and Decoding:**
Given a query point $\mathbf{x}$ and direction $\mathbf{d}$:
1.  Perform trilinear interpolation $\mathcal{T}$ on the decomposed grids to retrieve the density and appearance features.
2.  Compute density: $\sigma(\mathbf{x}) = \mathcal{T}(\mathcal{G}_\sigma, \mathbf{x})$.
3.  Compute color: $c(\mathbf{x}, \mathbf{d}) = f_{\text{MLP}}(\mathcal{T}(\mathcal{G}_a, \mathbf{x}), \mathbf{d})$, where $f_{\text{MLP}}$ is a small neural network decoding the appearance features and viewing direction into color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_feature_grid(x, d, grids, mlp):
    """
    Query density and color from the decomposed feature grids.
    
    Args:
        x: 3D coordinate vector (r, theta, phi) in spherical space.
        d: Viewing direction vector.
        grids: Dictionary containing decomposed factors for Yin and Yang components.
               Structure: {
                   'sigma': {
                       'Yin': {'v_R': [...], 'M_ThetaPhi': [...], ...}, # Factors for G_sigma^Yin
                       'Yang': {'v_R': [...], 'M_ThetaPhi': [...], ...} # Factors for G_sigma^Yang
                   },
                   'appearance': {
                       'Yin': {'A_R': [...], 'b_0': [...], ...}, # Factors for G_a^Yin
                       'Yang': {'A_R': [...], 'b_0': [...], ...} # Factors for G_a^Yang
                   }
               }
        mlp: A small neural network function f_MLP(features, direction) -> color.
    
    Returns:
        sigma: Scalar volume density.
        c: Color vector.
    """
    
    # Helper to reconstruct grid value at x from decomposed factors
    def reconstruct_density(x, factors):
        # x = (r, theta, phi)
        # Sum over n=1 to N_sigma
        total = 0
        for n in range(len(factors['v_R'])):
            # Term 1: v_R(n) outer M_ThetaPhi(n)
            # This creates a 3D tensor. We need to evaluate it at x.
            # v_R is size N_r. M_ThetaPhi is size N_theta x N_phi.
            # Value at x is v_R[r] * M_ThetaPhi[theta, phi]
            term1 = factors['v_R'][n][x.r] * factors['M_ThetaPhi'][n][x.theta, x.phi]
            
            # Term 2: v_Theta(n) outer M_PhiR(n)
            # v_Theta is size N_theta. M_PhiR is size N_phi x N_r.
            term2 = factors['v_Theta'][n][x.theta] * factors['M_PhiR'][n][x.phi, x.r]
            
            # Term 3: v_Phi(n) outer M_RTheta(n)
            # v_Phi is size N_phi. M_RTheta is size N_r x N_theta.
            term3 = factors['v_Phi'][n][x.phi] * factors['M_RTheta'][n][x.r, x.theta]
            
            total += term1 + term2 + term3
        return total

    def reconstruct_appearance(x, factors):
        # Sum over n=1 to N_a
        total_features = []
        for n in range(len(factors['A_R'])):
            # Term 1: A_R(n) outer b_0(n)
            # A_R is size N_theta x N_phi. b_0 is size N_r.
            # This contributes to the feature vector.
            # We assume the feature dimension C is distributed or b vectors are scalars 
            # that scale the matrix output, or b vectors are feature vectors.
            # Based on formula: A (matrix) outer b (vector).
            # If A is (N_theta, N_phi) and b is (N_r), the result is (N_r, N_theta, N_phi).
            # We evaluate at x: A[theta, phi] * b[r]
            val = factors['A_R'][n][x.theta, x.phi] * factors['b_0'][n][x.r]
            total_features.append(val)
            
            # Term 2: A_Theta(n) outer b_1(n)
            # A_Theta is (N_r, N_phi). b_1 is (N_theta).
            val = factors['A_Theta'][n][x.r, x.phi] * factors['b_1'][n][x.theta]
            total_features.append(val)
            
            # Term 3: A_Phi(n) outer b_2(n)
            # A_Phi is (N_r, N_theta). b_2 is (N_phi).
            val = factors['A_Phi'][n][x.r, x.theta] * factors['b_2'][n][x.phi]
            total_features.append(val)
            
        # Concatenate or sum? The formula sums the tensors.
        # If C is the feature dim, and b vectors are scalars, we sum the scalar contributions.
        # If b vectors are feature vectors, we sum the feature vectors.
        # The source says "b represents vector factors". 
        # Assuming b vectors are scalars for density, but for appearance, 
        # the formula structure implies b are vectors if A are matrices.
        # However, standard tensor decomposition for appearance usually results in a feature vector.
        # Let's assume the result of the sum is the feature vector at x.
        return sum(total_features)

    # 1. Reconstruct Density
    sigma_yin = reconstruct_density(x, grids['sigma']['Yin'])
    sigma_yang = reconstruct_density(x, grids['sigma']['Yang'])
    sigma = sigma_yin + sigma_yang # Union of Yin and Yang
    
    # 2. Reconstruct Appearance Features
    feat_yin = reconstruct_appearance(x, grids['appearance']['Yin'])
    feat_yang = reconstruct_appearance(x, grids['appearance']['Yang'])
    features = feat_yin + feat_yang
    
    # 3. Decode Color
    c = mlp(features, d)
    
    return sigma, c
```

**Parameters:**
*   $N_y^r, N_y^\theta, N_y^\phi$: Grid resolutions for radial, polar, and azimuthal dimensions.
*   $N_\sigma, N_a$: Number of factorization terms for density and appearance.
*   $C$: Dimension of appearance features.

**Edge Cases:**
*   **Boundary Conditions:** Trilinear interpolation $\mathcal{T}$ requires handling coordinates outside the grid bounds. Standard clamping or periodic boundary conditions should be applied.
*   **Precision:** The decomposition involves summing multiple outer products. Numerical stability should be monitored, especially for large $N_\sigma$ or $N_a$.

## Subtopics

### [Masked Wavelet Representation for Compact Neural Radiance Fields](9619-masked-wavelet-representation-for-compact-neural-radiance-fi.md)
Masked Wavelet Representation (MW-NeRF) reduces the memory footprint of grid-based neural radiance fields by transforming spatial grid coefficients into the wavelet domain, applying a trainable binary mask to induce sparsity, and compressing the result. It sets itself apart from standard grid-based methods like [Feature Grid as Radiance Field](5508-feature-grid-as-radiance-field.md) by leveraging the compactness of wavelet coefficients for smooth signals and per-scene optimized masking, achieving state-of-the-art performance within a 2 MB memory budget.

### [3D Bilateral Grid Slicing](10529-3d-bilateral-grid-slicing.md)
3D Bilateral Grid Slicing is a method for approximating continuous, view-dependent color transformations (such as Image Signal Processing or ISP) in neural radiance fields by storing affine transformation matrices in a 3D grid indexed by spatial coordinates $(x, y)$ and a color guidance coordinate (typically grayscale intensity). It solves the problem of applying non-linear, spatially-varying color corrections to rendered images by learning a mapping from pixel position and color to an affine matrix, avoiding the need for explicit per-pixel parameter networks. It differs from sibling methods like Masked Wavelet Representation by focusing on explicit color-space transformations via grid lookup rather than compacting the underlying radiance field features.

## Operating characteristics
*   **Time Complexity:** Query time is $O(1)$ per sample, dominated by the trilinear interpolation and the small MLP evaluation. The reconstruction from factors adds a constant overhead proportional to the number of factorization terms $N_\sigma$ and $N_a$.
*   **Memory Complexity:** The low-rank tensor factorization reduces space complexity from $O(n^3)$ to $O(n^2)$, where $n$ is the grid resolution. This allows for higher resolutions within the same memory budget compared to dense grids.
*   **Strengths:** Efficient memory usage for high-resolution spherical grids; regular grid structure simplifies implementation and interpolation; suitable for egocentric scenes with spherical symmetry.
*   **Limitations:** The spherical parameterization may not be optimal for non-spherical or planar scenes; the decomposition introduces hyperparameters ($N_\sigma, N_a$) that require tuning.
*   **When to Use:** Egocentric view synthesis where the scene exhibits spherical symmetry and memory is a constraint.
*   **When Not to Use:** Scenes with strong planar structures or non-spherical geometry where hash grids or voxel grids might be more efficient.

## Background
The method was introduced in the paper "Balanced Spherical Grid for Egocentric View Synthesis" (EgoNeRF) [S1]. It addresses the limitations of MLP-based radiance fields in terms of memory and training speed for egocentric data, which often covers a large spherical field of view. By using explicit feature grids with low-rank factorization, it achieves a balance between the flexibility of neural fields and the efficiency of grid-based methods.

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
