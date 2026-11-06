# Soft Metablobby SDF

Also called: Soft implicit surface, Wyvill blobby model

## Overview
Soft Metablobby SDF defines a scalar field using piecewise cubic polynomials to approximate Gaussian blobby blending, avoiding expensive exponentiation and iterative root finding [S1]. It differs from standard Metaball Field Evaluation by using a localized, compact-support cubic kernel rather than a global Gaussian, providing smoother blends with lower computational cost per sample [S1].

## How it works
**Inputs:**
*   $n$: Number of key points (blobs).
*   $p_i \in \mathbb{R}^3$: Position of the $i$-th key point.
*   $R_i > 0$: Radius of the $i$-th key point.
*   $T$: Threshold value for the implicit surface (typically $T=1$).
*   $\mathbf{x} \in \mathbb{R}^3$: Query point.

**Outputs:**
*   $f(\mathbf{x})$: The scalar field value at $\mathbf{x}$. The surface is defined where $f(\mathbf{x}) = T$.

**Data Structures:**
*   Array of structures, each containing `position` (vec3) and `radius` (float).

**Steps:**
1.  Initialize accumulator $f(\mathbf{x}) = 0$.
2.  For each key point $i$ from $1$ to $n$:
    a. Calculate distance $r = \|\mathbf{x} - p_i\|$.
    b. Evaluate the kernel function $C_i(r)$.
    c. Add contribution to accumulator: $f(\mathbf{x}) \leftarrow f(\mathbf{x}) + C_i(r)$.
3.  Return $f(\mathbf{x})$.

**Formulas:**
The kernel function $C_i(r)$ is defined as [S1]:
$$
C_i(r) = \begin{cases} 
\frac{r^3}{R_i^2} - \frac{3r^2}{R_i} + 1 & \text{if } r < R_i \\
0 & \text{otherwise}
\end{cases}
$$

The total field is the sum of contributions [S1]:
$$
f(\mathbf{x}) = \sum_{i=1}^{n} C_i(\|\mathbf{x} - p_i\|)
$$

**Parameters:**
*   $T$: Threshold. Not given in the sources.
*   $R_i$: Radius. Must be positive.

**Pitfalls:**
*   **Negative Key Points:** To incorporate negative key points (voids or subtractive volumes), negate the value returned by $C_i(r)$ before adding to the accumulator [S1].
*   **Precision:** The function is continuous and differentiable ($C^1$), but the second derivative is discontinuous at $r=R_i$. This is sufficient for sphere tracing but may affect gradient-based optimization if used for animation [S1].
*   **Boundary:** The function is exactly zero outside $R_i$, ensuring local support and efficient culling [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_metablobby_sdf(x, blobs, threshold):
    """
    Evaluate the Soft Metablobby SDF at point x.
    
    Args:
        x: Query point (vec3).
        blobs: List of tuples (p_i, R_i).
        threshold: Surface threshold T.
        
    Returns:
        Scalar field value f(x).
    """
    f_val = 0.0
    for p_i, R_i in blobs:
        # Calculate distance
        dx = x[0] - p_i[0]
        dy = x[1] - p_i[1]
        dz = x[2] - p_i[2]
        r = sqrt(dx*dx + dy*dy + dz*dz)
        
        # Evaluate kernel
        if r < R_i:
            r_sq = r * r
            r_cu = r_sq * r
            # C_i(r) = r^3/R_i^2 - 3r^2/R_i + 1
            term1 = r_cu / (R_i * R_i)
            term2 = 3.0 * r_sq / R_i
            c_i = term1 - term2 + 1.0
            f_val += c_i
        # else: contribution is 0
        
    return f_val
```

## Subtopics

### Lipschitz Constant for Soft Metablobby Blend
Calculates the Lipschitz constant for the blending function $C_i(r)$ to bound the distance to the implicit blend [S1]. The derivative is $C_i'(r) = 12 \frac{r^2}{R_i^3} - \frac{6r}{R_i^2}$ [S1]. The maximum slope occurs at the midpoint $r = R_i/2$, yielding the Lipschitz constant $Lip(C_i(r)) = |C_i'(R_i/2)| = \frac{3}{2R_i}$ [S1]. The distance to the implicit blend $B$ is bounded by $d(x, B) \le f(x) + \sum_{i=1}^{n} \frac{R_i}{3}$ [S1].

## Operating characteristics
*   **Time Complexity:** $O(n)$ per query point, where $n$ is the number of blobs. Each evaluation involves a distance calculation and a few arithmetic operations.
*   **Memory Complexity:** $O(n)$ to store the blob parameters.
*   **Performance:** Avoids exponentiation and iterative root finding, making it faster than Gaussian blobbies [S1].
*   **Strengths:** Smooth blending, local support (efficient culling), analytic derivatives available for sphere tracing [S1].
*   **Limitations:** The resulting field is not a true Signed Distance Function (SDF) in the strict geometric sense (the gradient magnitude is not 1), though it can be used with sphere tracing with potential inaccuracies in step size [S1].

## Background
The "blobby" model was introduced by Blinn (1982) using Gaussian distributions [S1]. Wyvill et al. (1986) proposed "soft" objects using sixth-degree polynomials, but later refined this to piecewise cubics to avoid exponentiation and localize blends [S1]. This specific cubic formulation is detailed in *Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces* [S1].

## Sources
[S1] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
