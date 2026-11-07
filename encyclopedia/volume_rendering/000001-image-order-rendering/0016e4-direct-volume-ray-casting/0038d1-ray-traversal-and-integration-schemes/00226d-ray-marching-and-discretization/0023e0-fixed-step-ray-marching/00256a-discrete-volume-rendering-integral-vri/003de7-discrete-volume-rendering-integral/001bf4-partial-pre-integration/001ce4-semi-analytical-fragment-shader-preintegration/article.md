# Semi-Analytical Fragment Shader Preintegration

Also called: Semi-Analytical Preintegration, Adaptive Preintegration with Extrema

## Overview
Semi-Analytical Fragment Shader Preintegration is a GPU-based rendering technique that improves the accuracy of volume ray casting by performing adaptive preintegration over intervals defined by the local extrema of scalar and shading polynomials [S1]. It solves the problem of aliasing and integration error inherent in fixed-step ray marching by analytically determining where the signal and shading functions change most rapidly, thereby concentrating computational effort where it is most needed [S1]. Unlike its parent method, Partial Pre-Integration, which uses a fixed 2D $\psi$ table independent of the transfer function, this method adapts the integration intervals based on the specific polynomial coefficients of the current cell, allowing for higher fidelity with complex transfer functions without recomputing large tables [S1].

## How it works
The method operates within a fragment shader, triggered after a geometry shader has instantiated the front faces of visible volume cells (e.g., AMR patches) [S1]. The core logic computes the exit point of the ray through the cell, derives polynomial coefficients for signal and shading, finds local extrema to define integration bounds, and performs preintegration over the resulting intervals [S1].

**Inputs and Outputs:**
*   **Input:** Ray origin and direction (implicit in fragment shader context), cell geometry indices, transfer function parameters (implicit in table lookup or shading calculation).
*   **Output:** Accumulated color and opacity for the pixel.

**Data Structures:**
*   **Preintegration Table:** A 256 $\times$ 256 $\times$ 32 RGBA texture. Each component is 16-bit floating point [S1]. The third dimension (size 32) stores the length of the preintegration interval. To maintain precision for small intervals, logarithmic table access is used for the length parameter [S1].
*   **Offscreen Buffer:** A 16-bit floating point buffer used for intermediate blending to ensure accuracy before copying to the front buffer [S1].

**Algorithm Steps:**
1.  **Find Exit Point:** Compute the ray exit point of the current cell [S1].
2.  **Compute Cubic Coefficients:** Calculate coefficients $a, b, c, d$ for the scalar signal polynomial $s(t) = at^3 + bt^2 + ct + d$ [S1]. These are derived from the cell's data as per the method's specific reconstruction scheme (Subsection 3.1 in source) [S1].
3.  **Compute Quadratic Coefficients:** Calculate coefficients $e, f, g$ for the shading polynomial (Subsection 3.2 in source) [S1].
4.  **Initialize Extrema List:** Initialize an empty set $T = \emptyset$ [S1].
5.  **Find Scalar Extrema:**
    *   If $a \neq 0$ and $b \neq 0$: Compute discriminant $\Delta = 4b^2 - 12ac$. Add roots of the derivative to $T$:
        $$ T = T \cup \left\{ \frac{-2b + \sqrt{\Delta}}{6a}, \frac{-2b - \sqrt{\Delta}}{6a} \right\} $$
    *   Else if $a = 0$ and $b \neq 0$: Add the single root to $T$:
        $$ T = T \cup \left\{ \frac{-c}{2b} \right\} $$
    *   Else: No extrema from the cubic term [S1].
6.  **Find Shading Extrema:**
    *   If $e \neq 0$: Add the root of the quadratic derivative to $T$:
        $$ T = T \cup \left\{ \frac{-f}{2e} \right\} $$
    *   Else: No extrema from the quadratic term [S1].
7.  **Prune and Bound:**
    *   For each $t \in T$, if $t \le 0$ or $t \ge 1$, remove $t$ from $T$ [S1].
    *   Add boundaries to $T$: $T = T \cup \{0, 1\}$ [S1].
8.  **Sort:** Sort the points in $T$ in ascending order [S1].
9.  **Adaptive Preintegration:**
    *   For each consecutive pair $(t_1, t_2)$ in $T$:
        *   Reconstruct scalar values at boundaries:
            $$ s_1 = d + t_1(c + t_1(b + t_1 a)) $$
            $$ s_2 = d + t_2(c + t_2(b + t_2 a)) $$
        *   Reconstruct shading values $l_1, l_2$ at boundaries (using the quadratic coefficients $e, f, g$) [S1].
        *   Compute interval length $l = t_2 - t_1$ [S1].
        *   Fetch preintegration textures using $s_1, s_2$ and the logarithmically mapped $l$ [S1].
        *   Combine the fetched texture values (front and back weighted tables) [S1].
        *   Accumulate the result [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fragment_shader_preintegration(ray_exit_t, scalar_coeffs, shading_coeffs):
    # scalar_coeffs: [a, b, c, d]
    # shading_coeffs: [e, f, g]
    
    a, b, c, d = scalar_coeffs
    e, f, g = shading_coeffs
    
    T = []
    
    # Find extrema of cubic scalar polynomial s(t) = at^3 + bt^2 + ct + d
    # Derivative s'(t) = 3at^2 + 2bt + c
    if a != 0 and b != 0:
        delta = 4 * b**2 - 12 * a * c
        if delta >= 0:
            sqrt_delta = math.sqrt(delta)
            t1 = (-2 * b + sqrt_delta) / (6 * a)
            t2 = (-2 * b - sqrt_delta) / (6 * a)
            T.append(t1)
            T.append(t2)
    elif a == 0 and b != 0:
        # Derivative is linear: 2bt + c = 0
        T.append(-c / (2 * b))
        
    # Find extrema of quadratic shading polynomial
    # Derivative is linear: 2et + f = 0
    if e != 0:
        T.append(-f / (2 * e))
        
    # Prune points outside [0, 1]
    T = [t for t in T if 0 < t < 1]
    
    # Add boundaries
    T.append(0.0)
    T.append(1.0)
    
    # Sort
    T.sort()
    
    accumulated_color = vec4(0.0)
    accumulated_opacity = 0.0
    
    for i in range(len(T) - 1):
        t1 = T[i]
        t2 = T[i+1]
        
        # Reconstruct scalar values
        s1 = d + t1 * (c + t1 * (b + t1 * a))
        s2 = d + t2 * (c + t2 * (b + t2 * a))
        
        # Reconstruct shading values (assuming l(t) = et^2 + ft + g)
        l1 = g + t1 * (f + t1 * e)
        l2 = g + t2 * (f + t2 * e)
        
        # Compute length
        length = t2 - t1
        
        # Logarithmic access for length precision
        # Note: Exact log mapping function is implementation-specific [S1]
        log_length = log_access(length)
        
        # Fetch preintegration tables
        # front_table and back_table are 256x256x32 textures
        front_val = fetch_texture(front_table, s1, s2, log_length)
        back_val = fetch_texture(back_table, s1, s2, log_length)
        
        # Combine and accumulate
        # Specific combination formula depends on the transfer function and blending model [S1]
        # This step typically involves alpha compositing the pre-integrated result
        accumulated_color, accumulated_opacity = accumulate(accumulated_color, accumulated_opacity, front_val, back_val, length)
        
    return accumulated_color
```

**Parameters:**
*   **Table Resolution:** 256 $\times$ 256 for signal values, 32 for length [S1].
*   **Precision:** 16-bit floating point for table components and offscreen buffer [S1].

**Edge Cases and Pitfalls:**
*   **Discriminant Negative:** If $\Delta < 0$ in the cubic case, there are no real extrema, and no points are added from that branch [S1].
*   **Zero Coefficients:** Care must be taken when $a=0$ or $e=0$ to avoid division by zero [S1].
*   **Precision at Small Lengths:** Standard linear lookup for length loses precision for small intervals; logarithmic mapping is required [S1].

## Subtopics

### [Interval Integration for Volumetric Light Scattering](3090-interval-integration-for-volumetric-light-scattering.md)
Interval Integration is a fragment shader technique for computing in-scattered radiance by decomposing the integration path into intervals defined by the front and back faces of a light volume. It solves the problem of efficiently evaluating the scattering integral for directional, omnidirectional, and spot lights by leveraging geometric intersections and analytical or numerical solvers tailored to the light type. Unlike its parent method, Semi-Analytical Fragment Shader Preintegration, which relies on adaptive preintegration over extrema of scalar polynomials, Interval Integration explicitly uses the light volume's entry and exit points to form integration bounds, allowing for closed-form solutions for directional lights and lookup-based or numerical integration for complex light sources.

## Operating characteristics
*   **Time Complexity:** Depends on the number of extrema found. A cubic polynomial has at most 2 extrema, and a quadratic has 1, so the number of integration intervals is small (typically 3-5) per cell [S1]. This makes it faster than high-order fixed-step marching for similar accuracy but more expensive per fragment than simple fixed-step preintegration due to the polynomial evaluation and sorting.
*   **Memory Complexity:** Requires storage for the 256 $\times$ 256 $\times$ 32 16-bit RGBA preintegration table [S1]. This is a fixed memory cost independent of the volume resolution.
*   **Performance:** The method is implemented on the GPU using OpenGL [S1]. It allows for interactive rendering with AMR data structures [S1]. The use of 16-bit floating point buffers improves blending accuracy [S1].
*   **Strengths:** High accuracy for complex transfer functions; adaptive resolution where it matters; perspective-correct sorting of cells [S1].
*   **Limitations:** Requires computation of polynomial coefficients and extrema in the fragment shader; limited to polynomial signal representations within cells; precision issues at very small intervals if logarithmic mapping is not used [S1].

## Background
The method was introduced in the context of Adaptive Mesh Refinement (AMR) volume rendering [S1]. It builds on the concept of partial pre-integration but enhances it by making the integration intervals adaptive to the local signal behavior [S1]. The implementation details, including the use of 16-bit textures and logarithmic access, were presented in a paper focusing on high-quality semi-analytical volume rendering for AMR data [S1].

## Sources
[S1] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
