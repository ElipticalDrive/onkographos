# Neural Basis Functions

Also called: Fourier basis, General basis form, b basis

## Overview
Neural Basis Functions are a set of parameterized functions used to represent the neural basis $\tilde{H}$ in the rendering equation, allowing ablation of different functional forms [S1]. They solve the problem of encoding viewing direction into a differentiable feature vector for view-dependent rendering [S1]. What sets them apart from siblings like [Spherical Harmonics Encoder](4390-spherical-harmonics-encoder.md) is the explicit use of custom analytic forms tailored to specific viewing constraints rather than relying on standard orthogonal polynomial bases [S1].

## Shared principles
The core mechanism maps a viewing direction vector $v$ to a fixed-length feature vector $\tilde{H}(v)$ [S1]. For forward-facing scenarios, the viewing angle is constrained to a hemisphere, and the $z$-component is computed from the $x$ and $y$ components as $v_z = \sqrt{1 - v_x^2 - v_y^2}$ [S1]. Consequently, the basis functions can be parameterized using only $v_x$ and $v_y$ [S1]. The general form combines real and imaginary parts of complex-valued functions $K_{a,b}(v)$ and $P_{a,b}(v)$ [S1].

The Fourier basis is defined as $\tilde{H}_{FS}(v) = [\cos(2^{-1}\pi v_x), \sin(2^{-1}\pi v_x), \dots, \cos(2^N \pi v_y), \sin(2^N \pi v_y)]$ where $v$ is the viewing angle, $v_x, v_y$ are its components, and $N$ is the maximum frequency index [S1]. The ellipsis denotes a sequence of terms for frequency indices $k$ ranging from $-1$ to $N$ [S1].

The general analytic basis uses parameters $a, b, m$ [S1]. The component functions are defined as $K_{a,b}(v) = \left( \frac{v_x}{\sqrt{v_z - a}} \frac{v_z - a}{v_z + 1} \right)^m \left( \frac{v_y}{\sqrt{v_z + 1}} \frac{v_z - a}{v_z + 1} \right)^m$ and $P_{a,b}(v) = v_z^{1-a} \frac{v_z - 1 + m + 1}{b + 2m + 2}$ [S1]. The final basis combines them as $\tilde{H}(v) = [ \text{Re}(K_{a,b}(v)), \text{Im}(K_{a,b}(v)), \text{Re}(P_{a,b}(v) \cdot K_{a,b}(v)), \text{Re}(K_{a,b}(v)), \text{Re}(K_{a,b}(v)) ]$ [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_neural_basis(vx, vy, a=0.0, b=0.0, m=1, N=5):
    """
    Computes the Fourier and General Neural Basis feature vectors.
    Assumes forward-facing hemisphere constraint.
    """
    # Enforce hemisphere constraint: vz is derived from vx, vy
    vz = math.sqrt(max(0.0, 1.0 - vx**2 - vy**2))
    
    # Fourier Basis: sequence for k in [-1, N]
    fs_basis = []
    for k in range(-1, N + 1):
        freq = (2.0 ** k) * math.pi
        fs_basis.append(math.cos(freq * vx))
        fs_basis.append(math.sin(freq * vx))
        fs_basis.append(math.cos(freq * vy))
        fs_basis.append(math.sin(freq * vy))
        
    # General Basis: Ka,b and Pa,b evaluation
    denom = vz + 1.0
    sqrt_vz_a = math.sqrt(max(0.0, vz - a))
    sqrt_vz_p1 = math.sqrt(max(0.0, vz + 1.0))
    
    # K_{a,b}(v) calculation
    k_ab = ((vx / sqrt_vz_a) * ((vz - a) / denom)) ** m * \
           ((vy / sqrt_vz_p1) * ((vz - a) / denom)) ** m
    
    # P_{a,b}(v) calculation
    p_ab = (vz ** (1.0 - a)) * ((vz - 1.0 + m + 1.0) / (b + 2.0 * m + 2.0))
    
    # Combine into general basis vector
    h_real_k = k_ab.real if isinstance(k_ab, complex) else k_ab
    h_imag_k = k_ab.imag if isinstance(k_ab, complex) else 0.0
    h_p_k = (p_ab * k_ab).real if isinstance(p_ab * k_ab, complex) else (p_ab * k_ab)
    
    general_basis = [h_real_k, h_imag_k, h_p_k, h_real_k, h_real_k]
    
    return fs_basis, general_basis
```

## Subtopics

### Fourier Basis
A basis similar to the positional encoding used in NeRF [S1]. It is defined for forward-facing scenarios using only $v_x$ and $v_y$ [S1]. It is chosen when a simple, high-frequency periodic encoding is sufficient and the viewing domain is hemispherical [S1].

### Ka,b and Pa,b Basis
A general analytic form using parameters $a, b, m$ [S1]. It is defined for general viewing directions [S1]. It is chosen for more complex view-dependent effects where the Fourier basis may be insufficient [S1].

## Background
The concept was introduced in "NeX: Real-time View Synthesis with Neural Basis Expansion" (CVPR 2021) by Ben Mildenhall, Pratul P. Srinivasan, Rodrigo Ortiz-Cayon, Nima Khademi Kalantari, Ravi Ramamoorthi, Ren Ng, and Abhishek Kar [S1]. The work aims to provide practical view synthesis with prescriptive sampling guidelines [S1]. The ablation study of these basis functions is detailed in the supplementary material of the NeX paper [S1].

## Sources
[S1] NeX: Real-time View Synthesis with Neural Basis Expansion, https://doi.org/10.1109/cvpr46437.2021.00843, fetched 2026-10-26
