# One-pass Spectral Rendering Integral

Also called: Bergner Spectral Rendering, Spectral Volume Rendering Integral

## Overview
The One-pass Spectral Rendering Integral is a real-time approximation for light transport in participating media that simplifies the multiple scattering model by removing global illumination elements to facilitate ray casting with local illumination [S1, book]. It enables interactive data integration using different light sources during post-illumination by factoring the light source out of the volume rendering integral computation [S1, book]. It is distinguished from its siblings by its specific simplification of assigning a volumetric shadow ratio of 1 and considering only irradiance/radiance aligned with the viewing ray, computed via a local Phong-based model [S1, book].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Light source spectral power distribution $L(\lambda)$, viewing direction $\omega$, sampling position $x$, entry point $d$ (where the light ray first enters the volume), distance $D = |x - d|$, spectral extinction coefficient $\tau(x, \lambda)$, and local material reflectance properties for the Phong model.
*   **Outputs:** Irradiance $I_i(x, \omega, \lambda)$ at position $x$.

**Data Structures:**
*   Spectral extinction coefficient $\tau(x, \lambda)$: Defined per voxel and wavelength.
*   Local material reflectance: Parameters for the Phong model (ambient, diffuse, specular coefficients).

**Formulas:**
The irradiance $I_i(x, \omega, \lambda)$ is computed as [S1, book]:
$$I_i(x, \omega, \lambda) = L(\lambda) \cdot \tilde{I}_i(x, \omega, \lambda, D)$$
where $\tilde{I}_i$ is the proportion of light arriving at $x$ from the entry point $d$:
$$\tilde{I}_i(x, \boldsymbol{\omega}, \boldsymbol{\lambda}) = \tilde{I}_i(x - D\boldsymbol{\omega}, \boldsymbol{\omega}, \boldsymbol{\lambda}) \cdot T(x, \boldsymbol{\omega}, \boldsymbol{\lambda}, D) + \int_0^D \tilde{I}_r(x - s\boldsymbol{\omega}, \boldsymbol{\omega}, \boldsymbol{\lambda}) \cdot T(x, \boldsymbol{\omega}, \boldsymbol{\lambda}, s) ds$$
The spectral extinction function $T$ is defined as [S1, book]:
$$T(x, \omega, \lambda, s) = \exp\left(-\int_0^s \tau(x-t\omega, \lambda) dt\right)$$
The scattered radiance term $\tilde{I}_r$ is computed using a local illumination model based on the Phong model [S1, book].

**Steps:**
1.  Determine the entry point $d$ where the light ray first intersects the volume boundary for the given direction $\omega$.
2.  Calculate the distance $D$ from the sampling position $x$ to the entry point $d$.
3.  Evaluate the spectral extinction integral $\int_0^s \tau(x-t\omega, \lambda) dt$ for $s$ ranging from $0$ to $D$. This is typically discretized along the ray segment from $d$ to $x$.
4.  Compute the transmission $T(x, \omega, \lambda, s)$ using the exponential of the negative extinction integral.
5.  Compute the scattered radiance $\tilde{I}_r$ at each point along the ray using the local Phong illumination model. This involves calculating the diffuse and specular contributions based on the local material properties and the light direction, ignoring global shadows (shadow ratio = 1) [S1, book].
6.  Evaluate the integral $\int_0^D \tilde{I}_r(x - s\omega, \omega, \lambda) \cdot T(x, \omega, \lambda, s) ds$ using numerical quadrature (e.g., trapezoidal rule) over the sampled points.
7.  Compute the boundary term $\tilde{I}_i(x - D\omega, \omega, \lambda) \cdot T(x, \omega, \lambda, D)$. Since the ray enters at $d$, $\tilde{I}_i(x - D\omega, \omega, \lambda)$ represents the irradiance just outside the volume, which is typically $L(\lambda)$ or 0 if no light is present.
8.  Sum the boundary term and the integral result to get $\tilde{I}_i(x, \omega, \lambda)$.
9.  Multiply by the light source spectral power $L(\lambda)$ to obtain the final irradiance $I_i(x, \omega, \lambda)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def one_pass_spectral_rendering_integral(x, omega, L_lambda, tau_function, phong_params):
    """
    Computes the one-pass spectral rendering integral.
    
    Args:
        x: Current sampling position (vec3).
        omega: Light ray direction (vec3).
        L_lambda: Light source spectral power distribution (array of floats, one per wavelength).
        tau_function: Function returning spectral extinction coefficient tau(x, lambda) (array of floats).
        phong_params: Local material Phong model parameters.
        
    Returns:
        I_i: Irradiance at x (array of floats, one per wavelength).
    """
    # 1. Find entry point d and distance D
    # Note: The specific method to find d depends on the volume representation.
    # For a simple box, this is a ray-box intersection.
    d, D = find_entry_point(x, omega)
    
    if D <= 0:
        return zeros_like(L_lambda)
        
    # 2. Discretize the ray segment from d to x
    # Number of samples N depends on desired accuracy and resolution
    N = 64 
    ds = D / N
    wavelengths = get_wavelengths() # Array of lambda values
    
    # Precompute extinction along the ray
    # T(s) = exp(-integral_0^s tau(x-t*omega, lambda) dt)
    # We compute the cumulative optical depth tau_optical(s)
    tau_optical = zeros(N + 1)
    current_tau = 0.0
    for i in range(1, N + 1):
        s = i * ds
        # Sample tau at position x - s*omega
        # Note: The formula uses x-t*omega, where t goes from 0 to s.
        # This corresponds to points from x (t=0) to x-s*omega (t=s).
        # However, the integral in T is from 0 to s of tau(x-t*omega).
        # The ray goes from d to x. Let's parameterize from d.
        # Let's stick to the formula's definition: T(x, omega, lambda, s) = exp(-int_0^s tau(x-t*omega, lambda) dt)
        # Here s is the distance from x backwards along omega? 
        # The formula for I_i uses T(x, omega, lambda, D) and T(x, omega, lambda, s) inside the integral from 0 to D.
        # The term is tilde{I}_r(x - s*omega). This suggests s is distance from x towards d.
        # So we sample points p_s = x - s*omega for s in [0, D].
        
        p_s = x - s * omega
        tau_s = tau_function(p_s, wavelengths)
        
        # Approximate integral using trapezoidal rule or simple sum
        # d(tau_optical)/ds = tau(x-s*omega, lambda)
        # tau_optical(s) = integral_0^s tau(x-t*omega, lambda) dt
        # We can accumulate this.
        # For simplicity, assume constant tau over ds for this step
        current_tau += tau_s * ds
        tau_optical[i] = current_tau
        
    # Compute transmission T(s) = exp(-tau_optical(s))
    T_s = exp(-tau_optical)
    
    # 3. Compute scattered radiance I_r at each sample point
    # I_r is computed using local Phong model
    # I_r(x - s*omega, omega, lambda)
    I_r = zeros((N + 1, len(wavelengths)))
    for i in range(N + 1):
        s = i * ds
        p_s = x - s * omega
        # Compute Phong illumination at p_s for light direction omega
        # This is a local model, so it doesn't depend on other voxels
        I_r[i] = phong_illumination(p_s, omega, phong_params, L_lambda)
        
    # 4. Compute the integral term: int_0^D I_r(x-s*omega) * T(x, omega, s) ds
    # Using trapezoidal rule
    integral_term = zeros(len(wavelengths))
    for i in range(N):
        s_mid = (i + 0.5) * ds
        # Average of I_r and T at endpoints
        I_r_avg = (I_r[i] + I_r[i+1]) / 2.0
        T_avg = (T_s[i] + T_s[i+1]) / 2.0
        integral_term += I_r_avg * T_avg * ds
        
    # 5. Compute boundary term: I_i(x - D*omega) * T(x, omega, D)
    # x - D*omega is the entry point d
    # I_i(d) is the irradiance just outside the volume, which is L(lambda) if the light is visible
    # Assuming the light source is outside and visible at d
    I_i_boundary = L_lambda * T_s[N]
    
    # 6. Combine
    I_i = I_i_boundary + integral_term
    
    return I_i

def phong_illumination(p, light_dir, params, L_lambda):
    """
    Simplified Phong illumination model for spectral rendering.
    Returns spectral radiance.
    """
    # Normal vector n at p (not given in sources, assume precomputed or approximated)
    # [G] Normal vector n is required for Phong model. It must be provided or computed from volume gradient.
    n = compute_normal(p) 
    
    # Diffuse term
    # k_d * max(0, dot(n, light_dir)) * L_lambda
    diffuse = params['k_d'] * max(0.0, dot(n, light_dir)) * L_lambda
    
    # Specular term
    # k_s * max(0, dot(reflect(-light_dir, n), view_dir))^shininess * L_lambda
    # Note: The source says "considering only irradiance and radiance with the same direction as the viewing ray"
    # This simplification applies to the multiple scattering model's global terms.
    # For the local Phong model, we still need a view direction for specular highlights.
    # However, the source says "approximating the radiance at each point by the reflectance of local materials".
    # It's ambiguous if the view direction is used in the local Phong model here.
    # Standard Phong uses view direction. The simplification (ii) says "considering only irradiance and radiance 
    # with the same direction as the viewing ray". This might mean that the scattered radiance I_r is only 
    # considered in the viewing direction? Or that the light direction is the viewing direction?
    # The formula uses I_r(x-s*omega, omega, lambda). Omega is the light ray direction.
    # So the scattered radiance is computed for the light direction omega.
    # This implies the "viewing ray" in simplification (ii) refers to the light ray direction for the purpose of 
    # the local illumination model? Or that the final radiance is only evaluated in the viewing direction?
    # Given the context of "local illumination", it's likely a standard Phong but evaluated for the light direction.
    # Let's assume standard Phong with a hypothetical view direction, or that the specular term is simplified.
    # Since the source doesn't specify the view direction for the local Phong, and simplification (ii) mentions 
    # "same direction as the viewing ray", it's possible that the light direction omega IS the viewing direction 
    # for the local model? No, omega is the light ray direction.
    # Let's assume a standard Phong with a default view direction (e.g., camera direction) or that the specular 
    # term is ignored/simplified. 
    # [G] Without explicit view direction, we assume a standard Phong model where view_dir is the camera direction.
    # If the simplification implies that the scattered radiance is only non-zero in the viewing direction, 
    # then I_r is only computed for the viewing direction. But the integral is over the light ray.
    # This is a contradiction. 
    # Re-reading: "considering only irradiance and radiance with the same direction as the viewing ray".
    # This likely means that the multiple scattering term I_r is only considered for the viewing direction.
    # But the formula integrates I_r along the light ray.
    # Perhaps it means that the *incoming* radiance to the scattering event is only from the viewing direction?
    # No, I_r is "scattered radiance from the external illumination".
    # Let's stick to the most literal interpretation: The local Phong model is used to compute I_r.
    # The "simplification" might refer to the fact that we don't trace secondary rays for global illumination,
    # but just use a local model.
    # We will use a standard Phong model with a provided view direction.
    view_dir = get_camera_direction()
    reflect_dir = reflect(-light_dir, n)
    specular = params['k_s'] * pow(max(0.0, dot(reflect_dir, view_dir)), params['shininess']) * L_lambda
    
    # Ambient term
    ambient = params['k_a'] * L_lambda
    
    return ambient + diffuse + specular
```

**Parameters:**
*   $N$: Number of samples along the ray segment. Typical value: 64 [G].
*   Phong parameters ($k_a, k_d, k_s, shininess$): Material-specific.

**Edge Cases and Pitfalls:**
*   **Precision:** The exponential function in the extinction term can lead to underflow for large optical depths. Use clamping or log-space arithmetic if necessary.
*   **Boundaries:** The entry point $d$ must be correctly identified. If the ray does not enter the volume, the result is zero.
*   **Degenerate Input:** If $D=0$, the integral is zero.

## Subtopics

### Spectral Precomputation
Spectral Precomputation is a variant that uses full spectral computation instead of RGB to simulate light scattering, resulting in smoother, less saturated, and more natural colors [S2]. It uses an array of 4D textures to emulate a spectral texture with $M$ spectral channels, requiring $\lceil (M+1)/4 \rceil$ textures (the +1 is for storing $I_{SM}$) [S2]. Constants must be spectral (sampled at more than three wavelengths), and incident light intensity is calculated as blackbody radiation at 5778K according to Planck's law [S2]. All textures in the array are fetched using exactly the same texture coordinates for both reading and writing [S2]. Spectral color values are converted to CIE XYZ space using standard XYZ primaries, then moved to sRGB gamut and converted to sRGB by multiplication with an XYZ→sRGB conversion matrix [S2]. The final sRGB result is stored in a single 4D texture for use in rendering [S2]. A key pitfall is that RGB precomputation values cannot be used directly as RGB color components; they must be treated as 3-sample spectra and converted via the spectral-to-RGB pipeline [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray $N$ and the number of wavelengths $M$. The precomputation step (if used) is offline.
*   **Memory Complexity:** Depends on the volume resolution and the number of spectral channels. The spectral precomputation variant uses an array of 4D textures, which can be memory-intensive.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with resolution and spectral samples.
*   **Strengths:** Enables interactive rendering with different light sources via post-illumination; more natural colors than RGB methods.
*   **Limitations:** Simplifications (shadow ratio = 1, local illumination only) reduce accuracy for complex lighting scenarios; spectral computation is more expensive than RGB.
*   **When to use:** Real-time applications requiring natural-looking spectral effects with local lighting.
*   **When not to use:** Applications requiring accurate global illumination or shadowing.

## Background
The One-pass Spectral Rendering Integral was developed by Bergner et al. [7, 8] as part of their work on spectral volume rendering [S1, book]. It is based on a simplified multiple scattering model described in Section 4.1 of their work [S1, book]. The method was designed to facilitate ray casting with local illumination and post-illumination with different light sources by removing global illumination elements from the model [S1, book].

## Sources
[S1] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S2] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
