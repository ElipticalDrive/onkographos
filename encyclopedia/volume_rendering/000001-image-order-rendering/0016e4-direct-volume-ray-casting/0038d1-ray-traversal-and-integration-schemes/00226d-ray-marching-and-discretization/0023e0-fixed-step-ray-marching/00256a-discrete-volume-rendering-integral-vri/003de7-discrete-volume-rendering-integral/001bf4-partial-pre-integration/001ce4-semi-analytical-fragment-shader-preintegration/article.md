# Semi-Analytical Fragment Shader Preintegration

Also called: Semi-analytical preintegration, Adaptive preintegration intervals

## Overview
Semi-Analytical Fragment Shader Preintegration is a GPU-based rendering technique that improves the accuracy of discrete volume rendering integrals by adaptively subdividing the ray integration interval based on the local extrema of the underlying scalar and shading polynomials [S1]. Unlike fixed-step ray marching or static preintegration tables, this method computes the roots of the derivative of the signal and shading functions within the fragment's bounding box to define integration sub-intervals, ensuring that the piecewise-linear approximation of the transfer function error is minimized [S1]. It sets itself apart from its parent [Partial Pre-Integration](1148-partial-pre-integration.md) by dynamically determining the integration bounds per fragment rather than relying on a uniform or coarse subdivision, and from its sibling [Hybrid Volume and Isosurface Rendering with Partial Pre-Integration](1311-hybrid-volume-and-isosurface-rendering-with-partial-pre-inte.md) by focusing exclusively on the fragment-shader-side optimization of the volume integral without explicit isosurface hybridization.

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin and direction (in object space), scalar field coefficients (cubic polynomial), shading coefficients (quadratic polynomial), and preintegration lookup tables.
*   **Output:** Accumulated color and opacity for the fragment.

**Data Structures:**
*   **Preintegration Table:** A 256 × 256 × 32 RGBA texture where each component is 16-bit floating point [S1]. The dimensions correspond to the scalar values at the start and end of the interval ($s_1, s_2$) and the interval length ($l$). Logarithmic table access is used for the length dimension to provide greater precision at small lengths [S1].
*   **Offscreen Buffer:** A 16-bit floating point buffer used for intermediate blending to improve accuracy before copying to the front buffer [S1].

**Polynomial Representation:**
The scalar signal $s(t)$ and shading factor $l(t)$ along the ray parameter $t \in [0, 1]$ are approximated by polynomials:
$$ s(t) = at^3 + bt^2 + ct + d $$
$$ l(t) = et^2 + ft + g $$
where $a,b,c,d$ are cubic coefficients and $e,f,g$ are quadratic coefficients derived from the cell data [S1].

**Algorithm Steps:**
1.  **Find Exit Point:** Compute the ray exit point of the current cell/patch [S1].
2.  **Compute Coefficients:** Calculate $a,b,c,d$ for the scalar signal and $e,f,g$ for the shading factor [S1].
3.  **Initialize Extrema List:** Set $T = \emptyset$.
4.  **Find Scalar Extrema:**
    *   If $a \neq 0$ and $b \neq 0$: Compute discriminant $\Delta = 4b^2 - 12ac$. Add roots of $s'(t)=0$ to $T$:
        $$ t = \frac{-2b \pm \sqrt{\Delta}}{6a} $$
    *   Else if $a = 0$ and $b \neq 0$: Add root of $s'(t)=0$ to $T$:
        $$ t = \frac{-c}{2b} $$
5.  **Find Shading Extrema:**
    *   If $e \neq 0$: Add root of $l'(t)=0$ to $T$:
        $$ t = \frac{-f}{2e} $$
6.  **Prune and Sort:**
    *   Remove any $t \in T$ such that $t \le 0$ or $t \ge 1$.
    *   Add boundary points $0$ and $1$ to $T$.
    *   Sort $T$ in ascending order.
7.  **Adaptive Preintegration:**
    *   For each consecutive pair $(t_1, t_2)$ in $T$:
        *   Reconstruct scalar values $s_1 = s(t_1)$ and $s_2 = s(t_2)$.
        *   Reconstruct shading values $l_1 = l(t_1)$ and $l_2 = l(t_2)$.
        *   Compute interval length $l = t_2 - t_1$.
        *   Fetch preintegration values from the front and back weighted tables using $(s_1, s_2, l)$ [S1].
        *   Combine the fetched values (accumulating color and opacity).
    *   The final fragment color is the result of this accumulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def semi_analytical_preintegration(a, b, c, d, e, f, g):
    # T stores the sorted integration boundaries
    T = []

    # Find extrema of scalar polynomial s(t) = at^3 + bt^2 + ct + d
    # s'(t) = 3at^2 + 2bt + c = 0
    if a != 0 and b != 0:
        delta = 4 * b**2 - 12 * a * c
        if delta >= 0:
            sqrt_delta = math.sqrt(delta)
            t1 = (-2 * b + sqrt_delta) / (6 * a)
            t2 = (-2 * b - sqrt_delta) / (6 * a)
            T.append(t1)
            T.append(t2)
    elif a == 0 and b != 0:
        # s'(t) = 2bt + c = 0
        t = -c / (2 * b)
        T.append(t)

    # Find extrema of shading polynomial l(t) = et^2 + ft + g
    # l'(t) = 2et + f = 0
    if e != 0:
        t = -f / (2 * e)
        T.append(t)

    # Prune points outside [0, 1]
    T = [t for t in T if 0 < t < 1]

    # Add boundaries and sort
    T.append(0.0)
    T.append(1.0)
    T.sort()

    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0

    # Iterate over intervals
    for i in range(len(T) - 1):
        t1 = T[i]
        t2 = T[i+1]

        # Reconstruct scalar values
        s1 = d + t1 * (c + t1 * (b + t1 * a))
        s2 = d + t2 * (c + t2 * (b + t2 * a))

        # Reconstruct shading values
        l1 = g + t1 * (f + t1 * e)
        l2 = g + t2 * (f + t2 * e)

        # Compute interval length
        length = t2 - t1

        # Fetch preintegration tables
        # Note: Logarithmic access for length is handled by the texture lookup function
        front_val = fetch_preintegration_table(s1, s2, length, type='front')
        back_val = fetch_preintegration_table(s1, s2, length, type='back')

        # Combine and accumulate
        # The exact combination logic follows the partial pre-integration model
        # where front/back weights are applied to the transfer function approximation
        accumulated_color, accumulated_opacity = accumulate(
            accumulated_color, accumulated_opacity,
            front_val, back_val, l1, l2
        )

    return accumulated_color, accumulated_opacity
```

**Edge Cases and Pitfalls:**
*   **Precision:** Small interval lengths require high precision. The use of 16-bit floating point textures and logarithmic indexing for the length dimension mitigates precision loss at small $l$ [S1].
*   **Degenerate Polynomials:** If $a=0$ and $b=0$, the scalar function is linear, and no extrema exist within the interval (other than boundaries). The code handles $a=0, b \neq 0$ explicitly [S1].
*   **Complex Roots:** If the discriminant $\Delta < 0$, there are no real extrema, and no points are added to $T$ from that polynomial [S1].

## Subtopics

### [Interval Integration for Volumetric Light Scattering](3090-interval-integration-for-volumetric-light-scattering.md)
Interval Integration computes in-scattered radiance by decomposing the ray path into intervals defined by the front and back faces of a light volume, allowing analytical evaluation of the scattering integral for directional lights and numerical or lookup-based evaluation for point and spot lights. It sets itself apart from sibling methods by explicitly treating light volume boundaries as integration limits, enabling a "sum and difference" approach where contributions are added for front-face intersections and subtracted for back-face intersections.

## Operating characteristics
*   **Time Complexity:** The fragment shader performs a constant number of arithmetic operations to find roots and a number of texture fetches proportional to the number of extrema found (typically small, 2-4 intervals). This is more expensive per fragment than fixed-step marching but yields higher accuracy with fewer steps.
*   **Memory Complexity:** Requires a 256 × 256 × 32 16-bit RGBA texture for the preintegration table [S1].
*   **Performance:** The method is implemented on the GPU using OpenGL [S1]. It uses a 16-bit floating point offscreen buffer for blending accuracy [S1]. Performance measurements are presented in the source but specific frame rates are not detailed in the provided notes [S1].
*   **Strengths:** Perspective-correct sorting of cells is ensured [S1]. High-quality rendering with adaptive accuracy.
*   **Limitations:** Requires computing polynomial coefficients and roots in the fragment shader, which can be computationally intensive if many extrema are present.

## Background
The method was implemented as part of a hybrid CPU-GPU Adaptive Mesh Refinement (AMR) volume rendering system [S1]. The CPU traverses a KD-tree of AMR patches back-to-front, sending indices to the GPU [S1]. The geometry shader instantiates faces, and the fragment shader performs the semi-analytical preintegration [S1]. The approach builds on the concept of Partial Pre-Integration [1148-partial-pre-integration.md] but adapts it for AMR data structures and fragment-shader execution [S1].

## Sources
[S1] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
