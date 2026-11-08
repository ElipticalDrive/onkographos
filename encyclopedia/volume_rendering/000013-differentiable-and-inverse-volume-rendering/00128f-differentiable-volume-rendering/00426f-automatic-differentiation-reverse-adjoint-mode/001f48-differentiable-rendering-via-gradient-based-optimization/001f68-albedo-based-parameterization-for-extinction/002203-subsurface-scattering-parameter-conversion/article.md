# Subsurface Scattering Parameter Conversion
Also called: Chiang et al. conversion

## Overview
Converts artistic subsurface scattering parameters—specifically a medium color vector $\mathbf{A}$ and a scattering distance $d$—into physical volume rendering coefficients (extinction $\sigma_t$ and scattering $\sigma_s$) for a homogeneous medium [S1, unofficial]. This method enables artists to control the visual appearance of subsurface scattering using intuitive parameters while maintaining the physical consistency required for differentiable volume rendering [S1, unofficial]. It is distinct from direct coefficient specification by providing a closed-form approximation of the inverse relationship between albedo and surface color, decoupling the scattering behavior from the extinction scale [S1, unofficial].

## How it works
**Inputs:**
- $\mathbf{A}$: Medium color, a 3-component vector (Vec3f) representing the desired subsurface scattering color [S1, unofficial].
- $d$: Scattering distance, a scalar representing the characteristic depth of light penetration [S1, unofficial].

**Outputs:**
- $\sigma_t$: Extinction coefficient (Vec3f) [S1, unofficial].
- $\sigma_s$: Scattering coefficient (Vec3f) [S1, unofficial].
- $\sigma_a$: Absorption coefficient (Vec3f), derived as $\sigma_t - \sigma_s$ [S1, unofficial].

**Data Structures:**
- $\mathbf{A}$: `float[3]` or `Vec3f` structure.
- $d$: `float`.
- $\sigma_t, \sigma_s, \sigma_a$: `float[3]` or `Vec3f` structures.

**Steps:**
1. Compute the scalar albedo $\alpha$ from the medium color $\mathbf{A}$ using a cubic polynomial fit [S1, unofficial]:
   $$ \alpha = 1 - \exp(-5.09406 \cdot A + 2.61188 \cdot A^2 - 4.31805 \cdot A^3) $$
   Note: The sources define $\alpha$ as a scalar derived from $\mathbf{A}$, but $\mathbf{A}$ is a vector. In implementation, this formula is applied component-wise to each channel of $\mathbf{A}$ to produce a vector $\boldsymbol{\alpha}$, or $\mathbf{A}$ is treated as a scalar luminance value if the medium is grayscale. Given the context of "medium color" and "Vec3f", the polynomial is applied per-channel [S1, unofficial]. Let $\alpha_i$ be the albedo for channel $i$:
   $$ \alpha_i = 1 - \exp(-5.09406 \cdot A_i + 2.61188 \cdot A_i^2 - 4.31805 \cdot A_i^3) $$

2. Compute the scalar scaling factor $s$ from the medium color $\mathbf{A}$ using a quadratic polynomial [S1, unofficial]:
   $$ s = 1.9 - A + 3.5 \cdot (A - 0.8)^2 $$
   Similarly, this is applied component-wise:
   $$ s_i = 1.9 - A_i + 3.5 \cdot (A_i - 0.8)^2 $$

3. Compute the extinction coefficient $\sigma_t$ using the scattering distance $d$ and the scaling factor $s$ [S1, unofficial]:
   $$ \sigma_t = \frac{1}{d \cdot s} $$
   Applied component-wise:
   $$ \sigma_{t,i} = \frac{1}{d \cdot s_i} $$

4. Compute the scattering coefficient $\sigma_s$ using the albedo $\alpha$ and extinction $\sigma_t$ [S1, unofficial]:
   $$ \sigma_s = \alpha \cdot \sigma_t $$
   Applied component-wise:
   $$ \sigma_{s,i} = \alpha_i \cdot \sigma_{t,i} $$

5. Compute the absorption coefficient $\sigma_a$ [S1, unofficial]:
   $$ \sigma_a = \sigma_t - \sigma_s $$
   Applied component-wise:
   $$ \sigma_{a,i} = \sigma_{t,i} - \sigma_{s,i} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_subsurface_scattering_params(A_vec, d):
    """
    Convert artistic subsurface scattering parameters to physical coefficients.
    
    Args:
        A_vec (Vec3f): Medium color (albedo-like input).
        d (float): Scattering distance.
        
    Returns:
        sigma_t (Vec3f): Extinction coefficient.
        sigma_s (Vec3f): Scattering coefficient.
        sigma_a (Vec3f): Absorption coefficient.
    """
    # Initialize output vectors
    sigma_t = Vec3f(0.0, 0.0, 0.0)
    sigma_s = Vec3f(0.0, 0.0, 0.0)
    sigma_a = Vec3f(0.0, 0.0, 0.0)
    
    # Process each color channel
    for i in range(3):
        A_i = A_vec[i]
        
        # Step 1: Compute albedo alpha_i
        # alpha = 1 - exp(-5.09406 * A + 2.61188 * A^2 - 4.31805 * A^3)
        alpha_i = 1.0 - exp(-5.09406 * A_i + 2.61188 * A_i**2 - 4.31805 * A_i**3)
        
        # Step 2: Compute scaling factor s_i
        # s = 1.9 - A + 3.5 * (A - 0.8)^2
        s_i = 1.9 - A_i + 3.5 * (A_i - 0.8)**2
        
        # Step 3: Compute extinction sigma_t_i
        # sigma_t = 1 / (d * s)
        # Guard against division by zero if s_i is 0
        if s_i == 0:
            sigma_t_i = 0.0
        else:
            sigma_t_i = 1.0 / (d * s_i)
            
        # Step 4: Compute scattering sigma_s_i
        # sigma_s = alpha * sigma_t
        sigma_s_i = alpha_i * sigma_t_i
        
        # Step 5: Compute absorption sigma_a_i
        # sigma_a = sigma_t - sigma_s
        sigma_a_i = sigma_t_i - sigma_s_i
        
        # Assign to output vectors
        sigma_t[i] = sigma_t_i
        sigma_s[i] = sigma_s_i
        sigma_a[i] = sigma_a_i
        
    return sigma_t, sigma_s, sigma_a
```

**Parameters:**
- Polynomial coefficients for $\alpha$: $-5.09406, 2.61188, -4.31805$ [S1, unofficial].
- Polynomial coefficients for $s$: $1.9, 3.5, -0.8$ (offset) [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $s_i = 0$, division by zero occurs in $\sigma_t$ calculation. This happens when $1.9 - A_i + 3.5(A_i - 0.8)^2 = 0$. The discriminant of the quadratic $3.5A_i^2 - 5.3A_i + 1.28$ is $5.3^2 - 4 \cdot 3.5 \cdot 1.28 = 28.09 - 17.92 = 10.17 > 0$, so real roots exist. Roots are $\frac{5.3 \pm \sqrt{10.17}}{7} \approx \frac{5.3 \pm 3.19}{7}$, yielding $A_i \approx 1.2$ and $A_i \approx 0.3$. If the medium color channel is near $0.3$ or $1.2$, $s_i$ approaches zero, leading to numerical instability. Clamping or handling this singularity is required [G].
- The polynomial for $\alpha$ is valid for $A \in [0, 1]$. Values outside this range may produce albedos outside $[0, 1]$, which is physically invalid for scattering albedo [G].

## Subtopics

### Numerical Albedo Inversion
A two-stage curve fitting process to invert the mapping from scattering albedo and anisotropy to surface color, enabling artist control over surface color in the path traced model [S2, unofficial]. It differs from the standard conversion by solving the inverse problem: given a desired surface color $x$ and anisotropy $g$, it computes the required scattering albedo $\alpha$ [S2, unofficial].

**Definition:**
Inverts the relationship $\alpha, g \rightarrow x$ to $x, g \rightarrow \alpha$ using a parametric function fitted to 1600 rendered images of a single slab geometry [S2, unofficial].

**Steps:**
1. Render images of a single slab geometry with varying scattering albedo $\alpha$ (40 values) and anisotropy $g$ (40 values), resulting in 1600 combinations [S2, unofficial].
2. Fit a parametric function to map surface color $x$ to albedo $\alpha$ using a linear blend of two arctan functions [S2, unofficial].
3. Fit coefficients $A, C, D, F$ as polynomials of $g$, and $B, E$ as exponential-of-polynomials of $g$ [S2, unofficial].

**Formula:**
$$ \alpha(x, g) = (1 - x^{0.25}) \cdot A(g) \cdot \tan^{-1}(B(g)x)^{C(g)} + x^{0.25} \cdot D(g) \cdot \tan^{-1}(E(g)x)^{F(g)} $$
where:
$$ A(g) = \sum_{i=0}^{7} a_i g^i $$
$$ B(g) = a_0 + a_1 \exp\left(\sum_{i=2}^{8} a_i g^{i-1}\right) $$
(Note: The source specifies $B(g) = a_0 + a_1 \exp(a_2 g + a_3 g^2 + \dots + a_8 g^7)$) [S2, unofficial].

**When to choose:**
Use when the artist specifies a target surface color and anisotropy, and the system needs to determine the underlying volume scattering albedo to achieve that appearance [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ per voxel/point, involving a few polynomial evaluations and arithmetic operations [S1, unofficial].
- Memory: $O(1)$, storing only the input parameters and output coefficients [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Scales linearly with the number of voxels or points in the volume, as the conversion is applied independently per location [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Provides a direct, closed-form conversion from intuitive artistic parameters to physical coefficients, facilitating optimization in differentiable rendering [S1, unofficial].
- Limitations: The approximation is specific to homogeneous media and may not accurately represent complex anisotropic scattering or non-homogeneous media without extension [S1, unofficial]. The singularity at specific albedo values requires careful handling [G].

**When to use:**
Use when integrating artistic subsurface scattering parameters into a differentiable volume renderer that requires physical extinction and scattering coefficients [S1, unofficial].

**When not:**
Do not use when the medium is highly anisotropic and requires a full phase function parameterization, or when the medium is non-homogeneous and requires spatially varying coefficients derived from more complex models [S1, unofficial].

## Background
Originated from the work of Chiang et al. in the context of volumetric path tracing and subsurface scattering parameterization [S1, unofficial]. The method was developed to bridge the gap between artistic control and physical simulation in volume rendering, allowing artists to specify medium color and scattering distance while the renderer computes the appropriate physical coefficients [S1, unofficial]. It is used in differentiable volume rendering pipelines where gradient-based optimization of material parameters is required [S1, unofficial].

## Sources
[S1] volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S2] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
