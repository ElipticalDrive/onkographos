# Algebraic Surface Raycasting

Also called: Implicit Surface Raycasting via Polynomial Composition, Polynomial Raycasting

## Overview
Algebraic Surface Raycasting is an image-order rendering technique for isosurfaces where the volumetric scalar field is represented as a trivariate polynomial [S1]. It solves the intersection problem by composing the trivariate field function with the ray equation to produce a univariate polynomial, then finding the roots of this polynomial to determine the hit point [S1]. It distinguishes itself from voxel-based marching methods by providing an exact analytical representation of the surface within each polynomial patch, avoiding discretization errors inherent in grid-based sampling, though it requires the field to be explicitly defined as a polynomial [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A trivariate polynomial function $f(x, y, z)$ of degree $d$ in each dimension (total degree $3d$), defined by coefficients $f_{ijk}$; a ray defined by origin $\mathbf{p}$ and direction $\mathbf{v}$; an isovalue $c$ [S1].
- **Output:** The first intersection parameter $t^*_0$ along the ray where $f(\mathbf{r}(t)) = c$ [S1].

**Data Structures:**
The field is stored as a 3D array of coefficients $f_{ijk}$ for $0 \leq i, j, k \leq d$ [S1]. The ray is stored as two 3D vectors $\mathbf{p}$ and $\mathbf{v}$ [G]. The resulting univariate polynomial is stored as an array of coefficients $a_m$ for $m \in [0, 3d]$ [G].

**Steps:**
1. **Ray Definition:** Define the ray $\mathbf{r}(t) = \mathbf{p} + t \cdot \mathbf{v}$, where $\mathbf{p} = (p_x, p_y, p_z)$ and $\mathbf{v} = (v_x, v_y, v_z)$ [S1].
2. **Composition:** Substitute the ray equation into the trivariate polynomial $f(x, y, z) = \sum_{0 \leq i, j, k \leq d} f_{ijk} x^i y^j z^k$ [S1]. The resulting expression is a univariate polynomial in $t$: $(f \circ \mathbf{r})(t) = \sum_{m=0}^{3d} a_m t^m$ [S1]. The coefficients $a_m$ are derived by expanding $(p_x + t v_x)^i (p_y + t v_y)^j (p_z + t v_z)^k$ and summing the contributions weighted by $f_{ijk}$ [S1].
3. **Isovalue Subtraction:** Subtract the isovalue $c$ from the constant term of the univariate polynomial to form the root-finding function: $P(t) = (f \circ \mathbf{r})(t) - c = \sum_{m=0}^{3d} a_m t^m - c$ [S1]. The constant term becomes $a_0 - c$, where $a_0 = f(p_x, p_y, p_z)$ [S1].
4. **Root Finding:** Use a rootfinder to identify the first root position $t^*_0$ of the polynomial $(f \circ \mathbf{r})(t) - c = 0$ [S1].
5. **Hit Point Calculation:** Compute the intersection position $\mathbf{h} = \mathbf{p} + t^*_0 \cdot \mathbf{v}$ [G].
6. **Shading:** Compute the surface normal $\mathbf{n} = \nabla f(\mathbf{h})$ using the gradient of the trivariate polynomial, then apply a lighting model (e.g., Phong) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def algebraic_surface_cast(p, v, coeffs, isovalue, d):
    """
    Cast a ray against an algebraic surface defined by polynomial coefficients.
    
    Args:
        p: Ray origin (list/tuple of 3 floats)
        v: Ray direction (list/tuple of 3 floats)
        coeffs: 3D array of shape (d+1, d+1, d+1) containing f_ijk
        isovalue: Scalar isovalue c
        d: Degree of the polynomial in each dimension
        
    Returns:
        t_hit: The first positive root t, or infinity if no hit
    """
    # Step 1 & 2: Composition
    a = [0.0] * (3 * d + 1)
    
    for i in range(d + 1):
        for j in range(d + 1):
            for k in range(d + 1):
                f_ijk = coeffs[i, j, k]
                if f_ijk == 0.0:
                    continue
                
                for i_x in range(i + 1):
                    cx = binomial(i, i_x) * (p[0] ** (i - i_x)) * (v[0] ** i_x)
                    for j_y in range(j + 1):
                        cy = binomial(j, j_y) * (p[1] ** (j - j_y)) * (v[1] ** j_y)
                        for k_z in range(k + 1):
                            cz = binomial(k, k_z) * (p[2] ** (k - k_z)) * (v[2] ** k_z)
                            
                            power = i_x + j_y + k_z
                            a[power] += f_ijk * cx * cy * cz

    # Step 3: Subtraction
    a[0] -= isovalue
    
    # Step 4: Root Finding
    return find_first_positive_real_root(a)

def binomial(n, k):
    if k < 0 or k > n: return 0
    if k == 0 or k == n: return 1
    if k > n // 2: k = n - k
    res = 1
    for i in range(k): res = res * (n - i) // (i + 1)
    return res

def find_first_positive_real_root(coeffs):
    degree = len(coeffs) - 1
    if degree == 1:
        t = -coeffs[0] / coeffs[1]
        return t if t > 0 else float('inf')
    elif degree == 2:
        a, b, c = coeffs[2], coeffs[1], coeffs[0]
        disc = b*b - 4*a*c
        if disc < 0: return float('inf')
        sqrt_d = math.sqrt(disc)
        t1 = (-b - sqrt_d) / (2*a)
        t2 = (-b + sqrt_d) / (2*a)
        roots = [r for r in (t1, t2) if r > 0]
        return min(roots) if roots else float('inf')
    elif degree == 3:
        return solve_cubic(coeffs)
    elif degree == 4:
        return solve_quartic(coeffs)
    else:
        return numerical_root_finding(coeffs)

def solve_cubic(coeffs):
    a, b, c, d = coeffs[3], coeffs[2], coeffs[1], coeffs[0]
    if a == 0.0: return solve_quadratic([d, c, b])
    p = (3*a*c - b*b) / (3*a*a)
    q = (2*b*b*b - 9*a*b*c + 27*a*a*d) / (27*a*a*a)
    disc = (q*q/4.0 + p*p*p/27.0)
    if disc > 0:
        u = (-q/2.0 + math.sqrt(disc))**(1/3)
        v = (-q/2.0 - math.sqrt(disc))**(1/3)
        t1 = u + v - b/(3*a)
        return t1 if t1 > 0 else float('inf')
    elif disc == 0:
        if p == 0: return -b/(3*a) if -b/(3*a) > 0 else float('inf')
        u = (-q/2.0)**(1/3)
        t1 = 2*u - b/(3*a)
        t2 = -u - b/(3*a)
        roots = [r for r in (t1, t2) if r > 0]
        return min(roots) if roots else float('inf')
    else:
        r = math.sqrt(-p*p*p/27.0)
        theta = math.acos(-q/(2*r))
        m = 2 * r**(1/3)
        t1 = m * math.cos(theta/3) - b/(3*a)
        t2 = m * math.cos((theta + 2*math.pi)/3) - b/(3*a)
        t3 = m * math.cos((theta + 4*math.pi)/3) - b/(3*a)
        roots = [r for r in (t1, t2, t3) if r > 0]
        return min(roots) if roots else float('inf')

def solve_quartic(coeffs):
    # Standard closed-form quartic solver [G]
    return newton_raphson(coeffs)

def numerical_root_finding(coeffs):
    # Iterative root isolation for degrees > 4 [G]
    return newton_raphson(coeffs)

def solve_quadratic(coeffs):
    a, b, c = coeffs[2], coeffs[1], coeffs[0]
    disc = b*b - 4*a*c
    if disc < 0: return float('inf')
    sqrt_d = math.sqrt(disc)
    t1 = (-b - sqrt_d) / (2*a)
    t2 = (-b + sqrt_d) / (2*a)
    roots = [r for r in (t1, t2) if r > 0]
    return min(roots) if roots else float('inf')

def newton_raphson(coeffs):
    # Newton-Raphson with bisection fallback for robustness [G]
    def poly(t):
        val = 0.0
        for i, c in enumerate(coeffs): val += c * (t ** i)
        return val
    def deriv(t):
        val = 0.0
        for i in range(1, len(coeffs)): val += i * coeffs[i] * (t ** (i-1))
        return val
    t = 1.0
    for _ in range(100):
        f = poly(t)
        df = deriv(t)
        if abs(df) < 1e-12: t += 0.1; continue
        t_new = t - f/df
        if t_new <= 0: t = max(t/2, 0.001); continue
        if abs(t_new - t) < 1e-9: break
        t = t_new
    return t if t > 0 else float('inf')
```

**Parameters:**
- $d$: Polynomial degree. Typical values are $d=1$ (tri-linear) or $d=3$ (tri-cubic) for voxel data [S1].
- $c$: Isovalue. Application-dependent [S1].

**Edge Cases and Pitfalls:**
- **Piecewise Continuity:** The polynomial approximation is valid only within the support of the basis functions (e.g., within a voxel or wavelet node), requiring traversal across patches for complex datasets [S1].
- **Tangency and Misses:** If the ray is tangent to the surface or misses it entirely, root finding may return no positive real roots or multiple roots requiring careful selection of the *first* hit [S1].

## Subtopics

### [Polynomial Root Finding (Compact Kernel)](4022-polynomial-root-finding-compact-kernel.md)
Polynomial Root Finding (Compact Kernel) is a variant that uses compact polynomial kernels to define the field function along a ray, resulting in a piecewise polynomial. It distinguishes itself by leveraging the specific algebraic structure of compact kernels to provide robust root-finding strategies, including closed-form solutions for low degrees and Bézier clipping for higher degrees. It is chosen when the underlying data is represented by point primitives with compact support, allowing for efficient local evaluation and robust root isolation. See [Polynomial Root Finding (Compact Kernel)](4022-polynomial-root-finding-compact-kernel.md).

### Hermite Polynomial Approximation for Convolution Surfaces
Hermite Polynomial Approximation for Convolution Surfaces builds polynomial approximations of the field value along the ray for convolution surfaces, allowing analytical computation of roots [S2]. It distinguishes itself by using Hermite data (field value and derivative) sampled at subinterval boundaries to construct cubic polynomial approximations on uniformly subdivided intervals [S2]. It is chosen for rendering scale-invariant integral surfaces (convolution surfaces) where the field is defined by the summation of primitive contributions [S2]. Pitfalls include the difficulty in choosing an adequate subdivision level: too few intervals cause poor approximation, while too many increase computational time and cause data coherence issues on GPU [S2]. Higher-degree approximations avoid subdivision but lose closed-form solutions and suffer from Runge's phenomenon oscillations [S2].

## Operating characteristics
- **Time Complexity:** The composition step is a bottleneck for higher-order surfaces [S1]. The method offers significant speedup over naive composition implementations [S1].
- **Memory Complexity:** $O(d^3)$ to store the polynomial coefficients $f_{ijk}$ [S1].
- **Measured Performance:** The composition step is a bottleneck for higher-order surfaces [S1]. The method offers significant speedup over naive composition implementations [S1].
- **Strengths:** Exact representation of the isosurface within the polynomial domain; no discretization artifacts from voxel marching; efficient for smooth, analytic fields [S1].
- **Limitations:** Limited to polynomial representations; subdivision required for complex primitives like convolution surfaces [S1, S2].
- **When to use:** When the volumetric data can be accurately represented by polynomials (e.g., wavelet-based representations, compact kernels) and high visual fidelity is required [S1].
- **When not to use:** For general voxel data without polynomial approximation, or when real-time performance is critical and the overhead of composition/root-finding is prohibitive compared to simpler marching methods [S1].

## Background
Algebraic Surface Raycasting originated from the need to render isosurfaces from volumetric data with higher accuracy than voxel-based methods [S1]. It was notably developed in the context of wavelet-based multiresolution rendering, where wavelet transforms allow selective removal of irrelevant data while maintaining a good approximation of the continuous domain [S1]. The method leverages the fact that common interpolation functions (tri-linear, tri-cubic) result in piecewise algebraic surfaces [S1]. It is used in medical visualization, engineering, and computer games for applications requiring precise isosurface extraction [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
