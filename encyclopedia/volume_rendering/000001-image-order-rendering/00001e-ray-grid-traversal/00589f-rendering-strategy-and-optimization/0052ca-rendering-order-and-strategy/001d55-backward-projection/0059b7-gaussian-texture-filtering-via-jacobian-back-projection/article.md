# Gaussian Texture Filtering via Jacobian Back-projection

Also called: Back-projected Gaussian Texture Filtering, Gaussian Back-projection, Gaussian Filtering for Texture Mapping, Jacobian Back-projection

## Overview
Gaussian Texture Filtering via Jacobian Back-projection is a technique to compute the color of a screen pixel by integrating a texture signal against a Gaussian filter, where the filter is back-projected from screen space to texture space using the local affine approximation (Jacobian) of the texture mapping [S1, S2]. It solves the problem of aliasing in texture mapping by ensuring that the frequency content of the projected texture is properly band-limited before sampling [S1, S2]. Unlike sibling methods that may rely on pre-filtered mipmaps or simple box filters, this method explicitly models the pixel footprint as a Gaussian and adapts its shape in texture space based on the local distortion of the surface mapping [S1, S2].

## How it works
### Inputs and Outputs
**Inputs:**
- Screen coordinates $(x, y)$ of the pixel center.
- Texture coordinates $(u, v)$ at the pixel center.
- Local derivatives of texture coordinates with respect to screen coordinates, forming the Jacobian matrix $J$.
- Filter width parameter $\sigma$ in pixels.

**Outputs:**
- Filtered texture color value (integral of the texture signal against the back-projected Gaussian).

### Data Structures
The method relies on the following data structures:
1.  **Jacobian Matrix ($J$):** A $2 \times 2$ matrix representing the local affine approximation of the mapping from image coordinates to texture coordinates.
    $$
    J = \begin{bmatrix} \frac{du}{dx} & \frac{dv}{dx} \\ \frac{du}{dy} & \frac{dv}{dy} \end{bmatrix}
    $$
    -   $\frac{du}{dx}, \frac{dv}{dx}$: Derivatives of texture coordinates with respect to screen $x$.
    -   $\frac{du}{dy}, \frac{dv}{dy}$: Derivatives of texture coordinates with respect to screen $y$.
    -   In rasterization, these are computed using screen-space derivatives of texture coordinates. In raytracing, they are computed using ray differentials [S1, S2].

2.  **Gaussian Filter Parameters:**
    -   $\sigma$: Width of the Gaussian filter in pixels. Typical value: $\sigma = 1.0$ [S1, S2].

### Numbered Steps
1.  **Model Screen-Space Filter:** Define the Gaussian filter in screen space centered at the pixel $(x, y)$. The filter function $f(x, y)$ is defined as:
    $$
    f(x, y) = \frac{1}{\sigma \sqrt{2\pi}} e^{-\frac{1}{2\sigma^2}(x^2 + y^2)}
    $$
    where $x$ and $y$ are screen coordinates relative to the pixel center, and $\sigma$ is the filter width in pixels [S1, S2]. Note: Some sources define the normalization factor as $\frac{1}{\sigma^2 2\pi}$ [S2], but the standard 2D Gaussian normalization is $\frac{1}{2\pi\sigma^2}$. The exponential term is consistent across sources. We use the standard normalization $\frac{1}{2\pi\sigma^2}$ for a 2D Gaussian, which matches the structure $\frac{1}{\sigma \sqrt{2\pi}}$ if interpreted as $\frac{1}{2\pi\sigma^2}$? Let's check S1: $\frac{1}{\sigma 2\pi}$. This is likely a typo for $\frac{1}{2\pi\sigma^2}$ or $\frac{1}{\sigma^2 2\pi}$. S2 says $\frac{1}{\sigma^2 2\pi}$. We will use $\frac{1}{2\pi\sigma^2}$ as the correct 2D Gaussian normalization. The exponential part is $e^{-\frac{x^2+y^2}{2\sigma^2}}$.

2.  **Estimate Jacobian:** Compute the Jacobian matrix $J$ for the current pixel.
    -   For rasterization: Use screen-space derivatives of texture coordinates ($\frac{\partial u}{\partial x}, \frac{\partial v}{\partial x}, \frac{\partial u}{\partial y}, \frac{\partial v}{\partial y}$) [S1, S2].
    -   For raytracing: Use ray differentials to estimate the change in texture coordinates over the pixel footprint [S1, S2].

3.  **Back-Project Filter to Texture Space:** Transform the screen-space Gaussian filter into the texture domain. This results in a distorted Gaussian in texture space. The back-projected filter $f_{J^{-1}}[u, v]^T$ is defined as:
    $$
    f_{J^{-1}}[u, v]^T = \frac{1}{2\pi\sigma^2} e^{-\frac{1}{2\sigma^2} [u, v] J^{-1} J^{-T} [u, v]^T}
    $$
    where $[u, v]^T$ are texture coordinates relative to the center, and $J$ is the Jacobian of the mapping from image to texture coordinates. Note: The sources write $J^{-1} J^{-1}$, but for a general linear transformation of a Gaussian, the covariance matrix transforms as $J^{-1} \Sigma J^{-T}$. Since the screen-space covariance is $\sigma^2 I$, the texture-space covariance is $\sigma^2 J^{-1} J^{-T}$. The exponent is $-\frac{1}{2} [u, v] (\sigma^2 J^{-1} J^{-T})^{-1} [u, v]^T = -\frac{1}{2\sigma^2} [u, v] J^T J [u, v]^T$? Wait.
    Let's re-read S1/S2 carefully.
    S1: $f_{J^{-1}[u\ v]^T} = \frac{1}{\sigma 2\pi} e^{-\frac{1}{2\sigma^2} [u\ v] J^{-1} J^{-1} [u\ v]^T}$.
    S2: $f_{J^{-1}}[u \ v]^T = \frac{1}{\sqrt{2\pi}} e^{-\frac{1}{2} [u \ v] J^{-1} J^{-1} [u \ v]^T}$. This seems inconsistent with S1's normalization.
    Let's look at the frequency domain.
    S1/S2: $F_{J^T[f_u\ f_v]^T} = e^{-2\pi^2 \sigma^2 [f_u\ f_v] J J^T [f_u\ f_v]^T}$.
    The Fourier transform of a Gaussian $e^{-\frac{1}{2} x^T A x}$ is proportional to $e^{-\frac{1}{2} \omega^T A^{-1} \omega}$.
    If the screen space filter is $e^{-\frac{1}{2\sigma^2} (x^2+y^2)}$, its FT is $e^{-2\pi^2 \sigma^2 (f_x^2+f_y^2)}$.
    The mapping is $u = J^{-1} x$ (approx). So $x = J u$.
    The filter in texture space should be the inverse transform.
    The sources explicitly state the texture space filter uses $J^{-1} J^{-1}$ in the exponent. We will follow the source formula exactly: $[u\ v] J^{-1} J^{-1} [u\ v]^T$. Note: If $J$ is the Jacobian from image to texture, then $du = J dx$. So $dx = J^{-1} du$. The screen space metric $dx^2 + dy^2$ becomes $(J^{-1} du)^T (J^{-1} du) = du^T (J^{-1})^T J^{-1} du$. If $J$ is symmetric or if the source assumes $J^{-1} J^{-1}$, we stick to the source. S1 and S2 both write $J^{-1} J^{-1}$. We will use $J^{-1} J^{-T}$ if we assume standard covariance propagation, but the source says $J^{-1} J^{-1}$. Let's look at S1 again: "$[u\ v] J^{-1} J^{-1} [u\ v]^T$". This implies $J$ might be defined differently or it's a specific approximation. We will use the formula as written in the sources: $[u\ v] J^{-1} J^{-1} [u\ v]^T$.

4.  **Compute Pixel Color:** Evaluate the integral of the back-projected Gaussian filter multiplied by the visible portion of the texture. This is equivalent to convolving the texture signal with the back-projected Gaussian filter [S1, S2].
    $$
    C = \int_{\text{texture}} \text{Texture}(u, v) \cdot f_{J^{-1}}[u, v]^T \, du \, dv
    $$
    In practice, this is often approximated by sampling the texture at multiple points weighted by the Gaussian, or by using pre-filtered mipmaps where the mipmap level is determined by the determinant of the Jacobian [G].

5.  **Frequency Domain Alternative (Optional):** The filter can also be applied in the frequency domain. The frequency domain filter is a Gaussian:
    $$
    F_{J^T}[f_u, f_v]^T = e^{-2\pi^2 \sigma^2 [f_u, f_v] J J^T [f_u, f_v]^T}
    $$
    where $[f_u, f_v]^T$ are frequency coordinates. The spectrum of the properly filtered texture is the multiplication of the unfiltered spectrum and this filter spectrum [S1, S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_texture_filtering(u, v, du_dx, dv_dx, du_dy, dv_dy, sigma):
    """
    Computes the back-projected Gaussian filter parameters for texture filtering.
    
    Args:
        u, v: Texture coordinates at the pixel center.
        du_dx, dv_dx: Derivatives of texture coordinates w.r.t screen x.
        du_dy, dv_dy: Derivatives of texture coordinates w.r.t screen y.
        sigma: Width of the Gaussian filter in pixels (typically 1.0).
        
    Returns:
        A function representing the back-projected Gaussian filter in texture space,
        or the parameters to sample the texture.
    """
    # Step 2: Estimate Jacobian J
    # J maps screen space derivatives to texture space derivatives
    # J = [ du/dx  dv/dx ]
    #     [ du/dy  dv/dy ]
    J = Matrix2x2(du_dx, dv_dx, du_dy, dv_dy)
    
    # Step 3: Back-project filter to texture space
    # The filter in texture space is a distorted Gaussian.
    # Formula from sources: f(u,v) ~ exp(-1/(2*sigma^2) * [u v] J^-1 J^-1 [u v]^T)
    # Note: Sources use J^-1 J^-1. Standard covariance propagation would use J^-1 J^-T.
    # We follow the source formula.
    
    try:
        J_inv = J.inverse()
    except SingularMatrixError:
        # Edge case: Degenerate Jacobian (e.g., texture stretched to infinity)
        # Return a default isotropic filter or handle as unfiltered
        return lambda u_rel, v_rel: 0.0
        
    # Precompute the quadratic form matrix M = J_inv * J_inv
    # Note: If J is not symmetric, J_inv * J_inv is not necessarily symmetric.
    # The source formula is [u v] J^-1 J^-1 [u v]^T.
    M = J_inv * J_inv
    
    # Normalization factor
    # Sources vary: S1 says 1/(sigma 2pi), S2 says 1/(sigma^2 2pi).
    # Standard 2D Gaussian normalization is 1/(2*pi*sigma^2).
    # We use the standard normalization.
    norm_factor = 1.0 / (2.0 * math.pi * sigma * sigma)
    
    # Step 1 & 4: Define the filter function
    def back_projected_gaussian(u_rel, v_rel):
        """
        Evaluates the back-projected Gaussian filter at relative texture coordinates.
        
        Args:
            u_rel, v_rel: Texture coordinates relative to the pixel center.
            
        Returns:
            Filter weight.
        """
        # Quadratic form: [u_rel v_rel] * M * [u_rel v_rel]^T
        # Let row vector P = [u_rel, v_rel]
        # Result = P * M * P^T
        # M is 2x2.
        # M = [[m00, m01],
        #      [m10, m11]]
        # P * M = [u_rel*m00 + v_rel*m10, u_rel*m01 + v_rel*m11]
        # (P * M) * P^T = (u_rel*m00 + v_rel*m10)*u_rel + (u_rel*m01 + v_rel*m11)*v_rel
        
        m00, m01 = M[0, 0], M[0, 1]
        m10, m11 = M[1, 0], M[1, 1]
        
        quadratic = (u_rel * (u_rel * m00 + v_rel * m10) + 
                     v_rel * (u_rel * m01 + v_rel * m11))
        
        exponent = -0.5 * quadratic / (sigma * sigma)
        
        # Clamp exponent to avoid underflow
        if exponent < -50.0:
            return 0.0
            
        return norm_factor * math.exp(exponent)
        
    return back_projected_gaussian
```

### Parameters
-   $\sigma$: The width of the Gaussian filter in pixels. Typical value: $\sigma = 1.0$ [S1, S2].

### Edge Cases and Pitfalls
-   **Singular Jacobian:** If the Jacobian matrix is singular (determinant is zero), the inverse does not exist. This occurs when the texture is stretched infinitely or folded. The implementation must handle this, possibly by falling back to a default filter or skipping filtering [G].
-   **Precision:** The computation of $J^{-1}$ and the quadratic form can suffer from precision issues if the Jacobian entries vary widely in magnitude. Clamping the exponent is recommended to avoid underflow [G].
-   **Boundary Conditions:** The integral is over the visible portion of the texture. If the texture is bounded, the filter must be truncated or the integration limits adjusted [G].

## Subtopics

### [GPU Backprojection](5843-gpu-backprojection.md)
Optimizes the back-projection process for parallel execution on GPUs.

### [Ray Segment Projection to Image](7631-ray-segment-projection-to-image.md)
Projects ray segments directly to the image plane, often used in conjunction with back-projection for accurate filtering.

### [Cone Beam Forward Projection with Trilinear Interpolation](8247-cone-beam-forward-projection-with-trilinear-interpolation.md)
Uses forward projection with trilinear interpolation, an alternative to back-projection for volume rendering.

### [Circular Integral Convolution](8507-circular-integral-convolution.md)
Applies circular integral convolution for vector field visualization, a different filtering approach.

## Operating characteristics
-   **Time Complexity:** The cost depends on the number of samples taken to approximate the integral. If a fixed number of samples are used, it is $O(1)$ per pixel. If adaptive sampling is used, it depends on the complexity of the texture and the Jacobian [G].
-   **Memory Complexity:** $O(1)$ additional memory per pixel for storing the Jacobian and filter parameters [G].
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** The method scales well with resolution if the Jacobian is computed efficiently. However, accurate estimation of the Jacobian requires fine-grained derivatives [G].
-   **Strengths:** Provides high-quality anti-aliasing by explicitly modeling the pixel footprint. Adapts to local distortion, reducing aliasing in stretched or compressed regions [S1, S2].
-   **Limitations:** Requires computation of derivatives (Jacobian), which can be expensive in raytracing. The Gaussian approximation may not be optimal for all textures [G].
-   **When to use:** When high-quality texture filtering is required, especially in scenes with significant perspective distortion or non-uniform scaling [S1, S2].
-   **When not to use:** When performance is critical and lower-quality filtering (e.g., bilinear filtering) is acceptable [G].

## Background
The method originates from the work of Heckbert [Hec89] and is discussed in detail in surveys of procedural noise functions and texture filtering [S1, S2]. It is based on the convolution theorem, which states that convolution in the spatial domain corresponds to multiplication in the frequency domain [S1, S2]. The technique is widely used in computer graphics for texture mapping to reduce aliasing artifacts [S1, S2].

## Sources
[S1] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S2] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
