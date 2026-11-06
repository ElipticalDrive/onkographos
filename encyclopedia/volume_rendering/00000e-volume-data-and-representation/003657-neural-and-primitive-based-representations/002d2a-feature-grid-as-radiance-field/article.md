# Feature Grid as Radiance Field

Also called: EgoNeRF representation, Tensor Factorization

## Overview
This method represents a radiance field using explicit, low-rank feature grids for density and appearance, decomposed via tensor factorization to reduce memory complexity from cubic to quadratic relative to resolution [S1]. It differs from standard MLP-based fields by storing features in structured grids and from pure hash grids by using balanced spherical parameterization for egocentric scenes [S1].

## How it works
**Inputs:** A set of egocentric images with corresponding camera parameters.
**Outputs:** Volume density $\sigma(\mathbf{x})$ and color $c(\mathbf{x}, \mathbf{d})$ for any 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$.

**Data Structures:**
The representation relies on two primary grids, $\mathcal{G}_\sigma$ and $\mathcal{G}_a$, composed of "balanced spherical grids" [S1].
1.  **Density Grid ($\mathcal{G}_\sigma$):** Stores explicit volume density. It is a union of grids for "Yin" and "Yang" hemispheres: $\mathcal{G}_\sigma = \bigcup_{y \in Y} \mathcal{G}_\sigma^y$, where $Y = \{\text{Yin, Yang}\}$ [S1].
2.  **Appearance Grid ($\mathcal{G}_a$):** Stores $C$-dimensional neural appearance features. It is similarly a union: $\mathcal{G}_a = \bigcup_{y \in Y} \mathcal{G}_a^y$ [S1].

Each hemisphere grid $\mathcal{G}^y$ is decomposed into low-rank tensor factors (vectors and matrices) to avoid storing the full 3D tensor [S1].

**Parameters:**
*   $N_y$: Resolution parameter for the spherical grid. The grid dimensions are $2N_y^r \times N_y^\theta \times N_y^\phi$ [S1].
*   $N_\sigma$: Number of factor components for density [S1].
*   $N_a$: Number of factor components for appearance [S1].
*   $C$: Dimension of neural appearance features [S1].

**Algorithm:**
1.  **Grid Decomposition:** For each hemisphere $y \in \{\text{Yin, Yang}\}$, decompose the density and appearance tensors into outer products of vectors and matrices [S1].
    *   Density decomposition:
        $$ \mathcal{G}_\sigma^y = \sum_{n=1}^{N_\sigma} \mathbf{v}_{\sigma,n}^{y,R} \otimes \mathbf{M}_{\sigma,n}^{y,\Theta\Phi} + \mathbf{v}_{\sigma,n}^{y,\Theta} \otimes \mathbf{M}_{\sigma,n}^{y,\Phi R} + \mathbf{v}_{\sigma,n}^{y,\Phi} \otimes \mathbf{M}_{\sigma,n}^{y,R\Theta} $$
        where $\otimes$ is the outer product, and $\mathbf{v}$ are vectors, $\mathbf{M}$ are matrices [S1].
    *   Appearance decomposition:
        $$ \mathcal{G}_a^y = \sum_{n=1}^{N_a} \mathcal{A}_{a,n}^{y,R} \otimes \mathbf{b}_{3n-2}^y + \mathcal{A}_{a,n}^{y,\Theta} \otimes \mathbf{b}_{3n-1}^y + \mathcal{A}_{a,n}^{y,\Phi} \otimes \mathbf{b}_{3n}^y $$
        where $\mathcal{A}$ are vectors/matrices and $\mathbf{b}$ are vectors [S1].
2.  **Feature Retrieval:** Given a query point $\mathbf{x}$, determine its spherical coordinates $(r, \theta, \phi)$ and map them to the appropriate hemisphere $y$ (Yin or Yang) [S1].
3.  **Interpolation:** Perform trilinear interpolation $\mathcal{T}$ on the decomposed factors to retrieve the feature vector for density and appearance [S1].
    *   Density feature: $f_\sigma = \mathcal{T}(\mathcal{G}_\sigma, \mathbf{x})$
    *   Appearance feature: $f_a = \mathcal{T}(\mathcal{G}_a, \mathbf{x})$
4.  **Decoding:**
    *   Density is the interpolated value: $\sigma(\mathbf{x}) = f_\sigma$ [S1].
    *   Color is decoded by a tiny MLP: $c(\mathbf{x}, \mathbf{d}) = f_{\text{MLP}}(f_a, \mathbf{d})$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_radiance_field(x, d, grids, mlp):
    """
    Query density and color for position x and direction d.
    grids: dict containing 'sigma' and 'appearance' factors for Yin and Yang.
    mlp: Tiny MLP function decoding appearance features and direction to color.
    """
    # 1. Map 3D position to spherical coordinates and hemisphere
    r, theta, phi = to_spherical(x)
    hemisphere = 'Yin' if is_yin(theta, phi) else 'Yang'
    
    # 2. Retrieve density feature via trilinear interpolation on decomposed grid
    # Note: In practice, this involves interpolating the vector/matrix factors
    # and reconstructing the value, or interpolating the pre-computed dense grid
    # if memory allows. The source defines the decomposition for storage efficiency.
    sigma = trilinear_interpolate(grids['sigma'][hemisphere], r, theta, phi)
    
    # 3. Retrieve appearance feature
    appearance_feat = trilinear_interpolate(grids['appearance'][hemisphere], r, theta, phi)
    
    # 4. Decode color
    color = mlp(appearance_feat, d)
    
    return sigma, color
```

## Operating characteristics
*   **Space Complexity:** The low-rank tensor factorization reduces space complexity from $O(n^3)$ to $O(n^2)$, where $n$ is the resolution [S1].
*   **Strengths:** Maintains regular angular components while factorizing the grid using spherical parameterization, suitable for egocentric scenes [S1].
*   **Limitations:** Specific limitations beyond complexity are not detailed in the provided sources.

## Background
The method was introduced in "Balanced Spherical Grid for Egocentric View Synthesis" (EgoNeRF) [S1]. It addresses the challenge of reconstructing 3D scene representations from egocentric images by replacing MLP regression with explicit feature grids [S1].

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
