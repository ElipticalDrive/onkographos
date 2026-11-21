# Albedo Reconstruction

Also called: Property Reconstruction

## Overview
Albedo Reconstruction is a method within volume reconstruction via linear system inversion that optimizes the diffuse albedo field of a volume so that its rendered output matches target images. It isolates the albedo parameter by assuming single scattering, thereby decoupling it from the emission and extinction terms to form a linear system. Unlike joint optimization methods that solve for multiple properties simultaneously, this approach treats albedo as the sole unknown coefficient vector, reducing ambiguity in scenes where emission and reflected light are otherwise coupled [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Target radiance values $L^k_i(x_k, \omega_k)$ for a set of pixels $k$; known volume properties including extinction coefficient $\sigma_t(x_t)$, emission $L_{emit}(x_t)$, and single-scattering radiance $L_{scat}(x_t, \omega_k)$; ray origins $x_k$ and directions $\omega_k$; basis functions $\upsilon_i(x_t)$ representing the spatial domain.
*   **Outputs:** Albedo coefficients vector $\vec{\rho} = [\rho_1, \rho_2, \dots, \rho_N]^T$.

**Data Structures:**
*   **Basis Functions:** A set of $N$ basis functions $\upsilon_i(x_t)$ (e.g., trilinear, radial basis) used to represent the continuous albedo field $\rho(x_t) = \sum_{i=1}^N \rho_i \upsilon_i(x_t)$.
*   **Linear System Matrix:** A matrix $W$ of size $M \times N$, where $M$ is the number of pixel constraints and $N$ is the number of basis coefficients. Each row $k$ corresponds to a pixel constraint.
*   **Observation Vector:** A vector $o$ of size $M$, containing the residual radiance for each pixel.

**Numbered Steps:**
1.  **Linearization:** Start with the volume rendering equation. Split the medium radiance $L_m$ into emission and scattering components. Assume single scattering, which implies that the scattered radiance $L_{scat}(x_t, \omega_k)$ is independent of the albedo $\rho(x_t)$ [S1].
2.  **Residual Calculation:** For each pixel $k$, compute the observation vector component $o_k$ by subtracting the known contributions (transmitted background and emitted light) from the target radiance:
    $$ o_k = L^k_i(x_k, \omega_k) - T_r(x_k, x^k_s) L_o(x^k_s, \omega_k) - \int_c T_r(x_k, x_t) \sigma_t(x_t) L_{emit}(x_t) dt $$
    where $T_r$ is the transmittance, $x^k_s$ is the scene boundary, and $L_o$ is the background radiance [S1].
3.  **Albedo Formulation:** Substitute the albedo representation into the scattering integral. The remaining term depends linearly on $\rho$:
    $$ o_k = \int_c T_r(x_k, x_t) \sigma_t(x_t) L_{scat}(x_t, \omega_k) \rho(x_t) dt $$
4.  **Basis Expansion:** Represent $\rho(x_t)$ as a linear combination of basis functions $\upsilon_i(x_t)$:
    $$ \rho(x_t) = \sum_{i=1}^N \rho_i \upsilon_i(x_t) $$
5.  **Matrix Assembly:** Substitute the basis expansion into the integral. By linearity, the integral becomes a dot product between the coefficient vector $\vec{\rho}$ and a weight vector $\vec{w}_k$:
    $$ o_k = \sum_{i=1}^N \rho_i \left( \int_c T_r(x_k, x_t) \sigma_t(x_t) L_{scat}(x_t, \omega_k) \upsilon_i(x_t) dt \right) = \vec{\rho} \cdot \vec{w}_k $$
    The weight $w_{k,i}$ for pixel $k$ and basis $i$ is computed via ray marching, integrating the product of transmittance, extinction, single-scattering radiance, and the basis function along the ray [S1].
6.  **System Solution:** Assemble all $M$ equations into the linear system $W \vec{\rho} = o$. Solve for $\vec{\rho}$ using a linear solver (e.g., least squares if $M > N$, or direct inversion if $M = N$) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def albedo_reconstruction(pixels, volume_properties, basis_functions):
    """
    Reconstructs albedo coefficients by solving a linear system derived from 
    single-scattering approximation.
    
    Args:
        pixels: List of pixel constraints, each containing target radiance L_target,
                ray origin x_k, direction omega_k, and scene boundary x_s.
        volume_properties: Object containing functions for sigma_t(x), L_emit(x),
                           L_scat(x, omega), and transmittance T_r(x1, x2).
        basis_functions: List of N basis functions upsilon_i(x).
        
    Returns:
        rho: Array of shape (N,) containing albedo coefficients.
    """
    M = len(pixels)
    N = len(basis_functions)
    W = zeros((M, N))
    o = zeros(M)
    
    for k, pixel in enumerate(pixels):
        # Step 2: Calculate observation residual o_k
        # Subtract transmitted background and emitted light from target
        background_term = volume_properties.transmittance(pixel.x_k, pixel.x_s) * \
                          volume_properties.background_radiance(pixel.x_s, pixel.omega_k)
        
        emission_integral = 0.0
        # Ray march to compute emission contribution along the ray
        for t in ray_march_points(pixel.x_k, pixel.omega_k):
            transmittance = volume_properties.transmittance(pixel.x_k, t)
            sigma_t = volume_properties.sigma_t(t)
            l_emit = volume_properties.l_emit(t)
            # Differential path length dt is handled by integration step size
            emission_integral += transmittance * sigma_t * l_emit * dt
            
        o[k] = pixel.L_target - background_term - emission_integral
        
        # Step 5: Compute weights w_k for this pixel
        for i, upsilon in enumerate(basis_functions):
            weight = 0.0
            # Ray march to compute the integral for basis function i
            for t in ray_march_points(pixel.x_k, pixel.omega_k):
                transmittance = volume_properties.transmittance(pixel.x_k, t)
                sigma_t = volume_properties.sigma_t(t)
                l_scat = volume_properties.l_scat(t, pixel.omega_k)
                basis_val = upsilon(t)
                
                # Integrand: T_r * sigma_t * L_scat * upsilon_i
                weight += transmittance * sigma_t * l_scat * basis_val * dt
            
            W[k, i] = weight
            
    # Step 6: Solve linear system W * rho = o
    # Use least squares if M > N, otherwise direct solve
    rho = solve_linear_system(W, o)
    
    return rho
```

**Parameters:**
*   **Basis Functions:** Choice of basis (e.g., trilinear grid, radial basis) determines $N$. Not given in the sources.
*   **Ray Marching Resolution:** Step size for numerical integration. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Single Scattering Assumption:** The method relies on $L_{scat}$ being independent of $\rho$. If multiple scattering is significant, the linearization fails [S1].
*   **Degenerate Input:** If the volume is optically thin or emission dominates, the signal-to-noise ratio for albedo may be low, leading to unstable solutions [S1].
*   **Boundary Conditions:** Accurate calculation of $T_r(x_k, x^k_s)$ and $L_o(x^k_s, \omega_k)$ is critical; errors here propagate directly into $o_k$ [S1].

## Subtopics

### [Joint Emission & Albedo Reconstruction](14425-joint-emission-albedo-reconstruction.md)
Jointly optimizes emission and diffuse albedo parameters by treating them as a single linear system, leveraging their linear independence to solve for both simultaneously. Unlike methods that fix one property to solve for the other, this approach updates both in a unified coefficient vector, reducing ambiguity in scenes where emission and reflected light are coupled.

## Operating characteristics
*   **Time Complexity:** $O(M \cdot N \cdot R)$, where $M$ is the number of pixels, $N$ is the number of basis functions, and $R$ is the number of ray marching steps per pixel. Solving the linear system takes $O(\min(M^3, N^3))$ or $O(MN^2)$ for least squares.
*   **Memory Complexity:** $O(MN)$ to store the matrix $W$ and vector $o$.
*   **Strengths:** Provides a direct, non-iterative solution for albedo under the single-scattering assumption. Computationally efficient compared to gradient-based optimization methods.
*   **Limitations:** Restricted to single-scattering scenarios. Cannot handle complex lighting interactions or multiple bounces. Accuracy depends heavily on the correctness of the assumed emission and extinction fields [S1].
*   **When to Use:** Suitable for stylization or inverse rendering tasks where the volume is primarily single-scattering and emission/extinction are known or can be estimated separately.
*   **When Not to Use:** Unsuitable for dense, participating media with significant multiple scattering or when emission and albedo are strongly coupled and cannot be decoupled [S1].

## Background
The method originates from the work on property and lighting manipulations for static volume stylization. It was introduced by [S1] as a technique to allow artists to "paint" albedo properties directly into volume data by inverting the rendering equation. The approach builds on the linearization of the volume rendering equation under the single-scattering assumption, a common simplification in inverse volume rendering [S1].

## Sources
[S1] Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor, https://doi.org/10.1109/TVCG.2014.13, fetched 2026-10-26
