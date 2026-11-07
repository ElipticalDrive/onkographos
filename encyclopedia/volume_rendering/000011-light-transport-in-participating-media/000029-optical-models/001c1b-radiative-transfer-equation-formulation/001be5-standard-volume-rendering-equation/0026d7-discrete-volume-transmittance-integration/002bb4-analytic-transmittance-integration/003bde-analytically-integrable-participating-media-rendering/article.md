# Analytically Integrable Participating Media Rendering

Also called: Quasi-heterogeneous media rendering, Real-Time Light Transport in Analytically Integrable Participating Media

## Overview
A real-time rendering method for participating media that uses analytical integration of density functions to handle quasi-heterogeneous media (e.g., exponential density with altitude), improving upon homogeneous approximations [S1, unofficial]. Unlike Elek et al. [2013] which approximates media as homogeneous, this method handles non-homogeneous media by analytically integrating density and rescaling distances [S1, unofficial]. Unlike Premože et al. [2004] which samples light paths at multiple points (taking minutes), this method uses analytical integration and rescaling for real-time performance [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $L: \mathbb{N}^2 \rightarrow \mathbb{R}^3$: Input vacuum rendered image (radiance in a vacuum).
    *   $D: \mathbb{N}^2 \rightarrow \mathbb{R}$: World-space distances from pixels to the camera.
    *   $\varrho(\mathbf{x})$: Density function of the medium, assumed to be analytically integrable (quasi-heterogeneous).
    *   $C_a$: Absorption coefficient.
    *   $C_s$: Scattering coefficient.
    *   $C_t = C_a + C_s$: Total extinction coefficient.
*   **Outputs:**
    *   $L': \mathbb{N}^2 \rightarrow \mathbb{R}^3$: Final composited image with participating media effects.

**Data Structures**
*   $P: \mathbb{N}^2 \rightarrow \mathbb{R}$: Integrated density image.
*   $L_{\text{at}}: \mathbb{N}^2 \rightarrow \mathbb{R}^3$: Attenuated radiance image.
*   $L_{\text{sc}}: \mathbb{N}^2 \rightarrow \mathbb{R}^3$: Scattered radiance image.
*   $D': \mathbb{N}^2 \rightarrow \mathbb{R}$: Rescaled distance image.
*   $W: \mathbb{N}^2 \rightarrow \mathbb{R}$: Spreading width image.
*   $L_{\text{ssc}}: \mathbb{N}^2 \rightarrow \mathbb{R}^3$: Separated bright scattered radiance image.
*   MIP chains (pyramids) for $L_{\text{sc}}$, $W$, $D'$, and $L_{\text{ssc}}$, denoted as $L^{[0\dots K]}_{\text{sc}}$, $W^{[0\dots K]}$, $D'^{[0\dots K]}$, and $L^{[0\dots K]}_{\text{ssc}}$.

**Steps**

1.  **Preprocessing: Integrated Density**
    Integrate the density $\varrho$ along camera rays to compute the integrated density image $P$. The integration is performed from the camera position towards the pixel's position in 3D space [S1, unofficial]:
    $$
    P(\mathbf{x}) = \int_{\text{camera}}^{\text{pixel } \mathbf{x}} \varrho(\mathbf{x}') \, d\mathbf{x}'
    $$
    [S1, unofficial]

2.  **Preprocessing: Attenuated and Scattered Radiance**
    Compute the attenuated radiance $L_{\text{at}}$ and scattered radiance $L_{\text{sc}}$ using the Beer-Lambert law and the integrated density $P$ [S1, unofficial]:
    $$
    L_{\text{at}} = \exp(-C_t \cdot P) \cdot L
    $$
    $$
    L_{\text{sc}} = \exp(-C_a \cdot P) \cdot (1 - \exp(-C_s \cdot P)) \cdot L
    $$
    [S1, unofficial]

3.  **Preprocessing: Distance Rescaling**
    Rescale world-space distances $D$ to rescaled distances $D'$ to support non-homogeneous media. The method assumes that the ratio function can be approximated by the density function $\varrho'' \approx \varrho$, assuming a homogeneous medium density $\varrho' = 1$. This leads to the approximation $D' \approx P$ [S1, unofficial]:
    $$
    D' \approx P
    $$
    [S1, unofficial]

4.  **Preprocessing: Spreading Widths**
    Compute the spreading widths $W$ for the pixels' point spread functions using the rescaled distances $D'$ (or equivalently $P$). The spreading width function $W(D)$ is derived from the theory of Premože et al. [2004] but adapted for heterogeneous media via rescaling [S1, unofficial]:
    $$
    W(D') = W\left(\int_l \varrho''(\mathbf{x}) \, \mathrm{d}\mathbf{x}\right) \approx W\left(\int_l \varrho(\mathbf{x}) \, \mathrm{d}\mathbf{x}\right)
    $$
    Thus, $W = W(D') \approx W(P)$ [S1, unofficial].

5.  **Filtering: MIP Chain Construction**
    Build Gaussian MIP chains for the following images:
    *   $L^{[0\dots K]}_{\text{sc}}$: MIP chain for scattered radiance.
    *   $W^{[0\dots K]}$: MIP chain for luminance weighting.
    *   $D'^{[0\dots K]}$: MIP chain for depth blurring.
    [S1, unofficial]

6.  **Filtering: Pixel Separation**
    Separate bright pixels from $L_{\text{sc}}$ to create $L_{\text{ssc}}$ and build a separate MIP chain $L^{[0\dots K]}_{\text{ssc}}$ for them. This step reduces artifacts of the filtering approach [S1, unofficial].

7.  **Composition**
    Compose the final image $L'$ by interpolating pixels from the corresponding levels of the MIP pyramids ($L^{[0\dots K]}_{\text{sc}}$, $W^{[0\dots K]}$, $D'^{[0\dots K]}$, $L^{[0\dots K]}_{\text{ssc}}$) and adding an arbitrary emissive term for enhanced visual results [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytically_integrable_media_rendering(L, D, rho_func, C_a, C_s, K):
    """
    L: Input vacuum rendered image (N x M x 3)
    D: World-space distances (N x M)
    rho_func: Function returning density rho(x) for a given 3D point x
    C_a: Absorption coefficient
    C_s: Scattering coefficient
    K: Number of MIP levels
    """
    C_t = C_a + C_s
    N, M = L.shape[:2]
    
    # 1. Preprocessing: Integrated Density P
    # P(x) = integral from camera to pixel x of rho(x') dx'
    # This requires ray marching or analytical integration along camera rays.
    # For analytical integration, rho_func must be integrable.
    P = np.zeros((N, M))
    for y in range(N):
        for x in range(M):
            # Ray from camera to pixel (x, y)
            # Assuming camera origin and direction are known
            ray_origin = get_camera_origin()
            ray_direction = get_pixel_direction(x, y)
            
            # Integrate density along the ray segment from camera to distance D[y,x]
            # P[y,x] = integral_0^{D[y,x]} rho(ray_origin + t * ray_direction) dt
            # This step depends on the specific analytical form of rho_func.
            # If rho_func is complex, numerical integration might be used here,
            # but the method relies on analytical integrability for real-time performance.
            P[y, x] = integrate_density_along_ray(ray_origin, ray_direction, D[y, x], rho_func)
            
    # 2. Preprocessing: Attenuated and Scattered Radiance
    # L_at = exp(-C_t * P) * L
    L_at = np.exp(-C_t * P[:, :, np.newaxis]) * L
    
    # L_sc = exp(-C_a * P) * (1 - exp(-C_s * P)) * L
    L_sc = np.exp(-C_a * P[:, :, np.newaxis]) * (1 - np.exp(-C_s * P[:, :, np.newaxis])) * L
    
    # 3. Preprocessing: Distance Rescaling
    # D' approx P
    D_prime = P
    
    # 4. Preprocessing: Spreading Widths
    # W = W(D') approx W(P)
    # The exact form of W(D) is from Premoze et al. [2004], adapted for rescaled distance.
    # W(D) typically involves a Gaussian spread width proportional to D or sqrt(D) depending on scattering order.
    # Here we assume a generic function W_func that takes distance and returns width.
    W = compute_spreading_width(D_prime, C_s) # Implementation detail of W_func
    
    # 5. Filtering: Build MIP Chains
    # Build Gaussian MIP chains for L_sc, W, D_prime
    L_sc_mip = build_mip_chain(L_sc, K)
    W_mip = build_mip_chain(W, K)
    D_prime_mip = build_mip_chain(D_prime, K)
    
    # 6. Filtering: Pixel Separation
    # Separate bright pixels from L_sc to L_ssc
    # Thresholding logic is not specified in sources, assumed to be a standard bright pixel detection.
    threshold = 0.5 # Placeholder for bright pixel threshold
    mask = np.any(L_sc > threshold, axis=2) # Simple example mask
    L_ssc = np.where(mask[:, :, np.newaxis], L_sc, 0)
    L_ssc_mip = build_mip_chain(L_ssc, K)
    
    # 7. Composition
    # Compose final image L' by interpolating from MIP pyramids
    # The exact interpolation logic (e.g., weighted average based on W and D_prime) is not fully detailed in sources.
    # It involves fetching pixels from corresponding levels of MIP pyramids.
    L_prime = np.zeros_like(L)
    for y in range(N):
        for x in range(M):
            # Interpolate from MIP chains based on local properties
            # This step is complex and depends on the specific filtering implementation.
            # For example, using W to determine the blur kernel size and D_prime for depth-aware blending.
            L_prime[y, x] = compose_pixel(L_sc_mip, W_mip, D_prime_mip, L_ssc_mip, x, y)
            
    # Add arbitrary emissive term
    # L_prime += emissive_term
    return L_prime

def integrate_density_along_ray(origin, direction, max_dist, rho_func):
    """
    Integrates density along a ray from origin to origin + max_dist * direction.
    Assumes rho_func is analytically integrable.
    """
    # This is a placeholder for the specific analytical integration.
    # For example, if rho(z) = rho_0 * exp(-z/H), the integral can be computed analytically.
    # Here, we assume a generic numerical integration for demonstration,
    # but the method relies on analytical integration for real-time performance.
    # In practice, this would be replaced by the specific analytical formula for the chosen density function.
    num_samples = 100
    t_values = np.linspace(0, max_dist, num_samples)
    points = origin + t_values[:, np.newaxis] * direction[np.newaxis, :]
    densities = np.array([rho_func(p) for p in points])
    return np.trapz(densities, t_values) # Numerical integration for placeholder

def compute_spreading_width(D_prime, C_s):
    """
    Computes spreading width W based on rescaled distance D_prime and scattering coefficient C_s.
    Based on Premoze et al. [2004] theory, adapted for rescaled distance.
    """
    # The exact formula for W(D) is not provided in the sources.
    # It typically involves a Gaussian spread width.
    # For example, W(D) = k * sqrt(D) or similar.
    # Here, we assume a simple linear relationship for demonstration.
    return D_prime * C_s

def build_mip_chain(image, K):
    """
    Builds a MIP chain (pyramid) of K levels for the given image.
    """
    mip_chain = [image]
    current_image = image
    for _ in range(K):
        # Downsample by factor of 2
        next_image = np.zeros((current_image.shape[0] // 2, current_image.shape[1] // 2, current_image.shape[2]))
        for y in range(next_image.shape[0]):
            for x in range(next_image.shape[1]):
                # Simple average of 2x2 block
                next_image[y, x] = np.mean(current_image[2*y:2*y+2, 2*x:2*x+2], axis=(0, 1))
        mip_chain.append(next_image)
        current_image = next_image
    return mip_chain

def compose_pixel(L_sc_mip, W_mip, D_prime_mip, L_ssc_mip, x, y):
    """
    Composes a single pixel by interpolating from MIP pyramids.
    The exact interpolation logic is not specified in the sources.
    """
    # Placeholder for the complex interpolation logic.
    # It likely involves selecting the appropriate MIP level based on W and D_prime,
    # and blending between L_sc and L_ssc.
    return L_sc_mip[0][y, x] # Placeholder
```

## Operating characteristics
*   **Time Complexity:** Real-time performance is claimed, significantly faster than sampling-based methods like Premože et al. [2004] which take minutes [S1, unofficial]. The preprocessing step involves integrating density along rays, which can be costly if not analytically tractable, but the filtering and composition steps are efficient due to the use of MIP chains and interpolation [S1, unofficial].
*   **Memory Complexity:** Requires storage for multiple images ($L$, $D$, $P$, $L_{\text{at}}$, $L_{\text{sc}}$, $D'$, $W$, $L_{\text{ssc}}$) and their MIP chains. The memory usage scales with the image resolution and the number of MIP levels [S1, unofficial].
*   **Strengths:** Handles quasi-heterogeneous media (e.g., exponential density) in real-time, which is an improvement over homogeneous approximations [S1, unofficial].
*   **Limitations:** The method relies on the assumption that the density function is analytically integrable. It may introduce errors in heterogeneous media where density varies significantly perpendicular to the light path, as the 2D Gaussian distribution assumption may be violated [S1, unofficial].
*   **When to use:** Suitable for real-time rendering of participating media with quasi-heterogeneous density distributions where visual quality is important but full path tracing is too slow [S1, unofficial].
*   **When not to use:** Not suitable for media with arbitrary, non-analytically integrable density functions or where high accuracy in complex heterogeneous media is required [S1, unofficial].

## Background
The approach is based on Elek et al. [2013] with improvements for non-homogeneous media and pixel separation [S1, unofficial]. It extends the theory of Premože et al. [2004] [S1, unofficial]. The method was developed to address the limitations of homogeneous media approximations and the high computational cost of sampling-based methods [S1, unofficial].

## Sources
[S1] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
