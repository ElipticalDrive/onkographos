# PSF-based Multiple Scattering Rendering

Also called: Elek et al. (2013) Screen-Space Multiple Scattering, Elek et al. method, PSF convolution method, Path integral and spatial spreading (Elek), Point Spread Function method, Screen-Space Multiple Scattering, Screen-space reference

## Overview
PSF-based Multiple Scattering Rendering approximates volumetric light transport by convolving a vacuum-rendered image with a spatially varying Gaussian Point Spread Function (PSF) derived from the medium's optical properties [S1][S2, unofficial]. It solves the problem of computationally expensive multiple scattering simulation by treating the medium as a blur filter, allowing for real-time performance on homogeneous media while handling directional lighting and visibility effects that simpler diffusion models miss [S1]. Unlike diffusion approximations which assume optically dense, homogeneous media with uniform angular distribution, this method explicitly models spatial spreading via path integrals, supporting spatially varying materials and anisotropic sources [S1].

## How it works
The method operates in two phases: preprocessing the vacuum-rendered image into attenuated and scattered components, and applying a spatially varying Gaussian blur to the scattered component based on pixel depth [S2, unofficial].

**Inputs:**
*   $L$: Input image rendered in a vacuum (radiance per pixel).
*   $D$: Depth buffer (distance from camera to surface for each pixel).
*   $\sigma_a$: Absorption coefficient.
*   $\sigma_s$: Scattering coefficient.
*   $g$: Anisotropy factor.
*   $\sigma_t = \sigma_a + \sigma_s$: Extinction coefficient.

**Outputs:**
*   $L'$: Final image with multiple scattering effects.

**Data Structures:**
*   $L_{\text{at}}$: Image of attenuated radiance (same dimensions as $L$).
*   $L_{\text{sc}}$: Image of scattered radiance (same dimensions as $L$).
*   $W(s)$: Function returning the standard deviation of the Gaussian kernel for a given depth $s$.
*   MIP map pyramid of $L_{\text{sc}}$: Precomputed levels of detail for efficient filtering [S2, unofficial].

**Steps:**

1.  **Preprocessing:** Compute the attenuated and scattered radiance images using the Beer-Lambert law.
    *   Compute attenuated radiance $L_{\text{at}}$:
        $$L_{\text{at}}(\mathbf{x}) = \exp(-\sigma_t D(\mathbf{x})) \cdot L(\mathbf{x})$$
        where $D(\mathbf{x})$ is the depth at pixel $\mathbf{x}$ [S2, unofficial].
    *   Compute scattered radiance $L_{\text{sc}}$:
        $$L_{\text{sc}}(\mathbf{x}) = \exp(-\sigma_a D(\mathbf{x})) \cdot (1 - \exp(-\sigma_s D(\mathbf{x}))) \cdot L(\mathbf{x})$$
        Note: $L_{\text{at}} + L_{\text{sc}} \neq L$ because absorbed radiance is lost [S2, unofficial].

2.  **PSF Derivation:** Calculate the standard deviation $W(s)$ for the Gaussian kernel based on the distance $s$ between points. For a collimated beam, the width is derived from the path integral framework [S2, unofficial]:
    $$W(s) = \sqrt{\frac{1}{2} \left( \frac{2\sigma_a}{3s} + \frac{4}{s^3\sigma_s(1-g)} \right)^{-1}}$$
    where $s$ is the distance (depth) [S2, unofficial].

3.  **Filtering:** Apply a spatially varying Gaussian blur to $L_{\text{sc}}$. The final image $L'$ is the sum of the direct attenuated light and the blurred scattered light:
    $$L'(\mathbf{x}) = L_{\text{at}}(\mathbf{x}) + \sum_{\mathbf{x}'} G_{\mathbf{x}'}(\|\mathbf{x}' - \mathbf{x}\|) L_{\text{sc}}(\mathbf{x}')$$
    where $G_{\mathbf{x}'}$ is a 2D Gaussian distribution centered at $\mathbf{x}'$ with standard deviation $W(D(\mathbf{x}'))$ [S2, unofficial].
    *   Implementation Note: To achieve real-time performance, this summation is typically implemented using MIP map filtering, where the kernel size determines the MIP level, avoiding $O(n^2)$ gathering costs [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def psf_multiple_scattering(L, D, sigma_a, sigma_s, g):
    """
    L: Input vacuum image (H, W, 3)
    D: Depth buffer (H, W)
    sigma_a: Absorption coefficient (scalar)
    sigma_s: Scattering coefficient (scalar)
    g: Anisotropy factor (scalar)
    Returns: L_prime (H, W, 3)
    """
    sigma_t = sigma_a + sigma_s
    
    # Step 1: Preprocessing
    # Attenuated radiance: light that reaches camera directly
    exp_at = np.exp(-sigma_t * D)
    L_at = L * exp_at[..., np.newaxis]
    
    # Scattered radiance: light scattered towards camera but not absorbed
    # Formula: exp(-sigma_a * D) * (1 - exp(-sigma_s * D)) * L
    exp_scatter = np.exp(-sigma_a * D) * (1.0 - np.exp(-sigma_s * D))
    L_sc = L * exp_scatter[..., np.newaxis]
    
    # Step 2 & 3: Filtering
    # Calculate PSF width W(s) for each pixel depth s
    # W(s) = sqrt( 0.5 * ( (2*sigma_a)/(3*s) + 4/(s^3 * sigma_s * (1-g)) )^-1 )
    # Avoid division by zero for s=0
    s = np.maximum(D, 1e-6)
    
    term1 = (2.0 * sigma_a) / (3.0 * s)
    term2 = 4.0 / (s**3 * sigma_s * (1.0 - g))
    inv_var = term1 + term2
    W = np.sqrt(0.5 / inv_var)
    
    # Apply spatially varying Gaussian blur to L_sc using W
    # In practice, this is done via MIP map filtering where kernel size ~ W
    # Here we denote the operation as a spatially varying convolution
    L_sc_blurred = spatially_varying_gaussian_blur(L_sc, W)
    
    # Composite
    L_prime = L_at + L_sc_blurred
    
    return L_prime
```

**Edge Cases and Pitfalls:**
*   **Homogeneous Media Only:** The method assumes homogeneous media. It does not natively support non-homogeneous analytically integrable media without modification [S2, unofficial].
*   **Energy Conservation:** $L_{\text{at}} + L_{\text{sc}} \neq L$. Absorbed radiance is not present in either image and is effectively lost [S2, unofficial].
*   **Depth Accuracy:** The PSF width $W(s)$ depends critically on depth $s$. Incorrect depth buffers (e.g., from occlusion culling errors) will result in incorrect blurring scales [S2, unofficial].
*   **Performance:** Naive gathering is $O(n^2 m^2)$ or worse. MIP map filtering is required for real-time performance [S2, unofficial].

## Subtopics

### [Narrow beam distribution method](9069-narrow-beam-distribution-method.md)
A screen-space multiple-scattering approximation that filters vacuum-rendered images using a heuristic distribution function derived from the assumption that light remains concentrated in narrow beams. It differs from sibling methods by explicitly modeling the spatial variation of scattering moments for heterogeneous media, though it is generally too slow for real-time performance compared to simpler MIP-map based approaches.

## Operating characteristics
*   **Time Complexity:** Naive gathering is $O(n^2 m^2)$ or $O(n^2 m^4)$ depending on neighborhood size [S2, unofficial]. With MIP map filtering, performance improves significantly, achieving ~2 milliseconds for a sample image compared to 30 seconds for gathering and 15 hours for path-tracing reference [S2, unofficial].
*   **Memory Complexity:** Requires storage for $L_{\text{at}}$, $L_{\text{sc}}$, and MIP map pyramids of $L_{\text{sc}}$ [S2, unofficial].
*   **Strengths:** Handles spatially varying general materials and highly directional effects; accounts for anisotropic sources and visibility effects; much faster than Monte Carlo or finite element techniques [S1].
*   **Limitations:** Restricted to homogeneous media; assumes spatial spreading can be modeled by a Gaussian PSF; not suitable for optically thin or highly heterogeneous media without extension [S1][S2, unofficial].
*   **When to Use:** Real-time rendering of homogeneous participating media where multiple scattering is desired and performance is critical.
*   **When Not to Use:** Heterogeneous media, optically dense media requiring diffusion approximation, or scenes requiring exact energy conservation without post-processing corrections.

## Background
The method was introduced by Elek et al. in 2013 as a screen-space multiple-scattering approximation [S1][S2, unofficial]. It builds on the mathematical path integration framework introduced by Tesfa [Tes87] and Perez et al. [PAS03] to describe multiple scattering effects in terms of spatial and angular blurring [S1]. The specific formula for the standard deviation of the Gaussian spatial-spreading distribution for a collimated beam was derived by Premože et al. [2004] [S2, unofficial]. The method was developed to address the computational expense of volumetric Monte Carlo and finite element techniques, which can take hours to days to solve [S1].

## Sources
[S1] Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S2] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
