# Gabor Noise Surface Filtering

Also called: Analytical filtering of Gabor noise, Gabor Noise Filtering

## Overview
Gabor Noise Surface Filtering is an analytical method for generating anisotropically filtered procedural noise on surfaces, solving the problem of aliasing and visual inconsistency caused by non-uniform perspective distortions during camera or object motion [S1, S3]. Unlike methods that discretize the spectrum into distinct bands or rely on post-hoc blurring, this approach exploits the fact that Gabor kernels are Gaussians in the frequency domain, allowing the filter to be applied by analytically modifying the kernel parameters per pixel [S1, S3]. It builds on [Anisotropic Noise](16450-anisotropic-noise.md) and provides a view-dependent spectral control that adapts to the screen-space projection of the surface [S3].

## How it works
The method generates filtered noise by operating in the frequency domain, leveraging the closed-form nature of Gaussian functions. It avoids explicit convolution in the spatial domain by computing the product of the noise spectrum and the filter spectrum, resulting in a new set of kernel parameters that directly yield the filtered noise.

### Inputs and Outputs
*   **Inputs**:
    *   Surface geometry (position $\mathbf{p}$, normal $\mathbf{n}$, tangent $\mathbf{t}$, bitangent $\mathbf{b}$) for the point being shaded.
    *   Camera parameters (projection matrix, view matrix) to determine screen-space derivatives.
    *   Gabor noise parameters: A set of $N$ oriented subbands. Each subband $i$ is defined by a center frequency vector $\mathbf{k}_i$ (magnitude and orientation) and a bandwidth parameter $\sigma_i$ (width of the Gaussian in frequency space) [S1, S3].
    *   Filter specification: Typically a low-pass filter defined by a cutoff frequency or a Gaussian width in screen space [S3].
*   **Outputs**:
    *   A noise value $N(\mathbf{p})$ that is anisotropically filtered according to the view direction and surface orientation.

### Data Structures
The Gabor noise is represented as a sum of $N$ Gabor kernels. Each kernel $i$ is defined by:
1.  **Center Frequency**: $\mathbf{k}_i = (k_{i,x}, k_{i,y}, k_{i,z})$ in 3D space, often derived from a 2D orientation projected onto the surface tangent plane.
2.  **Bandwidth**: $\sigma_i$, controlling the spread of the Gaussian in the frequency domain.
3.  **Phase**: $\phi_i$, a random offset.

In the frequency domain, the spectrum of a single Gabor kernel is a Gaussian function centered at $\mathbf{k}_i$ with covariance determined by $\sigma_i$.

### Steps
1.  **Compute Screen-Space Derivatives**: For the shading point $\mathbf{p}$, compute the partial derivatives of the position with respect to screen coordinates $(u, v)$. This determines the local scale and orientation of the surface projection in screen space [S3].
2.  **Determine Filter Gaussian**: Calculate the filter Gaussian in the frequency domain. Since the filter is view-dependent, its shape and orientation in texture space vary per pixel. The filter Gaussian $F(\mathbf{u})$ is typically defined by the screen-space derivatives, representing the area covered by a single pixel in the frequency domain [S3].
3.  **Iterate Over Kernels**: For each Gabor kernel $i$ in the noise definition:
    a.  **Identify Gabor Gaussian**: The spectrum of the unfiltered kernel is a Gaussian $G_i(\mathbf{u})$ centered at $\mathbf{k}_i$ with width $\sigma_i$.
    b.  **Compute Product**: Multiply the Gabor Gaussian $G_i(\mathbf{u})$ by the filter Gaussian $F(\mathbf{u})$ in the frequency domain. Since both are Gaussians, their product is another Gaussian $G'_i(\mathbf{u})$ [S1, S2, S3].
    c.  **Derive Filtered Parameters**: Analytically compute the parameters of the resulting Gaussian $G'_i(\mathbf{u})$. This involves combining the means and covariances of the two Gaussians. The new center frequency $\mathbf{k}'_i$ and bandwidth $\sigma'_i$ are derived from the product [S1, S3].
4.  **Generate Filtered Noise**: Use the new parameters $\mathbf{k}'_i$ and $\sigma'_i$ to evaluate the Gabor noise function. The noise value is the sum of the contributions from all filtered kernels:
    $$ N(\mathbf{p}) = \sum_{i=1}^{N} w_i \cdot \text{Gabor}(\mathbf{p}, \mathbf{k}'_i, \sigma'_i, \phi_i) $$
    where $w_i$ are the weights of the subbands [S1, S3].

### Formulas
The spectrum of a Gabor kernel $i$ in 3D frequency space $\mathbf{u}$ is modeled as a Gaussian:
$$ G_i(\mathbf{u}) = \exp\left( -\frac{1}{2} (\mathbf{u} - \mathbf{k}_i)^T \Sigma_i^{-1} (\mathbf{u} - \mathbf{k}_i) \right) $$
where $\Sigma_i$ is the covariance matrix related to $\sigma_i$.

The filter Gaussian $F(\mathbf{u})$ in texture space, derived from screen-space derivatives, is also a Gaussian. The product of two Gaussians $G_i(\mathbf{u})$ and $F(\mathbf{u})$ results in a new Gaussian $G'_i(\mathbf{u})$:
$$ G'_i(\mathbf{u}) = G_i(\mathbf{u}) \cdot F(\mathbf{u}) = C \cdot \exp\left( -\frac{1}{2} (\mathbf{u} - \mathbf{k}'_i)^T \Sigma'_i^{-1} (\mathbf{u} - \mathbf{k}'_i) \right) $$
The new mean $\mathbf{k}'_i$ and covariance $\Sigma'_i$ are computed as:
$$ \Sigma'_i = (\Sigma_i^{-1} + \Sigma_F^{-1})^{-1} $$
$$ \mathbf{k}'_i = \Sigma'_i (\Sigma_i^{-1} \mathbf{k}_i + \Sigma_F^{-1} \mathbf{k}_F) $$
where $\mathbf{k}_F$ and $\Sigma_F$ are the mean and covariance of the filter Gaussian. Typically, the filter is centered at the origin in the differential frequency space, so $\mathbf{k}_F = \mathbf{0}$ [S1, S3].

### Parameters
*   **Number of Kernels ($N$)**: Determines the complexity and anisotropy of the noise. Typical values range from 4 to 16 for real-time applications [S1].
*   **Bandwidth ($\sigma_i$)**: Controls the frequency spread. Smaller $\sigma_i$ results in sharper frequency localization.
*   **Filter Width**: Determined by the screen-space derivatives. Not a fixed parameter but computed per pixel.

### Edge Cases and Pitfalls
*   **View-Dependence**: The filter Gaussian varies per screen pixel. Failing to compute the correct filter shape for each pixel leads to incorrect filtering, especially on tilted surfaces [S3].
*   **Degenerate Input**: If the surface is viewed edge-on or the derivatives are zero, the filter Gaussian may become singular. Guard against division by zero when inverting covariance matrices [G].
*   **Precision**: The analytical combination of Gaussians requires careful handling of numerical precision, especially when $\sigma_i$ and the filter width differ significantly [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gabor_noise_surface_filtering(p, normal, tangent, bitangent, camera, noise_params):
    """
    Generate filtered Gabor noise at point p.
    
    Args:
        p: 3D position vector.
        normal: Surface normal vector.
        tangent: Surface tangent vector.
        bitangent: Surface bitangent vector.
        camera: Camera object with methods for screen-space derivatives.
        noise_params: List of Gabor kernel parameters (k_i, sigma_i, phi_i, w_i).
        
    Returns:
        Filtered noise value.
    """
    # Step 1: Compute screen-space derivatives
    # This determines how the surface maps to screen space
    dp_du, dp_dv = camera.compute_screen_derivatives(p, normal, tangent, bitangent)
    
    # Step 2: Determine Filter Gaussian parameters
    # The filter is a Gaussian in frequency space defined by the pixel footprint
    # Compute the covariance matrix of the filter based on dp_du and dp_dv
    # This is view-dependent and varies per pixel
    sigma_F = compute_filter_covariance(dp_du, dp_dv)
    k_F = [0, 0, 0] # Filter is centered at origin in differential space
    
    filtered_noise = 0.0
    
    # Step 3: Iterate over kernels
    for kernel in noise_params:
        k_i = kernel.center_frequency
        sigma_i = kernel.bandwidth
        phi_i = kernel.phase
        w_i = kernel.weight
        
        # Step 3a: Identify Gabor Gaussian parameters
        # Sigma_i is derived from sigma_i (bandwidth)
        Sigma_i = compute_covariance_from_bandwidth(sigma_i)
        
        # Step 3b: Compute Product of Gaussians
        # Inverse covariances
        Sigma_i_inv = invert_matrix(Sigma_i)
        Sigma_F_inv = invert_matrix(sigma_F)
        
        # New covariance
        Sigma_prime_inv = add_matrices(Sigma_i_inv, Sigma_F_inv)
        Sigma_prime = invert_matrix(Sigma_prime_inv)
        
        # New mean
        k_prime = multiply_matrix_vector(Sigma_prime, 
                                         add_matrices(
                                             multiply_matrix_vector(Sigma_i_inv, k_i),
                                             multiply_matrix_vector(Sigma_F_inv, k_F)
                                         ))
        
        # Step 3c: Generate Filtered Noise
        # Evaluate the Gabor kernel with new parameters
        # The Gabor function is typically: cos(2*pi * k . p + phi) * exp(-pi * sigma^2 * ||p||^2)
        # However, since we are in frequency domain, we use the spatial domain representation
        # with the modified frequency and bandwidth.
        
        # Note: The exact spatial domain formula for Gabor noise depends on the specific
        # implementation of the base Gabor noise function. Here we assume a standard form.
        # The key is that k_prime and sigma_prime are used instead of k_i and sigma_i.
        
        noise_val = evaluate_gabor_kernel(p, k_prime, sigma_prime, phi_i)
        filtered_noise += w_i * noise_val
        
    return filtered_noise

def compute_filter_covariance(dp_du, dp_dv):
    """
    Compute the covariance matrix of the filter Gaussian based on screen-space derivatives.
    This represents the area covered by a pixel in the frequency domain.
    """
    # The filter covariance is related to the inverse of the Jacobian of the projection
    # Simplified: The filter width is proportional to the magnitude of the derivatives
    # In practice, this involves computing the local frequency response
    # For a 2D surface in 3D, the filter is anisotropic
    # This is a placeholder for the specific derivation from screen derivatives
    # The exact formula depends on the camera projection model
    # Typically, Sigma_F is computed from the partial derivatives of the texture coordinates
    # with respect to screen coordinates.
    
    # Let's assume we have texture coordinates u, v
    # du/dx, du/dy, dv/dx, dv/dy are the screen derivatives
    # The filter covariance in frequency space is related to these derivatives
    
    # For the purpose of this pseudocode, we assume a helper function exists
    # that computes the 2x2 covariance matrix in the tangent plane
    # projected from screen space.
    
    # This is a critical step that varies per pixel
    # The filter is a Gaussian in the frequency domain
    # Its width is determined by the pixel footprint on the surface
    
    # Return a 3x3 covariance matrix in 3D space, aligned with the surface
    # For simplicity, we assume the filter is isotropic in the tangent plane
    # and zero in the normal direction
    
    # Compute the scale of the pixel on the surface
    scale_u = length(dp_du)
    scale_v = length(dp_dv)
    
    # The filter bandwidth is inversely proportional to the scale
    # Sigma_F is a 3x3 matrix
    Sigma_F = zeros(3, 3)
    
    # Align with tangent and bitangent
    # This is a simplification; the actual derivation involves the Jacobian
    # of the projection from 3D to 2D screen space
    
    # For a more accurate implementation, one would compute the
    # frequency domain filter as the Fourier transform of the pixel shape
    # projected onto the surface.
    
    # Return the computed covariance
    return Sigma_F

def evaluate_gabor_kernel(p, k, sigma, phi):
    """
    Evaluate a single Gabor kernel at point p.
    """
    # Gabor noise is typically defined as:
    # N(p) = cos(2 * pi * dot(k, p) + phi) * exp(-pi * sigma^2 * dot(p, p))
    # However, for surface filtering, the frequency k is in 3D space
    # and the Gaussian envelope is also in 3D space.
    
    # The exact formula depends on the specific Gabor noise implementation
    # The key is that k and sigma are the filtered parameters
    
    dot_product = dot(k, p)
    phase = 2 * pi * dot_product + phi
    envelope = exp(-pi * sigma**2 * dot(p, p))
    
    return cos(phase) * envelope
```

## Subtopics

### [NPR Gabor Noise](16655-npr-gabor-noise.md)
NPR Gabor Noise is a variant of Gabor noise designed for coherent stylization in non-photorealistic rendering (NPR). It solves the problem of maintaining visual consistency of noise patterns during camera or object motion by decoupling the noise parameter definition from the point sampling space. Unlike standard Gabor noise where parameters are often tied to object space, this method defines noise parameters in 2D screen space while defining point distribution in 3D object space, thereby preserving the 2D aspect of the noise and the 3D motion of the scene. It builds on [Gabor Noise Surface Filtering](16556-gabor-noise-surface-filtering.md).

## Operating characteristics
*   **Time Complexity**: The method requires computing screen-space derivatives and then iterating over all Gabor kernels. For each kernel, it performs matrix inversions and multiplications to update the parameters. The complexity is $O(N)$ per pixel, where $N$ is the number of kernels. The matrix operations are constant time (3x3 matrices).
*   **Memory Complexity**: Minimal additional memory is required, as the filtering is done analytically by modifying parameters. The noise function itself requires storing the kernel parameters.
*   **Measured Performance**: Performance depends on the number of kernels and the cost of evaluating the Gabor function. Analytical filtering avoids the need for multi-sample anti-aliasing or post-processing blurs, which can be expensive.
*   **Scaling**: The method scales well with resolution, as the filtering is done per pixel based on screen-space derivatives. It adapts to non-uniform perspective distortions without requiring scene-dependent precomputation.
*   **Strengths**:
    *   Analytical filtering avoids discretization artifacts.
    *   Adapts to view-dependent perspective distortions.
    *   Maintains anisotropy correctly on tilted surfaces.
*   **Limitations**:
    *   Requires accurate screen-space derivatives.
    *   Computationally more expensive per pixel than unfiltered noise due to the parameter updates.
    *   May struggle with very high-frequency noise if the filter width is too small, leading to numerical instability [G].
*   **When to Use**: Use when high-quality anisotropic filtering is required for procedural noise on surfaces, especially in real-time rendering where post-processing blurs are not feasible or when perspective distortions are significant.
*   **When Not to Use**: Avoid if the noise is isotropic and the surface is viewed front-on, as simpler filtering methods may suffice. Also avoid if the computational budget is extremely tight and the number of kernels is large.

## Background
Gabor Noise was introduced by Goldberg et al. [GZD08] in 2008 as a noise function that supports high-quality anisotropic filtering by tiling the frequency domain into oriented subbands. The analytical filtering method for Gabor noise was described by Lagae et al. [LLDD09a] in 2009, exploiting the property that Gabor kernels correspond to Gaussians in the frequency domain. This allows for direct generation of filtered noise by modifying kernel parameters, avoiding the need for explicit convolution or discretization of the spectrum. The method is part of the broader field of procedural noise functions, which are widely used in computer graphics for texture synthesis, terrain generation, and visual effects.

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S3] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
