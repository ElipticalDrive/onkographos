# Algebraic Surface Raycasting

Also called: Implicit Surface Raycasting via Polynomial Composition, Polynomial Raycasting

## Overview
Algebraic Surface Raycasting is an image-order rendering technique for isosurfaces where the volumetric scalar field is represented as a trivariate polynomial. It solves the intersection problem by composing the trivariate field function with the ray equation to produce a univariate polynomial, then finding the roots of this polynomial to determine the hit point [S1]. It distinguishes itself from voxel-based marching methods by providing an exact analytical representation of the surface within each polynomial patch, avoiding discretization errors inherent in grid-based sampling, though it requires the field to be explicitly defined as a polynomial [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A trivariate polynomial function $f(x, y, z)$ of degree $d$ in each dimension (total degree $3d$), defined by coefficients $f_{ijk}$; a ray defined by origin $\mathbf{p}$ and direction $\mathbf{v}$; an isovalue $c$.
- **Output:** The first intersection parameter $t^*_0$ along the ray where $f(\mathbf{r}(t)) = c$.

**Data Structures:**
The field is stored as a 3D array of coefficients $f_{ijk}$ for $0 \leq i, j, k \leq d$. The ray is stored as two 3D vectors $\mathbf{p}$ and $\mathbf{v}$. The resulting univariate polynomial is stored as an array of coefficients $a_m$ for $m \in [0, 3d]$.

**Steps:**
1.  **Ray Definition:** Define the ray $\mathbf{r}(t) = \mathbf{p} + t \cdot \mathbf{v}$, where $\mathbf{p} = (p_x, p_y, p_z)$ and $\mathbf{v} = (v_x, v_y, v_z)$ [S1].
2.  **Composition:** Substitute the ray equation into the trivariate polynomial $f(x, y, z) = \sum_{0 \leq i, j, k \leq d} f_{ijk} x^i y^j z^k$ [S1].
    The resulting expression is a univariate polynomial in $t$:
    $$ (f \circ \mathbf{r})(t) = \sum_{m=0}^{3d} a_m t^m $$
    The coefficients $a_m$ are derived by expanding $(p_x + t v_x)^i (p_y + t v_y)^j (p_z + t v_z)^k$ and summing the contributions weighted by $f_{ijk}$ [S1].
3.  **Isovalue Subtraction:** Subtract the isovalue $c$ from the constant term of the univariate polynomial to form the root-finding function:
    $$ P(t) = (f \circ \mathbf{r})(t) - c = \sum_{m=0}^{3d} a_m t^m - c $$
    Note that the constant term becomes $a_0 - c$, where $a_0 = f(p_x, p_y, p_z)$ [S1].
4.  **Root Finding:** Find the smallest positive real root $t^*_0$ of $P(t) = 0$.
    - For degrees $\leq 4$, closed-form solutions (quadratic, cubic, quartic formulas) may be used [S1].
    - For higher degrees, numerical methods such as Bézier clipping are employed to isolate and converge to the first root [S1].
5.  **Hit Point Calculation:** Compute the intersection position $\mathbf{h} = \mathbf{p} + t^*_0 \cdot \mathbf{v}$ [S1].
6.  **Shading:** Compute the surface normal $\mathbf{n} = \nabla f(\mathbf{h})$ using the gradient of the trivariate polynomial, then apply a lighting model (e.g., Phong) [S1, S3, S6].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def algebraic_surface_cast(p, v, coeffs, isovalue, max_degree):
    """
    Cast a ray against an algebraic surface defined by polynomial coefficients.
    
    Args:
        p: Ray origin (vec3)
        v: Ray direction (vec3)
        coeffs: 3D array of shape (d+1, d+1, d+1) containing f_ijk
        isovalue: Scalar isovalue c
        max_degree: Degree d of the polynomial in each dimension
        
    Returns:
        t_hit: The first positive root t, or infinity if no hit
    """
    d = max_degree
    
    # Step 1 & 2: Composition
    # Initialize univariate polynomial coefficients a_m for m in 0..3d
    a = [0.0] * (3 * d + 1)
    
    # Iterate over all trivariate coefficients
    for i in range(d + 1):
        for j in range(d + 1):
            for k in range(d + 1):
                f_ijk = coeffs[i, j, k]
                if f_ijk == 0:
                    continue
                
                # Expand (px + tx)^i * (py + ty)^j * (pz + tz)^k
                # This results in a polynomial of degree i+j+k
                # We need to add f_ijk * this_poly to the global sum a
                
                # Precompute binomial coefficients or compute on fly
                # For efficiency, one might use Horner's method or precomputed powers,
                # but here we expand explicitly for clarity.
                
                # Term x^i: sum_{a=0}^i C(i,a) * px^(i-a) * vx^a * t^a
                # Term y^j: sum_{b=0}^j C(j,b) * py^(j-b) * vy^b * t^b
                # Term z^k: sum_{c=0}^k C(k,c) * pz^(k-c) * vz^c * t^c
                
                # Combine to get coefficient for t^(a+b+c)
                for a_idx in range(i + 1):
                    coeff_x = binomial(i, a_idx) * (p.x ** (i - a_idx)) * (v.x ** a_idx)
                    for b_idx in range(j + 1):
                        coeff_y = binomial(j, b_idx) * (p.y ** (j - b_idx)) * (v.y ** b_idx)
                        for c_idx in range(k + 1):
                            coeff_z = binomial(k, c_idx) * (p.z ** (k - c_idx)) * (v.z ** c_idx)
                            
                            power_t = a_idx + b_idx + c_idx
                            term_val = coeff_x * coeff_y * coeff_z * f_ijk
                            a[power_t] += term_val

    # Step 3: Subtraction
    a[0] -= isovalue
    
    # Step 4: Root Finding
    # Find smallest positive real root of sum(a[m] * t^m) = 0
    t_hit = find_first_positive_real_root(a)
    
    return t_hit

def binomial(n, k):
    """Compute binomial coefficient C(n, k)."""
    if k < 0 or k > n:
        return 0
    if k == 0 or k == n:
        return 1
    if k > n // 2:
        k = n - k
    
    res = 1
    for i in range(k):
        res = res * (n - i) // (i + 1)
    return res

def find_first_positive_real_root(coeffs):
    """
    Find the smallest positive real root of the polynomial defined by coeffs.
    Uses closed-form for degree <= 4, else numerical method.
    """
    degree = len(coeffs) - 1
    
    if degree == 1:
        # a1*t + a0 = 0 => t = -a0/a1
        if abs(coeffs[1]) < 1e-9:
            return float('inf')
        t = -coeffs[0] / coeffs[1]
        return t if t > 0 else float('inf')
        
    elif degree == 2:
        # a2*t^2 + a1*t + a0 = 0
        a, b, c = coeffs[2], coeffs[1], coeffs[0]
        disc = b*b - 4*a*c
        if disc < 0:
            return float('inf')
        sqrt_disc = math.sqrt(disc)
        t1 = (-b - sqrt_disc) / (2*a)
        t2 = (-b + sqrt_disc) / (2*a)
        roots = [r for r in [t1, t2] if r > 0]
        return min(roots) if roots else float('inf')
        
    elif degree == 3:
        # Cubic formula implementation omitted for brevity, 
        # but standard closed-form exists [S1].
        return solve_cubic(coeffs)
        
    elif degree == 4:
        # Quartic formula implementation omitted for brevity,
        # standard closed-form exists [S1].
        return solve_quartic(coeffs)
        
    else:
        # Higher degrees: Use numerical method like Bézier clipping [S1]
        return numerical_root_finding(coeffs)
```

**Parameters:**
- $d$: Polynomial degree. Typical values are $d=1$ (tri-linear) or $d=3$ (tri-cubic) for voxel data [S1].
- $c$: Isovalue. Application-dependent.

**Edge Cases and Pitfalls:**
- **Precision:** High-degree polynomials can suffer from numerical instability during coefficient calculation and root finding [S1].
- **Boundary Conditions:** The polynomial approximation is valid only within the support of the basis functions (e.g., within a voxel or wavelet node). Outside this domain, the field definition may be discontinuous or undefined [S1].
- **Degenerate Input:** If the ray is tangent to the surface or misses it entirely, root finding may return no positive real roots or multiple roots requiring careful selection of the *first* hit [S1].

## Subtopics

### [Polynomial Root Finding (Compact Kernel)](4022-polynomial-root-finding-compact-kernel.md)
Polynomial Root Finding (Compact Kernel) is a variant that uses compact polynomial kernels to define the field function along a ray, resulting in a piecewise polynomial. It distinguishes itself by leveraging the specific algebraic structure of compact kernels to provide robust root-finding strategies, including closed-form solutions for low degrees and Bézier clipping for higher degrees. It is chosen when the underlying data is represented by point primitives with compact support, allowing for efficient local evaluation and robust root isolation. See [Polynomial Root Finding (Compact Kernel)](4022-polynomial-root-finding-compact-kernel.md).

### Hermite Polynomial Approximation for Convolution Surfaces
Hermite Polynomial Approximation for Convolution Surfaces builds polynomial approximations of the field value along the ray for convolution surfaces, allowing analytical computation of roots [S2]. It distinguishes itself by using Hermite data (field value and derivative) sampled at subinterval boundaries to construct cubic polynomial approximations on uniformly subdivided intervals [S2]. It is chosen for rendering scale-invariant integral surfaces (convolution surfaces) where the field is defined by the summation of primitive contributions [S2]. Pitfalls include the difficulty in choosing an adequate subdivision level: too few intervals cause poor approximation, while too many increase computational time and cause data coherence issues on GPU [S2]. Higher-degree approximations avoid subdivision but lose closed-form solutions and suffer from Runge's phenomenon oscillations [S2].

## Operating characteristics
- **Time Complexity:** Composition is $O(d^3)$ per ray if implemented naively by expanding all terms, but can be optimized. Root finding is $O(1)$ for low degrees (closed-form) or iterative for higher degrees.
- **Memory Complexity:** $O(d^3)$ to store the polynomial coefficients $f_{ijk}$ [S1].
- **Measured Performance:** The composition step is a bottleneck for higher-order surfaces [S1]. The method offers significant speedup over naive composition implementations [S1].
- **Strengths:** Exact representation of the isosurface within the polynomial domain; no discretization artifacts from voxel marching; efficient for smooth, analytic fields [S1].
- **Limitations:** Limited to polynomial representations; high-degree polynomials can be numerically unstable; subdivision required for complex primitives like convolution surfaces [S1, S2].
- **When to use:** When the volumetric data can be accurately represented by polynomials (e.g., wavelet-based representations, compact kernels) and high visual fidelity is required [S1].
- **When not to use:** For general voxel data without polynomial approximation, or when real-time performance is critical and the overhead of composition/root-finding is prohibitive compared to simpler marching methods [S1].

## Background
Algebraic Surface Raycasting originated from the need to render isosurfaces from volumetric data with higher accuracy than voxel-based methods [S1]. It was notably developed in the context of wavelet-based multiresolution rendering, where wavelet transforms allow selective removal of irrelevant data while maintaining a good approximation of the continuous domain [S1]. The method leverages the fact that common interpolation functions (tri-linear, tri-cubic) result in piecewise algebraic surfaces [S1]. It is used in medical visualization, engineering, and computer games for applications requiring precise isosurface extraction [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
