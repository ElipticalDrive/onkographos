# One-pass Spectral Rendering Integral

Also called: Bergner Spectral Rendering, Spectral Volume Rendering Integral

## Overview
The One-pass Spectral Rendering Integral is a real-time approximation for light transport in participating media that factors out the light source to enable interactive post-illumination [S1, book]. It simplifies the multiple scattering model by removing global illumination elements, allowing ray casting with local illumination and subsequent application of different light sources [S1, book]. It is distinguished from its siblings by its specific spectral formulation and the decoupling of light source intensity $L(\lambda)$ from the volumetric transport term $\tilde{I}_i$, which is not present in the non-spectral [Precomputed Multiple Scattering Table](3324-precomputed-multiple-scattering-table.md) or the [PSF-based Multiple Scattering Rendering](1297-psf-based-multiple-scattering-rendering.md).

## How it works
**Inputs:**
*   $L(\lambda)$: Light source intensity at wavelength $\lambda$.
*   $\tau(x, \lambda)$: Spectral extinction coefficient (optical depth density) at position $x$ and wavelength $\lambda$.
*   $\tilde{I}_r(x, \omega, \lambda)$: Scattered radiance from external illumination, computed via a local Phong-based model [S1, book].
*   $\omega$: Direction of the light ray.
*   $x$: Sampling position within the volume.
*   $D$: Distance from $x$ to the entry point $d$ where the light ray first enters the volume.

**Outputs:**
*   $I_i(x, \omega, \lambda)$: Irradiance at position $x$ from light ray direction $\omega$.

**Data Structures:**
*   The method operates on volumetric data where $\tau(x, \lambda)$ is stored per voxel. No specific bit-packing is mandated by the source, but standard spectral volume representations (e.g., RGB or multi-spectral channels) are implied.

**Steps:**
1.  **Simplify Global Illumination:** Assign the volumetric shadow ratio to a constant value of 1 for all voxels and directions [S1, book].
2.  **Align Directions:** Consider only irradiance and radiance with the same direction as the viewing ray [S1, book].
3.  **Compute Local Scattered Radiance:** Calculate $\tilde{I}_r(x, \omega, \lambda)$ using a local illumination model based on the Phong reflection model [S1, book].
4.  **Compute Spectral Extinction:** For a given path segment of length $s$, compute the spectral extinction function $T$:
    $$T(x, \omega, \lambda, s) = \exp\left(-\int_0^s \tau(x-t\omega, \lambda) dt\right)$$
5.  **Compute Transport Proportion:** Calculate $\tilde{I}_i(x, \omega, \lambda)$, the proportion of light arriving at $x$ from the entry point $d$, using the recursive integral:
    $$\tilde{I}_i(x, \omega, \lambda) = \tilde{I}_i(x - D\omega, \omega, \lambda) \cdot T(x, \omega, \lambda, D) + \int_0^D \tilde{I}_r(x - s\omega, \omega, \lambda) \cdot T(x, \omega, \lambda, s) ds$$
    *Note: $\tilde{I}_i(x - D\omega, \omega, \lambda)$ represents the irradiance at the entry point $d$. If $d$ is outside the volume or in vacuum, this term is typically the incident light proportion or zero depending on boundary conditions.*
6.  **Factor Light Source:** Multiply the transport proportion by the light source intensity to get final irradiance:
    $$I_i(x, \omega, \lambda) = L(\lambda) \cdot \tilde{I}_i(x, \omega, \lambda, D)$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def one_pass_spectral_rendering_integral(
    tau: Function3D,      # Spectral extinction coefficient tau(x, lambda)
    phi_func: Function3D, # Local Phong illumination model for I_r
    L_lambda: float,      # Light source intensity at wavelength lambda
    omega: Vector3,       # Light ray direction
    x: Vector3,           # Sampling position
    D: float              # Distance to volume entry point d
) -> float:
    """
    Computes the one-pass spectral rendering integral irradiance.
    """
    # Step 1 & 2: Simplifications are implicit in the structure
    # Shadow ratio = 1, direction alignment handled by input omega

    # Step 3: Compute scattered radiance I_r at current position
    # phi_func returns I_r(x, omega, lambda) based on local Phong model
    I_r_x = phi_func(x, omega)

    # Step 4 & 5: Compute the transport proportion I_tilde_i
    # We need I_tilde_i at the entry point d = x - D * omega
    d = x - D * omega
    
    # Recursively or iteratively compute I_tilde_i at d.
    # If d is outside the volume, I_tilde_i(d) is typically 0 or 1 
    # depending on whether it's a boundary condition for external light.
    # For a single pass starting from entry, we assume I_tilde_i(d) = 1 
    # if d is the source boundary, or computed from previous steps.
    # Here we assume d is the entry point where light enters the volume.
    # If d is outside, I_tilde_i(d) = 0 for internal scattering contribution.
    I_tilde_i_d = 0.0 # [G] Assumption: Entry point has no prior internal scattering
    
    # Compute extinction from d to x (distance D)
    # T(x, omega, lambda, D) = exp(- integral_0^D tau(x - t*omega, lambda) dt)
    # Approximate integral using quadrature or precomputed optical depth
    optical_depth_D = integrate_tau(x, omega, D, tau)
    T_D = exp(-optical_depth_D)
    
    term1 = I_tilde_i_d * T_D
    
    # Compute integral of scattered radiance along the path
    # Integral_0^D I_r(x - s*omega) * T(x, omega, lambda, s) ds
    # This requires sampling along the segment from d to x
    integral_term = 0.0
    num_samples = 16 # [G] Typical sample count for integration
    ds = D / num_samples
    for i in range(num_samples):
        s = (i + 0.5) * ds
        pos = x - s * omega
        
        # Get scattered radiance at this position
        I_r_s = phi_func(pos, omega)
        
        # Get extinction from pos to x (distance s)
        optical_depth_s = integrate_tau(pos, omega, s, tau)
        T_s = exp(-optical_depth_s)
        
        integral_term += I_r_s * T_s * ds
        
    I_tilde_i_x = term1 + integral_term
    
    # Step 6: Factor out light source
    I_i = L_lambda * I_tilde_i_x
    
    return I_i

def integrate_tau(x: Vector3, omega: Vector3, s_max: float, tau: Function3D) -> float:
    """
    Approximates integral_0^s_max tau(x - t*omega, lambda) dt
    """
    num_samples = 16 # [G]
    ds = s_max / num_samples
    total_tau = 0.0
    for i in range(num_samples):
        t = (i + 0.5) * ds
        pos = x - t * omega
        total_tau += tau(pos) * ds
    return total_tau
```

**Parameters:**
*   $L(\lambda)$: Varies by light source.
*   $\tau(x, \lambda)$: Material property.
*   Phong parameters: Ambient, diffuse, specular coefficients, and shininess [S1, book].

**Edge Cases:**
*   **Entry Point Calculation:** $D$ must be correctly computed as the distance to the first intersection with the volume boundary. If the ray starts inside, $D=0$ and the first term vanishes.
*   **Precision:** The exponential extinction function can underflow for large optical depths. Clamping or using log-space arithmetic may be necessary.
*   **Local Illumination Approximation:** The use of a Phong model for $\tilde{I}_r$ ignores complex global scattering effects, leading to inaccuracies in highly scattering media [S1, book].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the light ray path for the integration step. The factorization allows $L(\lambda)$ to be applied at the end, enabling multiple light sources without re-rendering the volume transport [S1, book].
*   **Memory Complexity:** Depends on the storage of $\tau(x, \lambda)$ and the Phong model parameters. No large precomputed tables are required compared to [Precomputed Multiple Scattering Table](3324-precomputed-multiple-scattering-table.md).
*   **Strengths:** Enables interactive frame rates by decoupling light source computation from volume traversal. Simple to implement as it relies on standard local illumination models.
*   **Limitations:** The simplification of global illumination (shadow ratio = 1) and the use of local Phong scattering result in physically inaccurate lighting in dense or highly scattering media. It does not account for shadows cast by the volume itself on other parts of the volume [S1, book].
*   **When to Use:** Real-time applications where performance is critical and approximate lighting is acceptable. Not suitable for offline rendering requiring high physical accuracy.

## Background
The method was developed by Bergner et al. [7, 8] as a spectral volume rendering integral based on a simplified multiple scattering model [S1, book]. It was designed to facilitate ray casting with local illumination and post-illumination with different light sources by factoring the light source out of the volume rendering integral [S1, book]. The approach is described in the context of local and global illumination in the volume rendering integral [S1, book].

## Sources
[S1] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
