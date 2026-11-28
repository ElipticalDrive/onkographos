# Hermite Interpolation for Isosurface Extraction

Also called: Hermite contouring, Hermite interpolation

## Overview
Hermite interpolation for isosurface extraction determines the precise position of an isovertex along a zero-crossing edge by utilizing both scalar field values and gradient information (Hermite data) at the edge endpoints [S1]. Unlike [Linear Edge Intersection](868-linear-edge-intersection.md), which assumes linear variation and produces artifacts on non-linear implicit functions, this method fits higher-order polynomials to the local field behavior, significantly improving surface accuracy [S1]. It is primarily used in contouring algorithms like Marching Cubes where gradient data is available or can be approximated [S1].

## How it works

### Inputs and Outputs
**Inputs:** Edge endpoints $\mathbf{x}_0, \mathbf{x}_1 \in \mathbb{R}^3$, scalar field values $v_0 = F(\mathbf{x}_0)$ and $v_1 = F(\mathbf{x}_1)$, gradient vectors $\nabla F_0 = \nabla F(\mathbf{x}_0)$ and $\nabla F_1 = \nabla F(\mathbf{x}_1)$, and target isovalue $T$ (typically 0) [S1].
**Outputs:** Isovertex position $\mathbf{x}_{iso}$ along the edge such that $F(\mathbf{x}_{iso}) \approx T$ [S1].

### Data Structures
Not given in the sources.

### Steps
1. **Normalize the Edge Interval:** Map the physical edge segment $[\mathbf{x}_0, \mathbf{x}_1]$ to the parametric interval $[0, 1]$ by defining $e = \mathbf{x}_1 - \mathbf{x}_0$ and $L = \|e\|$ [S1].
2. **Compute Directional Derivatives:** Calculate the derivative of the scalar field along the edge direction at each endpoint as $f'_0 = \frac{\nabla F_0 \cdot e}{\|e\|^2}$ and $f'_1 = \frac{\nabla F_1 \cdot e}{\|e\|^2}$ [S1]. If the edge is axis-aligned, $f'$ is simply the corresponding component of $\nabla F$ [S1].
3. **Scale Derivatives for Unit Interval:** Scale the directional derivatives by the edge length to obtain derivatives for the normalized interval: $d_0 = f'_0 \cdot L$ and $d_1 = f'_1 \cdot L$ [S1].
4. **Adjust Values for Target Isovalue:** Shift the function values to target the zero-crossing: $v_0 \leftarrow v_0 - T$ and $v_1 \leftarrow v_1 - T$ [S1].
5. **Select Interpolation Order and Solve:** Choose between Cubic or Quadratic Hermite interpolation based on the implicit function's nature [S1].
   - **Cubic Hermite Interpolation:** Fit $p(t) = a_0 + a_1 t + a_2 t^2 + a_3 t^3$ satisfying $p(0)=v_0, p'(0)=d_0, p(1)=v_1, p'(1)=d_1$ [S1]. The coefficients are uniquely determined as $a_0 = v_0$, $a_1 = d_0$, $a_2 = 3v_1 - 3v_0 - 2d_0 - d_1$, and $a_3 = 2v_0 - 2v_1 + d_0 + d_1$ [S1].
   - **Quadratic Hermite Interpolation:** Use when the implicit function is known to be quadratic (e.g., PSR with B-splines) [S1]. Fit $p(t) = a_0 + a_1 t + a_2 t^2$ by interpolating values exactly ($a_0 = v_0$, $a_0 + a_1 + a_2 = v_1$) and fitting derivatives in a least-squares sense [S1]. The coefficients are $a_1 = \frac{d_0 - d_1}{2} + v_1 - v_0$ and $a_2 = \frac{d_1 - d_0}{2}$ [S1].
6. **Compute Isovertex Position:** Solve $p(t) = 0$ for $t \in [0, 1]$ and compute $\mathbf{x}_{iso} = \mathbf{x}_0 + t \cdot e$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def cbrt(x):
    if x >= 0:
        return x ** (1/3)
    else:
        return -((-x) ** (1/3))

def hermite_isovertex(x0, x1, v0, v1, grad0, grad1, T=0.0):
    """
    Calculate isovertex position using Hermite interpolation.
    
    Args:
        x0, x1: Endpoints of the edge (vectors).
        v0, v1: Scalar field values at endpoints.
        grad0, grad1: Gradient vectors at endpoints.
        T: Target isovalue.
        
    Returns:
        x_iso: Position of the isovertex.
    """
    # 1. Edge vector and length
    e = [x1[i] - x0[i] for i in range(3)]
    L_sq = dot(e, e)
    
    # Handle degenerate edge
    if L_sq < 1e-12:
        return x0
        
    L = math.sqrt(L_sq)
    
    # 2. Directional derivatives (projected onto edge direction)
    # f' = (grad . e) / ||e||^2
    df0 = dot(grad0, e) / L_sq
    df1 = dot(grad1, e) / L_sq
    
    # 3. Scale derivatives for unit interval [0, 1]
    # d = df * L
    d0 = df0 * L
    d1 = df1 * L
    
    # 4. Adjust values for target isovalue
    v0_adj = v0 - T
    v1_adj = v1 - T
    
    # 5. Solve for t
    # Using Cubic Hermite as the general robust method
    a0 = v0_adj
    a1 = d0
    a2 = 3 * v1_adj - 3 * v0_adj - 2 * d0 - d1
    a3 = 2 * v0_adj - 2 * v1_adj + d0 + d1
    
    # Solve cubic a3*t^3 + a2*t^2 + a1*t + a0 = 0
    t = solve_cubic_root_in_range(a3, a2, a1, a0, 0.0, 1.0)
    
    if t is None:
        # Fallback to linear if cubic fails to find root in range
        # Linear interpolation: t = -v0 / (v1 - v0)
        denom = v1_adj - v0_adj
        if abs(denom) < 1e-12:
            return x0
        t = -v0_adj / denom
        t = max(0.0, min(1.0, t))
        
    # 6. Compute position
    x_iso = [x0[i] + t * e[i] for i in range(3)]
    return x_iso

def solve_cubic_root_in_range(a3, a2, a1, a0, low, high):
    """
    Finds a real root of a3*t^3 + a2*t^2 + a1*t + a0 = 0 within [low, high].
    Returns the root or None if no root exists in range.
    """
    # Normalize if a3 is 0 (quadratic)
    if abs(a3) < 1e-12:
        if abs(a2) < 1e-12:
            # Linear: a1*t + a0 = 0
            if abs(a1) > 1e-12:
                t = -a0 / a1
                if low <= t <= high:
                    return t
            return None
        else:
            # Quadratic: a2*t^2 + a1*t + a0 = 0
            disc = a1*a1 - 4*a2*a0
            if disc < 0:
                return None
            sqrt_disc = math.sqrt(disc)
            t1 = (-a1 + sqrt_disc) / (2 * a2)
            t2 = (-a1 - sqrt_disc) / (2 * a2)
            if low <= t1 <= high: return t1
            if low <= t2 <= high: return t2
            return None
            
    # Cubic case
    # Convert to depressed cubic u^3 + pu + q = 0
    # t = u - a2/(3*a3)
    p = (3*a3*a1 - a2*a2) / (3*a3*a3)
    q = (2*a2*a2*a2 - 9*a3*a2*a1 + 27*a3*a3*a0) / (27*a3*a3*a3)
    
    # Discriminant
    delta = (q*q/4 + p*p*p/27)
    
    if delta > 0:
        # One real root
        sqrt_delta = math.sqrt(delta)
        u = cbrt(-q/2 + sqrt_delta) + cbrt(-q/2 - sqrt_delta)
        t = u - a2 / (3 * a3)
        if low <= t <= high:
            return t
    elif delta == 0:
        # Multiple roots
        if p == 0 and q == 0:
            t = -a2 / (3 * a3) # Triple root
            if low <= t <= high: return t
        else:
            u = cbrt(-q/2)
            t1 = 2*u - a2/(3*a3)
            t2 = -u - a2/(3*a3)
            if low <= t1 <= high: return t1
            if low <= t2 <= high: return t2
    else:
        # Three real roots (casus irreducibilis)
        r = math.sqrt(-p*p*p/27)
        theta = math.acos(-q / (2 * r))
        m = 2 * math.cbrt(r)
        t1 = m * math.cos(theta/3) - a2/(3*a3)
        t2 = m * math.cos((theta + 2*math.pi)/3) - a2/(3*a3)
        t3 = m * math.cos((theta + 4*math.pi)/3) - a2/(3*a3)
        
        candidates = [t1, t2, t3]
        for t in candidates:
            if low <= t <= high:
                return t
                
    return None
```

### Parameters
- **Target Isovalue ($T$):** Typically 0 [S1].
- **Interpolation Order:** Cubic is the default for general implicit functions; Quadratic is used for known quadratic fields (e.g., B-splines) [S1].

### Edge Cases and Pitfalls
- **Degenerate Edges:** If $\|\mathbf{x}_0 - \mathbf{x}_1\| \approx 0$, the derivative calculation involves division by zero [G].
- **Multiple Roots:** A cubic polynomial can have up to three real roots [S1]. To uniquely define the position of an isovertex, we observe that there must be an odd number of roots along a zero-crossing edge [S1].
- **Least-Squares Pitfall:** Do not use the standard least-squares solution for all four constraints (values and derivatives) in the quadratic case, because it does not guarantee that the polynomial $p(t)$ has a root along the edge even if $F(t)$ does [S1]. The value-interpolating, derivative-least-squares approach is preferred [S1].
- **Gradient Availability:** Hermite data (gradients) may not be available in all datasets; the approach remains effective even when gradients are approximated using finite differences [S1].

## Subtopics

### [Cubic Hermite Interpolation](13715-cubic-hermite-interpolation.md)
Cubic Hermite Interpolation fits a cubic polynomial to the four Hermite constraints (two values, two derivatives). It is the general-purpose method for non-linear implicit functions, offering higher accuracy than linear interpolation by reducing artifacts. It requires solving a cubic equation, which may yield up to three roots, necessitating a strategy to select the correct isovertex. Choose this when the implicit function is not known to be quadratic and high accuracy is required. See [Cubic Hermite Interpolation](13715-cubic-hermite-interpolation.md).

### Quadratic Hermite Interpolation (Least-Squares Derivatives)
This variant fits a quadratic polynomial that interpolates the endpoint values exactly and fits the endpoint derivatives in a least-squares sense [S1]. It is specifically designed for implicit functions known to be quadratic, such as those represented by linear combinations of second-order B-splines (e.g., PSR) [S1]. It is computationally cheaper than cubic interpolation (solving a quadratic vs. cubic equation) [G] and avoids the multiple-root ambiguity of cubics [S1]. Choose this when the underlying field is known to be quadratic [S1].

## Operating characteristics

- **Time Complexity:** $O(1)$ per edge. Cubic interpolation involves solving a cubic equation (constant number of operations) [G]. Quadratic interpolation involves solving a quadratic equation (constant number of operations) [G].
- **Memory Complexity:** $O(1)$ per edge. No additional storage is required beyond the input values and gradients [G].
- **Measured Performance:** Not given in the sources [G].
- **Scaling:** Scales linearly with the number of edges in the mesh [G].
- **Strengths:** Significantly reduces reconstruction artifacts compared to linear interpolation for non-linear implicit functions [S1]. Provides mathematically precise isovertex positions given gradient data [S1].
- **Limitations:** Requires gradient data (Hermite data), which may need to be computed via finite differences if not available, adding overhead and potential noise [S1]. Cubic interpolation can have multiple roots, requiring careful selection logic [S1].
- **When to Use:** When high-quality isosurfaces are required for non-linear implicit functions and gradient information is available or can be efficiently approximated [S1].
- **When Not to Use:** When gradient data is unavailable and finite difference approximation is too expensive or noisy, or when the implicit function is strictly linear (linear interpolation is sufficient and faster) [G].

## Background

Hermite interpolation for isosurface extraction was proposed to address the inaccuracies of linear interpolation in contouring algorithms like Marching Cubes when applied to non-linear implicit functions [S1]. The method utilizes the gradient of the implicit function to generate more accurate isosurfaces by means of higher-order interpolation techniques [S1]. The approach was demonstrated to provide clear quality improvements, even when Hermite data is not directly available and gradients are approximated using finite differences [S1].

## Sources

[S1] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
