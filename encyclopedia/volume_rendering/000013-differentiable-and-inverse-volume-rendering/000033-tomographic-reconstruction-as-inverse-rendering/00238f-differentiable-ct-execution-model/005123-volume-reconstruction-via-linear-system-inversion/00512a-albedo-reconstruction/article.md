# Albedo Reconstruction

Also called: Property Reconstruction

## Overview
Albedo Reconstruction is a method within volume reconstruction via linear system inversion that optimizes the diffuse albedo field of a volume so that its rendered output matches target images [S1]. It isolates the albedo parameter by assuming single scattering, thereby decoupling it from the emission and extinction terms to form a linear system [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Target radiance values $L^k_i(x_k, \omega_k)$ for a set of pixels $k$; known volume properties including extinction coefficient $\sigma_t(x_t)$, emission $L_{emit}(x_t)$, and single-scattering radiance $L_{scat}(x_t, \omega_k)$; ray origins $x_k$ and directions $\omega_k$; basis functions $\upsilon_i(x_t)$ representing the spatial domain [S1].
*   **Outputs:** Albedo coefficients vector $\vec{\rho} = [\rho_1, \rho_2, \dots, \rho_N]^T$ [S1].

**Data Structures:**
*   **Basis Functions:** A set of $N$ basis functions $\upsilon_i(x_t)$ used to represent the continuous albedo field $\rho(x_t) = \sum_{i=1}^N \rho_i \upsilon_i(x_t)$ [S1].
*   **Linear System Matrix:** A matrix $W$ of size $M \times N$, where $M$ is the number of pixel constraints and $N$ is the number of basis coefficients [S1]. Each row $k$ corresponds to a pixel constraint [S1].
*   **Observation Vector:** A vector $o$ of size $M$, containing the residual radiance for each pixel [S1].

**Numbered Steps:**
1.  **Linearization:** Start with the volume rendering equation [S1]. Split the medium radiance $L_m$ into emission and scattering components [S1]. Assume single scattering, which implies that the scattered radiance $L_{scat}(x_t, \omega_k)$ is independent of the albedo $\rho(x_t)$ [S1].
2.  **Residual Calculation:** For each pixel $k$, compute the observation vector component $o_k$ by subtracting the known contributions (transmitted background and emitted light) from the target radiance:
    $$ o_k = L^k_i(x_k, \omega_k) - T_r(x_k, x^k_s) L_o(x^k_s, \omega_k) - \int_c T_r(x_k, x_t) \sigma_t(x_t) L_{emit}(x_t) dt $$
    where $T_r$ is the transmittance, $x^k_s$ is the scene boundary, and $L_o$ is the background radiance [S1].
3.  **Albedo Formulation:** Substitute the albedo representation into the scattering integral [S1]. The remaining term depends linearly on $\rho$:
    $$ o_k = \int_c T_r(x_k, x_t) \sigma_t(x_t) L_{scat}(x_t, \omega_k) \rho(x_t) dt $$
4.  **Basis Expansion:** Represent $\rho(x_t)$ as a linear combination of basis functions $\upsilon_i(x_t)$:
    $$ \rho(x_t) = \sum_{i=1}^N \rho_i \upsilon_i(x_t) $$
5.  **Matrix Assembly:** Substitute the basis expansion into the integral [S1]. By linearity, the integral becomes a dot product between the coefficient vector $\vec{\rho}$ and a weight vector $\vec{w}_k$:
    $$ o_k = \sum_{i=1}^N \rho_i \left( \int_c T_r(x_k, x_t) \sigma_t(x_t) L_{scat}(x_t, \omega_k) \upsilon_i(x_t) dt \right) = \vec{\rho} \cdot \vec{w}_k $$
    The weight $w_{k,i}$ for pixel $k$ and basis $i$ is computed via ray marching, integrating the product of transmittance, extinction, single-scattering radiance, and the basis function along the ray [S1].
6.  **System Solution:** Assemble all $M$ equations into the linear system $W \vec{\rho} = o$ [S1]. Solve for $\vec{\rho}$ using a linear solver [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def albedo_reconstruction(pixels, volume_properties, basis_functions, num_steps=100):
    """
    Reconstructs albedo coefficients by solving a linear system derived from 
    single-scattering approximation.
    
    Args:
        pixels: List of pixel constraints, each containing target radiance L_target,
                ray origin x_k, direction omega_k, and scene boundary x_s.
        volume_properties: Object containing functions for sigma_t(x), L_emit(x),
                           L_scat(x, omega), and transmittance T_r(x1, x2).
        basis_functions: List of N basis functions upsilon_i(x).
        num_steps: Number of discrete steps for ray marching integration [G].
        
    Returns:
        rho: Array of shape (N,) containing albedo coefficients.
    """
    M = len(pixels)
    N = len(basis_functions)
    W = [[0.0] * N for _ in range(M)]
    o = [0.0] * M
    ray_length = 1.0  # Assumes unit ray length or precomputed [G]
    dt = ray_length / num_steps

    for k, pixel in enumerate(pixels):
        ray_dir = normalize(pixel.omega_k)  # Direction normalization [G]
        # Step 2: Calculate observation residual o_k
        background_term = volume_properties.transmittance(pixel.x_k, pixel.x_s) * \
                          volume_properties.background_radiance(pixel.x_s, pixel.omega_k)
        emission_integral = 0.0
        for j in range(num_steps):
            t = pixel.x_k + j * dt * ray_dir
            transmittance = volume_properties.transmittance(pixel.x_k, t)
            sigma_t = volume_properties.sigma_t(t)
            l_emit = volume_properties.l_emit(t)
            emission_integral += transmittance * sigma_t * l_emit * dt
        o[k] = pixel.L_target - background_term - emission_integral

        # Step 5: Compute weights w_k for this pixel
        for i, upsilon in enumerate(basis_functions):
            weight = 0.0
            for j in range(num_steps):
                t = pixel.x_k + j * dt * ray_dir
                transmittance = volume_properties.transmittance(pixel.x_k, t)
                sigma_t = volume_properties.sigma_t(t)
                l_scat = volume_properties.l_scat(t, pixel.omega_k)
                basis_val = upsilon(t)
                weight += transmittance * sigma_t * l_scat * basis_val * dt
            W[k][i] = weight

    # Step 6: Solve linear system W * rho = o
    rho = solve_linear_system(W, o)
    return rho

def solve_linear_system(W, o):
    # Solves W * rho = o using least squares if M > N, otherwise direct solve [G]
    MT = transpose(W)
    ATA = mat_mul(MT, W)
    ATb = mat_mul_vec(MT, o)
    return solve_linear_equations(ATA, ATb)

def transpose(M):
    return [[M[j][i] for j in range(len(M))] for i in range(len(M[0]))]

def mat_mul(A, B):
    return [[sum(A[i][k] * B[k][j] for k in range(len(B))) for j in range(len(B[0]))] for i in range(len(A))]

def mat_mul_vec(A, v):
    return [sum(A[i][j] * v[j] for j in range(len(v))) for i in range(len(A))]

def solve_linear_equations(A, b):
    # Gaussian elimination for Ax = b [G]
    n = len(A)
    for i in range(n):
        pivot = A[i][i]
        for j in range(n):
            A[i][j] /= pivot
        b[i] /= pivot
        for k in range(n):
            if k != i:
                factor = A[k][i]
                for j in range(n):
                    A[k][j] -= factor * A[i][j]
                b[k] -= factor * b[i]
    return b
```

**Parameters:**
*   **Basis Functions:** Choice of basis (e.g., trilinear grid, radial basis) determines $N$ [S1].
*   **Ray Marching Resolution:** Step size for numerical integration [G].
*   **Single Scattering Assumption:** The method relies on $L_{scat}$ being independent of $\rho$ [S1].

**Edge Cases and Pitfalls:**
*   **Single Scattering Assumption:** The method relies on $L_{scat}$ being independent of $\rho$ [S1].
*   **Boundary Conditions:** Accurate calculation of $T_r(x_k, x^k_s)$ and $L_o(x^k_s, \omega_k)$ is critical; errors here propagate directly into $o_k$ [S1].

## Subtopics

### [Joint Emission & Albedo Reconstruction](14425-joint-emission-albedo-reconstruction.md)
Jointly optimizes emission and diffuse albedo parameters by treating them as a single linear system, leveraging their linear independence to solve for both simultaneously. Unlike methods that fix one property to solve for the other, this approach updates both in a unified coefficient vector, reducing ambiguity in scenes where emission and reflected light are coupled.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Provides a direct, non-iterative solution for albedo under the single-scattering assumption [S1].
*   **Limitations:** Restricted to single-scattering scenarios [S1].
*   **When to Use:** Suitable for inverse rendering tasks where the volume is primarily single-scattering and emission/extinction are known [S1].
*   **When Not to Use:** Not given in the sources. [G]

## Background
The method originates from the work on property and lighting manipulations for static volume stylization [S1]. It builds on the linearization of the volume rendering equation under the single-scattering assumption, a common simplification in inverse volume rendering [S1].

## Sources
[S1] Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor, https://doi.org/10.1109/TVCG.2014.13, fetched 2026-10-26
