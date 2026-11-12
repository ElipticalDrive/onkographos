# NRFF

Also called: Neural Radiance Feature Field

## Overview
NRFF is a radiance field optimization method that improves upon Ref-NeRF by replacing standard positional encoding and view-dependent modeling with multiscale tensor decomposition (MTD) and Anisotropic Spherical Gaussians (ASG) [S1]. It distinguishes itself by predicting ASG features in the spatial network and integrating specular contributions via a custom Rendering Equation Encoding (REE) in the latent space, rather than relying on roughness maps and image-derived environment maps (IDE) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D position $\mathbf{x}$, view direction $\mathbf{d}$, and camera parameters.
*   **Output:** Volume density $\sigma$ and color $\mathbf{c}$, rendered via standard volumetric integration.

**Data Structures:**
1.  **Spatial MLP Output:** A vector of 128 values representing $N = 8 \times 16$ lobes of Anisotropic Spherical Gaussians (ASGs) on a unit sphere [S1].
2.  **ASG Bottleneck:** A vector $\mathbf{a}$ of length 2, and two scalar coefficients $\lambda, \mu$ [S1].
3.  **REE Vector:** An $8 \times 16 \times 2$ vector representing the integrated specular contribution in the latent space [S1].

**Algorithm Steps:**
1.  **Spatial Feature Extraction:** The spatial MLP processes the 3D position $\mathbf{x}$ using Multiscale Tensor Decomposition (MTD) instead of Positional Encoding (PE) to represent the scene in a coarse-to-fine scheme [S1].
2.  **ASG Prediction:** The spatial MLP predicts features for $N = 8 \times 16$ ASG lobes, resulting in a 128-dimensional vector [S1].
3.  **ASG Bottleneck Generation:** An auxiliary ASG MLP takes the 128 ASG features and converts them into:
    *   An anisotropic reflections bottleneck vector $\mathbf{a}$ of length 2.
    *   Two scalar coefficients $\lambda$ and $\mu$ modeling the shape of the ASG in orthogonal directions [S1].
4.  **Rendering Equation Encoding (REE):** Compute the REE vector at point $\mathbf{x}$ for a reflection direction $\boldsymbol{\omega}_r$ using the formula:
    $$
    \text{REE}(\boldsymbol{\omega}_r, \mathbf{a}, \lambda, \mu) = \sum_{i=0}^{N-1} a_i S(\boldsymbol{\omega}_r; \boldsymbol{\omega}_i) \exp\left(-\lambda_i (\boldsymbol{\omega}_r \cdot \boldsymbol{\omega}_{\lambda_i})^2 - \mu_i (\boldsymbol{\omega}_r \cdot \boldsymbol{\omega}_{\mu_i})^2\right)
    $$
    where:
    *   $N = 8 \times 16$ is the number of lobes [S1].
    *   $a_i$ is the bottleneck value of the $i$-th ASG [S1].
    *   $S(\boldsymbol{\omega}_r; \boldsymbol{\omega}_i) = \max(\boldsymbol{\omega}_r \cdot \boldsymbol{\omega}_i, 0)$ is a smooth term [S1].
    *   $\lambda_i, \mu_i > 0$ are bandwidth coefficients related to orthogonal axes $\boldsymbol{\omega}_{\lambda_i}, \boldsymbol{\omega}_{\mu_i}$ [S1].
    *   The set of directions $\boldsymbol{\omega}_i$ covers 360 degrees (the unit sphere) [S1].
5.  **Directional Processing:** The directional MLP ingests the $8 \times 16 \times 2$ REE vector to predict color and density components, replacing the roughness and IDE inputs used in Ref-NeRF [S1].
6.  **Loss Function:** The total loss includes the standard Ref-NeRF auxiliary term $L_o$ and a new regularization term $L_f$:
    $$
    L_f = \frac{\beta}{M} \sum_{i=0}^{M-1} \frac{|F_\tau^i|}{\tau}
    $$
    where $M$ is the number of features in the density field representation, $F_\tau$ is the density field feature map, and $\beta$ is a coefficient [S1]. This term encourages a sparse feature representation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nrff_forward(x, d, spatial_mlp, asg_mlp, directional_mlp, mtd_encoder):
    """
    Forward pass for NRFF.
    x: 3D position vector
    d: View direction vector
    """
    # Step 1: Spatial feature extraction using MTD
    # MTD replaces standard positional encoding
    spatial_features = spatial_mlp(mtd_encoder(x))
    
    # Step 2: Predict ASG features (128 values for 8x16 lobes)
    asg_features = spatial_features # 128-dimensional vector
    
    # Step 3: Generate ASG bottleneck and coefficients
    # Auxiliary ASG MLP converts 128 features into bottleneck and scalars
    a, lambda_coeffs, mu_coeffs = asg_mlp(asg_features)
    # a: length 2 vector
    # lambda_coeffs, mu_coeffs: scalars or vectors of length N depending on implementation details not fully specified
    # Note: Source implies lambda and mu model shape in orthogonal directions
    
    # Step 4: Compute REE Vector
    # N = 8 * 16 = 128 lobes
    N = 128
    ree_vector = []
    
    # Precompute ASG axes and directions if not provided by MLP
    # The source states "The set of omega_i covers 360 degrees"
    # Implementation requires defining the 128 directions omega_i and orthogonal axes
    # This is a placeholder for the specific geometric arrangement of the 128 lobes
    for i in range(N):
        # Placeholder for specific ASG geometry definition
        omega_i = get_asg_direction(i) 
        omega_lambda_i = get_asg_orthogonal_axis_lambda(i)
        omega_mu_i = get_asg_orthogonal_axis_mu(i)
        
        # Calculate smooth term S
        # S(omega_r; omega_i) = max(omega_r . omega_i, 0)
        # Note: The REE is computed for a reflection direction omega_r
        # In the context of the directional MLP, omega_r is derived from d and surface normal
        # However, the source describes REE as operating in latent space
        # We assume omega_r is passed or computed from d
        
        # Calculate exponential term
        # exp(-lambda_i * (omega_r . omega_lambda_i)^2 - mu_i * (omega_r . omega_mu_i)^2)
        # Note: lambda_i and mu_i are bandwidth coefficients. 
        # The source formula uses lambda_i and mu_i inside the sum, 
        # but the bottleneck output is a vector a of length 2.
        # This implies lambda and mu might be shared or derived from a.
        # Given the ambiguity, we use the coefficients from the ASG MLP output.
        
        # Re-reading source: "converts them into an anisotropic reflections bottleneck vector a of length 2, 
        # as well as 2 scalar coefficients, lambda and mu"
        # This suggests lambda and mu are global scalars for the point, not per-lobe.
        # However, the formula sums over i with lambda_i and mu_i.
        # This is a contradiction or simplification in the source.
        # We will assume lambda_i and mu_i are derived from the global lambda, mu or are per-lobe parameters 
        # not explicitly detailed in the bottleneck size.
        # Given "a of length 2", it's likely lambda and mu are the 2 values.
        # But the sum index i suggests per-lobe parameters.
        # Let's assume the source formula implies per-lobe bandwidths, 
        # but the bottleneck only stores 2 values. This is a gap.
        # [G] Assuming lambda and mu are broadcast or the bottleneck a contains more info than stated, 
        # or the formula is simplified. We will use the scalar lambda, mu for all i for consistency with "2 scalar coefficients".
        
        dot_lambda = dot(omega_r, omega_lambda_i)
        dot_mu = dot(omega_r, omega_mu_i)
        
        term = a[i] * max(dot(omega_r, omega_i), 0) * \
               exp(-lambda * dot_lambda**2 - mu * dot_mu**2)
        ree_vector.append(term)
        
    # Reshape to 8x16x2 if necessary for directional MLP
    ree_vector = reshape(ree_vector, (8, 16, 2))
    
    # Step 5: Directional MLP
    # Ingests REE vector
    output = directional_mlp(ree_vector)
    
    return output
```

**Parameters:**
*   $N$: Number of ASG lobes, set to $8 \times 16$ [S1].
*   $\mathbf{a}$: Anisotropic reflections bottleneck vector, length 2 [S1].
*   $\lambda, \mu$: Scalar coefficients modeling the shape of the ASG in orthogonal directions, must be $> 0$ [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** NRFF has shown proficiency in encoding view-dependent effects, enabling superior rendering quality on diverse datasets compared to Ref-NeRF [S1].
*   **Strengths:** The sparse feature representation enhanced by $L_f$ improves model interpretability and mitigates overfitting by removing less relevant or redundant information [S1]. MTD provides efficiency in reconstructing scene geometry and appearance [S1].
*   **Limitations:** Not given in the sources.

## Background
NRFF was proposed by Albert Barreiro, Roger Marı́, Rafael Redondo, Gloria Haro, Carles Bosch, and David Berga from Eurecat and Universitat Pompeu Fabra [S1]. The method was submitted on 2024–07–09, accepted on 2025–01–17, and published in *Image Processing On Line* on 2025–03–12 [S1]. It builds on the Ref-NeRF architecture but introduces architectural improvements to better handle specularities and view-dependent effects [S1].

## Sources
[S1] Specularity in NeRFs: A Comparative Study of Ref-NeRF and NRFF, https://doi.org/10.5201/ipol.2025.562, fetched 2026-10-26
