# SGGX Parameter Fitting from Directional Transparencies

Also called: Algorithm Aniso, Algorithm Iso, Computing temporary parameters ρ0 and A0, Computing temporary parameters ρ0 and A0 from directional transparencies

## Overview
This method computes the initial microflake parameters—density $\rho_0$ and self-shadowing $A_0$—for a low-resolution voxel such that its directional transparencies match those of the input high-resolution voxel block along the principal axes [S1]. It serves as the first stage of SGGX parameter fitting, preceding the estimation of final self-shadowing probabilities [S1]. The method distinguishes between two variants: **Algorithm Iso**, which assumes isotropic self-shadowing ($A_0=1$), and **Algorithm Aniso**, which computes anisotropic self-shadowing parameters to exactly match the target transparencies [S1]. Builds on [GPU-CPU Data Flow for SGGX-H](7839-gpu-cpu-data-flow-for-sggx-h.md).

## How it works
**Inputs:**
- $L$: The side length (size) of the low-resolution voxel [S1].
- $T(\omega_x), T(\omega_y), T(\omega_z)$: The directional transparencies of the input high-resolution voxel block along the principal axes $X, Y, Z$ [S1].
- $D(m)$: The microflake normal distribution function [S1].

**Outputs:**
- $\rho_0$: The initial microflake density [S1].
- $A_0(\omega)$: The initial self-shadowing function [S1]. For Algorithm Iso, this is a scalar $A_0=1$ [S1]. For Algorithm Aniso, this is defined by parameters $s_X, s_Y, s_Z$ [S1].

**Data Structures:**
- The self-shadowing parameters for the anisotropic case are stored as three scalars $s_X, s_Y, s_Z$ [S1].

**Formulas:**
The transparency of a low-resolution voxel in direction $\omega_i$ is modeled as [S1]:
$$ \exp(-L\sigma_i(\omega_i)) = \exp\left(-L\rho_0 A_0(\omega_i) \int D(m) \langle \omega_i \cdot m \rangle dm\right) $$
where $\langle \cdot \rangle$ denotes the positive part function (i.e., $\max(0, \cdot)$) [S1].

The anisotropic self-shadowing function is defined as a quadratic form [S1]:
$$ A_0(\omega) = \omega^T \begin{bmatrix} s_X & 0 & 0 \\ 0 & s_Y & 0 \\ 0 & 0 & s_Z \end{bmatrix} \omega $$

**Algorithm Iso:**
1. Set $A_0 = 1$ for all $\omega$ [S1].
2. Compute $\rho_0$ by averaging the required density across the three principal axes to preserve average transparency [S1]:
   $$ \rho_0 = \frac{1}{3} \sum_{i \in \{X, Y, Z\}} \frac{-\log(T(\omega_i))}{L \int D(m) \langle \omega_i \cdot m \rangle dm} \quad (7.6) $$

**Algorithm Aniso:**
1. Compute $\rho_0$ as the maximum density required among the three principal axes to ensure no axis exceeds its target transparency [S1]:
   $$ \rho_0 = \max_{i \in \{X, Y, Z\}} \left( \frac{-\log(T(\omega_i))}{L \int D(m) \langle \omega_i \cdot m \rangle dm} \right) \quad (7.7) $$
2. Compute the anisotropic self-shadowing parameters $s_X, s_Y, s_Z$ for each axis $i \in \{X, Y, Z\}$ using [S1]:
   $$ s_i = \frac{-\log(T(\boldsymbol{\omega}_i))}{\rho_0 L \int D(m) \langle \boldsymbol{\omega}_i \cdot m \rangle dm} \quad (7.9) $$
   This ensures that the directional transparencies of the low-res voxel match exactly $T(\omega_x), T(\omega_y)$ and $T(\omega_z)$ of the input voxel block [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fit_parameters_iso(L, T_x, T_y, T_z, D):
    """
    Algorithm Iso: Computes rho0 assuming isotropic self-shadowing A0=1.
    """
    # [G] Evaluate I_i = \int D(m) \langle \omega_i \cdot m \rangle dm for the given D(m).
    int_x = 0.0  # Placeholder for integral along X
    int_y = 0.0  # Placeholder for integral along Y
    int_z = 0.0  # Placeholder for integral along Z
    
    # Guard against division by zero [G]
    if int_x == 0 or int_y == 0 or int_z == 0:
        raise ValueError("Integral term is zero, cannot compute density.")
        
    rho_x = -math.log(T_x) / (L * int_x)
    rho_y = -math.log(T_y) / (L * int_y)
    rho_z = -math.log(T_z) / (L * int_z)
    
    rho0 = (rho_x + rho_y + rho_z) / 3.0
    A0 = 1.0
    return rho0, A0

def fit_parameters_aniso(L, T_x, T_y, T_z, D):
    """
    Algorithm Aniso: Computes rho0 and anisotropic self-shadowing parameters sX, sY, sZ.
    """
    # [G] Evaluate I_i = \int D(m) \langle \omega_i \cdot m \rangle dm for the given D(m).
    int_x = 0.0  # Placeholder for integral along X
    int_y = 0.0  # Placeholder for integral along Y
    int_z = 0.0  # Placeholder for integral along Z
    
    # Guard against division by zero [G]
    if int_x == 0 or int_y == 0 or int_z == 0:
        raise ValueError("Integral term is zero, cannot compute parameters.")
        
    rho_x = -math.log(T_x) / (L * int_x)
    rho_y = -math.log(T_y) / (L * int_y)
    rho_z = -math.log(T_z) / (L * int_z)
    
    rho0 = max(rho_x, rho_y, rho_z)
    
    s_x = -math.log(T_x) / (rho0 * L * int_x)
    s_y = -math.log(T_y) / (rho0 * L * int_y)
    s_z = -math.log(T_z) / (rho0 * L * int_z)
    
    return rho0, s_x, s_y, s_z
```

**Edge Cases and Pitfalls:**
- Division by zero: If the integral $\int D(m) \langle \omega_i \cdot m \rangle dm$ is zero for any axis, the density calculation is undefined [G].
- Transparency values: $T(\omega_i)$ must be in the range $(0, 1]$ [G]. If $T(\omega_i) = 0$, $-\log(T(\omega_i))$ is infinite, leading to infinite density [G]. If $T(\omega_i) = 1$, the density contribution is zero [G].

## Operating characteristics
**Time Complexity:** The method requires computing three integrals (one per axis) and a constant number of arithmetic operations [G].
**Memory Complexity:** Minimal, requiring storage for a few scalar values ($\rho_0, A_0$ or $s_X, s_Y, s_Z$) [S1].
**Strengths:** Algorithm Aniso exactly matches the directional transparencies along the principal axes, providing a more accurate representation of anisotropic scattering compared to Algorithm Iso [S1].
**Limitations:** Algorithm Iso may not accurately represent anisotropic scenes as it assumes uniform self-shadowing [S1]. Both algorithms only consider the principal axes, potentially missing directional information in other orientations [S1].

## Background
This method is described in the paper "A new microflake model with microscopic self-shadowing for accurate volume downsampling" [S1]. It is part of the SGGX-based downsampling pipeline, specifically following the GPU-CPU data flow for SGGX-H [S1]. The method is used to initialize the microflake parameters before further refinement steps, such as estimating self-shadowing [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
