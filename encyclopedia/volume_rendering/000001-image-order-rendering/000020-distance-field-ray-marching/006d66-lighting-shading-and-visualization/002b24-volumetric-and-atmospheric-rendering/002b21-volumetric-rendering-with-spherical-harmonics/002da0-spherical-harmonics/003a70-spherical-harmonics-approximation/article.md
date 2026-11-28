# Spherical Harmonics Approximation

Also called: SH Approximation, Spherical Harmonics Basis Approximation

## Overview
Spherical Harmonics (SH) Approximation is a technique to represent a spherical function $f(\omega)$ using a finite subset of the SH basis functions, enabling efficient and differentiable computation for inverse problems in volume rendering [S1]. It solves the problem of expensive Monte Carlo integration by projecting directional data (such as lighting, visibility, and in-scattering) into the SH domain, where integrals become simple dot products and convolutions become matrix-vector multiplications [S1].

## How it works
**Inputs:** A spherical function $f(\omega)$ defined over the domain $\Omega$ (the unit sphere), and a maximum bandwidth $L$ (or number of bands $n$) [S1].
**Outputs:** A vector of SH coefficients $\mathbf{f}$ of size $n = (L+1)^2$ [S1].

**Data Structures:**
The coefficients are stored in a 1D array $\mathbf{f}$ of length $n$ [S1]. The linear index $i$ maps the degree $l$ and order $m$ via:
$$ i = l^2 + l + m $$
where $l \in \{0, \dots, L\}$ and $m \in \{-l, \dots, l\}$ [S1].

**Steps:**
1. **Projection:** Compute the SH coefficients $f_i$ by projecting the function $f(\omega)$ onto the basis functions $y_i(\omega)$:
   $$ f_i = \int_{\Omega} f(\omega) y_i(\omega) d\omega $$
   [S1]. In practice, for inverse problems, this integral is often precomputed or solved via numerical integration at initialization to ensure differentiability with respect to scene parameters [S1].
2. **Reconstruction:** Approximate the original signal $\tilde{f}(\omega)$ by summing the basis functions weighted by the coefficients:
   $$ \tilde{f}(\omega) = \sum_{i=1}^{n} f_i y_i(\omega) $$
   [S1]. The accuracy increases with $n$ [S1].
3. **Integration via Dot Product:** The integral of the product of two approximated functions $\tilde{a}(\omega)$ and $\tilde{b}(\omega)$ is computed as the dot product of their coefficient vectors:
   $$ \int_{\Omega} \tilde{a}(\omega)\tilde{b}(\omega)d\omega = \mathbf{a} \cdot \mathbf{b} $$
   [S1].
4. **Convolution (Multiplication Projection):** The SH projection of the product of two functions, denoted $a * b$, is computed using a precomputed transfer matrix $C_{ijk}$:
   $$ (a * b)_i = \sum_{j=1}^{n} \sum_{k=1}^{n} C_{ijk} a_j b_k $$
   where $C_{ijk} = \int_{\Omega} y_i(\omega)y_j(\omega)y_k(\omega)d\omega$ [S1].

**Parameters:**
* **Bandwidth $n$:** The number of bands used. A value of $n=8$ is cited as providing good low-frequency approximations for inverse problems [S1]. Typical implementations use the lowest 3-5 levels of SH to keep the representation compact [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sh_coefficients(function_f, basis_functions_y, domain_samples):
    """
    Computes SH coefficients f_i by numerically integrating f(w) * y_i(w) over the sphere.
    function_f: callable, returns f(w) for a direction w.
    basis_functions_y: callable, returns y_i(w) for a direction w and index i.
    domain_samples: list of (w, weight) tuples representing the discretized sphere.
    """
    n = len(basis_functions_y)
    f = [0.0] * n
    for w, weight in domain_samples:
        val_f = function_f(w)
        for i in range(n):
            val_y = basis_functions_y(i, w)
            f[i] += val_f * val_y * weight
    return f

def reconstruct_sh_signal(coefficients_f, basis_functions_y, direction_w):
    """
    Reconstructs the signal at a specific direction using SH coefficients.
    coefficients_f: list of SH coefficients f_i.
    basis_functions_y: callable, returns y_i(w).
    direction_w: the direction vector w.
    """
    signal = 0.0
    for i, coeff in enumerate(coefficients_f):
        signal += coeff * basis_functions_y(i, direction_w)
    return signal

def compute_convolution_coefficients(coefficients_a, coefficients_b, transfer_matrix_C):
    """
    Computes the SH coefficients of the product of two functions a and b.
    coefficients_a: list of SH coefficients for a.
    coefficients_b: list of SH coefficients for b.
    transfer_matrix_C: 3D array C[i][j][k].
    """
    n = len(coefficients_a)
    result = [0.0] * n
    for i in range(n):
        sum_val = 0.0
        for j in range(n):
            for k in range(n):
                sum_val += transfer_matrix_C[i][j][k] * coefficients_a[j] * coefficients_b[k]
        result[i] = sum_val
    return result

def compute_integral_dot_product(coefficients_a, coefficients_b):
    """
    Computes the integral of the product of two SH approximated functions.
    """
    return sum(a * b for a, b in zip(coefficients_a, coefficients_b))
```

## Subtopics

### [Spherical Harmonic Projection](11243-spherical-harmonic-projection.md)
Spherical Harmonic Projection is the process of approximating an arbitrary spherical function $f(\omega)$ by computing the coefficients of its expansion in the SH basis [Child 11243]. It differs from the parent [Spherical Harmonics Approximation](8741-spherical-harmonics-approximation.md) by focusing specifically on the computational mechanics of deriving these coefficients via least-squares minimization and leveraging the basis's orthonormality for efficient GPU-based integration [Child 11243]. It is chosen when efficient representation of complex angular data (lighting, reflectance) is required for volume rendering [Child 11243].

### Hemisphere Projection in SH Space
This variant reformulates the computation of the hemisphere visibility term $H(x)$ to be differentiable with respect to the surface normal $n(x)$ [S1]. Instead of direct integration, it uses a linear rotation function $\phi_{SH}$ to rotate a standard z-axis symmetric integral:
$$ H(x) = \phi_{SH}(n(x)) \int_{\Omega} \max(i_z \cdot \omega, 0)y(\omega)d\omega $$
where $i_z = (0, 0, 1)^T$ [S1]. The integral is solved using numerical integration at initialization and does not need to be differentiable, as only the SH coefficients of the environment map are updated [S1].

### ZH-to-SH Projection via Minimization
This method approximates an SH vector $f$ as a sum of rotated Zonal Harmonic (ZH) basis functions by minimizing the L2 error $\psi = \|f - g\|$ [S2, unofficial]. It iterates through ZH levels from lowest to highest frequency [S2, unofficial]. For each level, it calculates several hundred rotations uniformly distributed on the hemisphere, solves for the optimal SH coefficients $g^*_l$ for each rotation, and keeps the rotation with the lowest error [S2, unofficial]. The optimal coefficients for a fixed rotation $\hat{z}$ are:
$$ g^*_l = \frac{4\pi}{2l + 1} \sum_{m=-l}^{l} Y_{lm}(\hat{z}) f_{lm} $$
[S2, unofficial]. The selected ZH function is projected into the SH basis and subtracted from the target vector before proceeding to the next level [S2, unofficial].

### Zonal Harmonics (ZH) Approximation
Zonal Harmonics are a subset of SH defined by $m=0$, consisting of functions symmetric about the vertical axis [S2, unofficial]. A ZH function $g$ is a sum of the $n$ lowest-order ZH functions: $g = \sum_{l=1}^{n} g_l Y_{l0}$ [S2, unofficial]. To rotate a ZH vector $g$ and project it into the SH basis, each coefficient $f_{lm}$ is calculated as:
$$ f_{lm} = g^*_l Y_{lm}(\hat{z}_l) $$
where $g^*_l = g_l \sqrt{\frac{2\pi}{2l + 1}}$ and $\hat{z}_l$ is the rotation axis for level $l$ [S2, unofficial]. Projecting an SH vector into the ZH basis is a hard problem solved by minimizing the L2 norm $\psi = \|f - g\|$ [S2, unofficial].

## Operating characteristics
* **Time Complexity:** Projection is typically precomputed or solved via numerical integration at initialization to avoid expensive Monte Carlo integration [S1]. Convolution is $O(n^3)$ due to the triple summation in the transfer matrix [S1]. Reconstruction is $O(n)$ [S1].
* **Memory Complexity:** Storing coefficients is $O(n)$. The transfer matrix $C_{ijk}$ is precomputed [S1].
* **Hardware, Resolution, and Scene Performance:** Not given in the sources.
* **Scaling:** Not given in the sources.
* **Strengths:** Enables differentiable computation for inverse problems; converts expensive integrals into dot products; allows real-time rendering of global illumination effects via PRT [S1][S2, unofficial].
* **Limitations:** Low-pass filtering nature limits high-frequency detail representation; requires precomputation of transfer matrices or integrals; rotation of general SH functions is complex, often requiring conversion to ZH or other bases [S1][S2, unofficial].
* **When to use:** When global illumination or in-scattering effects need to be represented compactly and differentially for optimization or real-time rendering [S1][S2, unofficial].
* **When not to use:** When high-frequency directional details are critical and cannot be captured by low-bandwidth SH [S1].

## Background
The method applies spherical harmonic function approximation to Precomputed Radiance Transfer (PRT) for real-time global illumination [S2, unofficial]. Sloan et al. introduced the use of SH for representing light transport and the rotation of Zonal Harmonics [S2, unofficial]. The specific application to differentiable shadow computation and inverse problems in volume rendering was detailed by [S1] in 2021. Zonal Harmonics were popularized by Sloan et al. (2005) for efficient rotation and projection [S2, unofficial].

## Sources
[S1] Efficient and Differentiable Shadow Computation for Inverse Problems, https://doi.org/10.1109/iccv48922.2021.01286, fetched 2026-11-01
[S2] PDF A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbc.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
[S3] Enhancing Fourier Volume Rendering Using Contour Extraction, https://doi.org/10.1007/978-3-540-30136-3_58, fetched 2026-10-26
