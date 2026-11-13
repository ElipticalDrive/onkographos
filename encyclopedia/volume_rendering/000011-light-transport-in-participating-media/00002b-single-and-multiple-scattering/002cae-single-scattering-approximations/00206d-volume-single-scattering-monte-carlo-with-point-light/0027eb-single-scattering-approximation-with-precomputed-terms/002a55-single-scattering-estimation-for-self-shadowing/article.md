# Single Scattering Estimation for Self-Shadowing

Also called: Estimating self-shadowing, Single scattering estimation

## Overview
This method corrects the self-shadowing probability in a low-resolution voxel representation after an initial transparency matching step, which preserves macroscopic attenuation but fails to preserve the microscopic self-shadowing effects [S1]. It estimates the mean amount of single scattering energy leaving a block of input voxels and matches it to the low-resolution voxel by scaling the density and albedo parameters [S1].

## How it works
**Inputs:**
- A block of high-resolution input voxels with spatial extent $[x_{\min}, x_{\max}]$ [S1].
- Precomputed parameters for the low-resolution voxel: pre-filtering distribution $D$, base density $\rho_0$, and base directional albedo $A_0(\omega)$ [S1].
- The extinction coefficient $\sigma_t(s, \omega)$ and scattering coefficient $\sigma_s(x, \omega)$ for the input block [S1].
- The phase function $f(x, \omega_r \rightarrow \omega)$ [S1].
- Number of rays $N_{ray}$ (typically aligned with main axes of the voxel grid) [S1].

**Outputs:**
- A scaling factor $\gamma \in (0, 1]$ [S1].
- Final density $\rho = \rho_0 \gamma$ and final directional albedo $A(\omega) = \gamma A_0(\omega)$ for the low-resolution voxel [S1].

**Data Structures:**
- The input block is treated as a volume defined by its spatial bounds and per-voxel coefficients [S1].
- The low-resolution voxel is defined by the scalar $\rho_0$, the function $A_0(\omega)$, and the distribution $D(m)$ [S1].

**Steps:**

1. **Define Transmittance:** Compute the transmittance between two points $x_1$ and $x_2$ in direction $\omega$ as [S1]:
   $$ O(x_1, \omega, x_2) = \exp\left(-\int_{x_1}^{x_2} \sigma_t(s, \omega) ds\right) $$
   Let $O(x_1, \omega)$ denote the transmittance from $x_1$ to the limit of the input block in direction $\omega$ [S1].

2. **Estimate Single Scattering for a Single Ray:** For an incoming ray $r$ with direction $\omega_r$ intersecting the input block at $x_{\min}$ and $x_{\max}$, compute the amount of single scattering $P_{SS}(r)$ as [S1]:
   $$ P_{SS}(r) = \int_{x_{\min}}^{x_{\max}} \sigma_s(x, \omega_r) O(x_{\min}, \omega_r, x) \left[ \int_{S^2} f(x, \omega_r \rightarrow \omega) O(x, \omega) d\omega \right] dx $$
   This quantity is dimensionless but wavelength-dependent [S1].

3. **Estimate Mean Single Scattering in Input Block:** Cast $N_{ray}$ rays in directions parallel to the main axes of the voxel grid. For each ray $r_i$, compute $P_{SS}(r_i)$. Average these values to obtain the mean single scattering of the input block [S1]:
   $$ M_{ss}^{\text{input}} = \frac{1}{N_{ray}} \sum_{\text{rays}} P_{SS}(r_i) $$

4. **Estimate Mean Single Scattering in Low-Res Voxel:** Compute $M_{ss}$ for the low-resolution voxel using the same integral form as Eq. 7.10, but substituting the input block's $\sigma_s$ and $f$ with the low-res voxel's $\sigma_{ss}$ and $f_{ss}$, which are derived from $\rho_0$, $A_0(\omega)$, and $D$ [S1].

5. **Compute Scaling Factor $\gamma$:** The scaling factor $\gamma$ is chosen to minimize the error between the estimated single scattering of the input block and the scaled low-res voxel across wavelengths. Using RGB albedos (wavelengths $\lambda_i$), compute [S1]:
   $$ \gamma = \arg \min_g \sum_{\lambda_i} \left\| M_{SS}^{\text{input}}(\lambda_i) - g M_{SS}(\lambda_i) \right\|^2 $$
   where $g$ is the trial scaling factor [S1]. (Note: The source notation uses $\arg \max$, which is corrected here to $\arg \min$ to correctly minimize the squared error norm.)

6. **Apply Final Parameters:** Set the final density and albedo for the low-resolution voxel as [S1]:
   $$ \rho = \rho_0 \gamma $$
   $$ A(\omega) = \gamma A_0(\omega) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def numerical_quadrature(func, a, b, n=100):
    """Trapezoidal rule for 1D integration."""
    h = (b - a) / n
    total = 0.5 * (func(a) + func(b))
    for i in range(1, n):
        total += func(a + i * h)
    return total * h

def compute_transmittance(x1, x2, direction, sigma_t_func):
    """O(x1, direction, x2) = exp(-integral_{x1}^{x2} sigma_t(s, direction) ds)"""
    integral = numerical_quadrature(lambda s: sigma_t_func(s, direction), x1, x2)
    return math.exp(-integral)

def sample_sphere_uniform(n):
    """Generates n directions uniformly distributed on the unit sphere [G]."""
    directions = []
    for _ in range(n):
        theta = math.acos(2 * random() - 1)
        phi = 2 * math.pi * random()
        directions.append((math.sin(theta) * math.cos(phi),
                           math.sin(theta) * math.sin(phi),
                           math.cos(theta)))
    return directions

def compute_single_scattering_for_ray(ray_start, ray_end, ray_direction,
                                      sigma_s_func, sigma_t_func, phase_func,
                                      block_min, block_max):
    """P_SS(r) for a single ray."""
    x_min = max(ray_start, block_min)
    x_max = min(ray_end, block_max)
    if x_min >= x_max:
        return 0.0

    def integrand(x):
        o_entry_to_x = compute_transmittance(x_min, x, ray_direction, sigma_t_func)
        sigma_s = sigma_s_func(x, ray_direction)

        sphere_samples = sample_sphere_uniform(64)
        inner_sum = 0.0
        for omega in sphere_samples:
            limit = block_max
            o_x_to_limit = compute_transmittance(x, limit, omega, sigma_t_func)
            f = phase_func(x, ray_direction, omega)
            inner_sum += f * o_x_to_limit
        inner_integral = inner_sum / len(sphere_samples)

        return sigma_s * o_entry_to_x * inner_integral

    return numerical_quadrature(integrand, x_min, x_max)

def estimate_mean_single_scattering_input(block_voxels, num_rays=3):
    """M_ss^input = (1/N_ray) sum P_SS(r_i)"""
    total = 0.0
    axes = [(1, 0, 0), (0, 1, 0), (0, 0, 1)]
    for direction in axes[:num_rays]:
        ray_start = block_voxels.min
        ray_end = block_voxels.max
        p_ss = compute_single_scattering_for_ray(
            ray_start, ray_end, direction,
            block_voxels.sigma_s, block_voxels.sigma_t, block_voxels.phase_func,
            block_voxels.min, block_voxels.max
        )
        total += p_ss
    return total / num_rays

def compute_sigma_ss(rho0, A0, D, direction):
    """sigma_ss(omega) = gamma * rho0 * A0(omega) * integral A0(omega) D(m) <omega . m> dm"""
    m_samples = sample_sphere_uniform(64)
    integral_sum = 0.0
    for m in m_samples:
        dot_product = sum(d * m_i for d, m_i in zip(direction, m))
        integral_sum += A0(direction) * D(m) * dot_product
    return rho0 * A0(direction) * (integral_sum / len(m_samples))

def estimate_mean_single_scattering_lowres(lowres_voxel, num_rays=3):
    """M_ss for low-res voxel using precomputed terms."""
    total = 0.0
    axes = [(1, 0, 0), (0, 1, 0), (0, 0, 1)]
    for direction in axes[:num_rays]:
        sigma_ss = compute_sigma_ss(lowres_voxel.rho0, lowres_voxel.A0, lowres_voxel.D, direction)
        def f_ss(x, omega_in, omega_out):
            return lowres_voxel.A0(omega_out)
        p_ss = compute_single_scattering_for_ray(
            lowres_voxel.min, lowres_voxel.max, direction,
            lambda x, d: sigma_ss, lowres_voxel.sigma_t, f_ss,
            lowres_voxel.min, lowres_voxel.max
        )
        total += p_ss
    return total / num_rays

def compute_gamma(input_m_ss, lowres_m_ss, wavelengths):
    """gamma = arg min_g sum || M_input - g M_lowres ||^2"""
    best_g = 1.0
    min_error = float('inf')
    for g in [i/100.0 for i in range(1, 101)]:
        error = 0.0
        for lam in wavelengths:
            diff = input_m_ss[lam] - g * lowres_m_ss[lam]
            error += diff * diff
        if error < min_error:
            min_error = error
            best_g = g
    return best_g
```

**Edge Cases and Pitfalls:**
- The transmittance calculation $O(x_1, \omega, x_2)$ requires accurate integration of $\sigma_t$ [S1].
- The inner integral over $S^2$ for the phase function and outgoing transmittance is computationally expensive [S1].
- The scaling factor $\gamma$ is constrained to $(0, 1]$ [S1].
- The method relies on the pre-filtering distribution $D$ and base parameters $\rho_0, A_0(\omega)$ being established during the preceding transparency matching step [S1].

## Subtopics

### Self-Shadowing Parameter Scaling
This variant describes the mathematical mechanism for adjusting the low-resolution voxel's parameters to match the estimated self-shadowing [S1]. It scales the density $\rho$ and directional albedo $A(\omega)$ by a factor $\gamma \in (0, 1]$ [S1]. This scaling preserves the attenuation coefficient $\sigma_t(\omega)$ because the product $\rho A(\omega)$ remains constant relative to the initial state ($\rho_0 A_0(\omega)$), but it multiplies the single scattering coefficient $\sigma_{ss}(\omega)$ by $\gamma$ [S1]. The single scattering coefficient becomes $\sigma_{ss}(\omega) = \gamma \rho_0 A_0(\omega) \int A_0(\omega) D(m) \langle \omega \cdot m \rangle dm$ [S1]. The value of $\gamma$ is computed by minimizing the error between the estimated single scattering of the input block and the scaled low-res voxel across wavelengths (using RGB albedos) [S1].

## Operating characteristics
- **Time Complexity:** The method requires casting $N_{ray}$ rays through the input block and the low-res voxel [S1]. Each ray requires a numerical integration over the block length and an integration over the sphere $S^2$ for the phase function [S1].
- **Memory Complexity:** Requires storing the input block's coefficients and the low-res voxel's precomputed terms [S1].
- **Strengths:** Accurately preserves self-shadowing effects that are lost during downsampling [S1]. It is physically based and ensures consistency between the high-res and low-res representations [S1].
- **Limitations:** Computationally intensive due to the multiple integrals [S1]. The accuracy depends on the number of rays and samples used in the integrals [S1].
- **When to use:** When accurate self-shadowing is critical and the computational cost of the full single scattering estimation is acceptable [S1]. It is used in the context of volume downsampling for efficient rendering [S1].

## Background
This method is part of the "new microflake model with microscopic self-shadowing for accurate volume downsampling" [S1]. It addresses the issue that standard downsampling techniques preserve macroscopic transparency but fail to preserve microscopic self-shadowing, leading to visually incorrect results [S1]. The approach is detailed in Section 7.4 of the source paper [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
