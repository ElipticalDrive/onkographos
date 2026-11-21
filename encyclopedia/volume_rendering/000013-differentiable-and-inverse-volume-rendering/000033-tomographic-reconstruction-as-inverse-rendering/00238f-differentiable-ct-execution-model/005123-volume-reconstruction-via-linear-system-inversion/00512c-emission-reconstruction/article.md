# Emission Reconstruction

Also called: Emission Optimization, Emission Inversion

## Overview
Emission reconstruction is a differentiable inverse volume rendering technique that solves for the spatial distribution of emitted radiance ($L_{emit}$) within a volume such that the rendered output matches target images [S1]. It operates by linearizing the volume rendering equation under the assumption of single scattering, isolating the emission term from scattering and background contributions to form a linear system [S1]. Unlike [Albedo Reconstruction](14422-albedo-reconstruction.md), which optimizes the scattering color properties, this method treats emission coefficients as the primary unknowns [S1].

## How it works
**Inputs:**
- Target pixel values $L^k_i(x_k, \omega_k)$ for $k$ pixels.
- Camera ray origins $x_k$ and directions $\omega_k$.
- Background radiance $L_o(x^k_s, \omega_k)$ at the scene boundary $x^k_s$.
- Extinction coefficient field $\sigma_t(x)$.
- Scattering phase function $\rho(x)$ and scattered light field $L_{scat}(x, \omega)$ (assumed fixed or negligible in single-scattering approximation).
- Basis functions $\upsilon_i(x)$ for representing emission.

**Outputs:**
- Emission coefficients vector $\vec{L}_{emit}$, where $L_{emit}(x) = \sum_i L^i_{emit} \upsilon_i(x)$.

**Data Structures:**
- **Observation Vector $\mathbf{o}$**: A column vector of size $K$ (number of pixels), where each element $o_k$ represents the residual radiance attributable to emission.
- **Transport Matrix $\mathbf{W}$**: A matrix of size $K \times N$ (where $N$ is the number of basis functions). Element $W_{ki}$ represents the contribution of the $i$-th basis function to the $k$-th pixel.

**Steps:**
1. **Linearization**: Start with the volume rendering equation. Assume single scattering. Split the integral into emitted and scattered light components [S1].
2. **Isolation**: Define the observation vector $o_k$ by subtracting background and single-scattering contributions from the target pixel value:
   $$
   o_k := L^k_i(x_k, \omega_k) - T_r(x_k, x^k_s) L_o(x^k_s, \omega_k) - \int_c T_r(x_k, x_t) \sigma_t(x_t) \rho(x) L_{scat}(x_t, \omega_k) dt
   $$
   This simplifies to the emission-only integral:
   $$
   o_k = \int_c T_r(x_k, x_t) \sigma_t(x_t) L_{emit}(x_t) dt
   $$
   where $T_r(x_k, x_t)$ is the transmittance from $x_k$ to $x_t$, and $c$ is the ray path [S1].
3. **Basis Expansion**: Represent the unknown emission field $L_{emit}(x_t)$ as a linear combination of basis functions $\upsilon_i(x_t)$ with coefficients $L^i_{emit}$:
   $$
   L_{emit}(x_t) = \sum_i L^i_{emit} \upsilon_i(x_t)
   $$
4. **Matrix Assembly**: Substitute the basis expansion into the integral. By linearity, the integral becomes a sum:
   $$
   o_k = \sum_i L^i_{emit} \left( \int_c T_r(x_k, x_t) \sigma_t(x_t) \upsilon_i(x_t) dt \right)
   $$
   Define the matrix entries $w_{ki}$ as:
   $$
   w_{ki} = \int_c T_r(x_k, x_t) \sigma_t(x_t) \upsilon_i(x_t) dt
   $$
   The system is now $\mathbf{o} = \mathbf{W} \vec{L}_{emit}$ [S1].
5. **Solving**: Solve the linear system for $\vec{L}_{emit}$. Depending on constraints (e.g., non-negativity of emission), use least squares or non-negative least squares (NNLS) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def emission_reconstruction(target_radiance, camera_rays, sigma_t, 
                            background_radiance, scattering_field, 
                            basis_functions, num_basis):
    """
    Reconstruct emission coefficients from target images.
    
    Args:
        target_radiance: Array of shape (K,) containing L^k_i(x_k, omega_k).
        camera_rays: List of K rays, each with origin x_k, direction omega_k, 
                     and path c.
        sigma_t: Function returning extinction coefficient at point x.
        background_radiance: Function returning L_o(x^k_s, omega_k) at boundary.
        scattering_field: Function returning L_scat(x, omega) for single scattering.
        basis_functions: List of N basis functions upsilon_i(x).
        num_basis: Number of basis functions N.
        
    Returns:
        L_emit_coeffs: Array of shape (N,) containing L^i_emit.
    """
    K = len(camera_rays)
    o = np.zeros(K)
    W = np.zeros((K, num_basis))
    
    for k in range(K):
        ray = camera_rays[k]
        x_k = ray.origin
        omega_k = ray.direction
        c = ray.path
        
        # 1. Calculate background and scattering contributions
        x_s = ray.boundary_point
        L_o_val = background_radiance(x_s, omega_k)
        
        # Transmittance to boundary
        T_r_boundary = compute_transmittance(x_k, x_s, sigma_t)
        
        # Single scattering integral
        # Integral of T_r(x_k, x_t) * sigma_t(x_t) * rho * L_scat(x_t, omega_k) dt
        # Note: rho is assumed constant or absorbed into L_scat definition in source
        scattering_integral = integrate_along_ray(
            path=c, 
            integrand=lambda x_t: compute_transmittance(x_k, x_t, sigma_t) * \
                                  sigma_t(x_t) * scattering_field(x_t, omega_k)
        )
        
        # 2. Isolate emission term o_k
        o[k] = target_radiance[k] - T_r_boundary * L_o_val - scattering_integral
        
        # 3. Assemble matrix row W[k, :]
        for i in range(num_basis):
            upsilon_i = basis_functions[i]
            # w_ki = integral of T_r(x_k, x_t) * sigma_t(x_t) * upsilon_i(x_t) dt
            W[k, i] = integrate_along_ray(
                path=c, 
                integrand=lambda x_t: compute_transmittance(x_k, x_t, sigma_t) * \
                                      sigma_t(x_t) * upsilon_i(x_t)
            )
            
    # 4. Solve linear system o = W * L_emit
    # Using least squares; add non-negativity constraint if required [G]
    L_emit_coeffs, residuals, rank, s = np.linalg.lstsq(W, o, rcond=None)
    
    return L_emit_coeffs

def compute_transmittance(x_start, x_end, sigma_t):
    """Compute transmittance T_r(x_start, x_end) along the segment."""
    # Integrate sigma_t along the segment from x_start to x_end
    # T_r = exp(- integral(sigma_t dt))
    optical_depth = integrate_along_segment(x_start, x_end, sigma_t)
    return np.exp(-optical_depth)

def integrate_along_segment(x_start, x_end, func):
    """Numerical integration of func(x) along the line segment."""
    # Simple trapezoidal rule or Gaussian quadrature [G]
    # Assuming uniform sampling for simplicity
    num_samples = 100
    t = np.linspace(0, 1, num_samples)
    points = x_start + t * (x_end - x_start)
    values = np.array([func(p) for p in points])
    # Trapezoidal rule
    return np.trapz(values, t) * np.linalg.norm(x_end - x_start)

def integrate_along_ray(path, integrand):
    """Integrate integrand(x) along the ray path."""
    # Path is a sequence of points or a parametric curve
    # For simplicity, assume path is defined by points and we integrate between them
    # This is a placeholder for specific ray marching implementation [G]
    # In practice, this involves stepping along the ray and summing contributions
    return 0.0 # Placeholder for actual ray marching integration
```

**Parameters:**
- **Basis Functions**: The choice of $\upsilon_i(x)$ determines the spatial resolution and smoothness of the reconstructed emission. Common choices include voxel-based step functions or hierarchical bases [S1].
- **Single Scattering Assumption**: The method assumes single scattering to linearize the problem. Multiple scattering requires iterative refinement or non-linear solvers, which are not covered in the base method [S1].

**Edge Cases and Pitfalls:**
- **Degenerate Rays**: If a ray does not intersect the volume, the integral is zero, and the corresponding row in $\mathbf{W}$ is zero. This can lead to singular systems if not handled [S1].
- **Precision**: Transmittance calculations involve exponentials of optical depth. High optical depths can lead to underflow, resulting in zero transmittance and loss of gradient information [G].
- **Boundary Conditions**: Accurate estimation of background radiance $L_o$ and boundary points $x^k_s$ is critical. Errors here propagate directly into $o_k$ [S1].

## Subtopics

### [Light Reconstruction via Linear Transport Operator](14452-light-reconstruction-via-linear-transport-operator.md)
This method optimizes environmental illumination (outgoing radiance from the environment) to match a desired volume appearance, assuming the volume's intrinsic properties (emission, albedo, extinction) are spatially varying but fixed during the optimization process. It differs from sibling methods like [Emission Reconstruction](14424-emission-reconstruction.md) by treating the light source coefficients as the unknown variables rather than the volume properties, solving for the lighting that best explains the observed pixel values through a linear transport operator.

## Operating characteristics
- **Time Complexity**: Dominated by the assembly of the transport matrix $\mathbf{W}$, which requires integrating along each ray for each basis function. Complexity is $O(K \cdot N \cdot S)$, where $K$ is the number of pixels, $N$ is the number of basis functions, and $S$ is the number of samples per ray. Solving the linear system is $O(K \cdot N^2)$ or $O(N^3)$ depending on the solver [S1].
- **Memory Complexity**: $O(K \cdot N)$ to store the transport matrix $\mathbf{W}$ and observation vector $\mathbf{o}$ [S1].
- **Strengths**: Provides a direct, non-iterative solution for emission under single scattering. Efficient for static volumes with known geometry and scattering properties [S1].
- **Limitations**: Restricted to single scattering. Accuracy degrades in highly scattering media. Sensitive to errors in background and scattering field estimates [S1].
- **When to Use**: Suitable for stylization and inverse rendering of volumes where emission is the dominant light source and scattering is weak or known [S1].

## Background
The method originates from the work on property and lighting manipulations for static volume stylization [S1]. It was developed to allow artists to manipulate emission properties of volumes to achieve desired visual effects while maintaining physical plausibility under simplified scattering assumptions [S1].

## Sources
[S1] Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor, https://doi.org/10.1109/TVCG.2014.13, fetched 2026-10-26
