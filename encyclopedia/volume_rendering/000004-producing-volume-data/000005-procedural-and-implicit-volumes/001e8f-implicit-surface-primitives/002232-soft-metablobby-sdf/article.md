# Soft Metablobby SDF
Also called: Soft implicit surface, Wyvill blobby model

## Overview
Soft Metablobby SDF defines an implicit surface by blending contributions from a set of key points using a piecewise cubic polynomial that approximates a Gaussian distribution [S1]. It differs from standard Metaball Field Evaluation by using a localized cubic support rather than an exponential decay, which avoids expensive exponentiation and iterative root finding while still producing smooth, organic blends [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $n$ key points $\mathbf{p}_i$, associated radii $R_i$, and a global threshold $T$.
*   **Output:** A scalar field value $f(\mathbf{x})$ at query point $\mathbf{x}$, and optionally a signed distance bound.

**Data Structures:**
*   Key points: Array of 3D vectors $\mathbf{p}_i$.
*   Radii: Array of floats $R_i$.
*   Threshold: Scalar float $T$.

**Method:**
1.  Initialize total field value $f(\mathbf{x}) = 0$.
2.  For each key point $i$ from $1$ to $n$:
    a. Calculate distance $r = \|\mathbf{x} - \mathbf{p}_i\|$.
    b. Evaluate the basis function $C_i(r)$:
       $$
       C_i(r) = \begin{cases} 
       \frac{r^3}{R_i^2} - \frac{3r^2}{R_i} + 1 & \text{if } r < R_i \\
       0 & \text{otherwise}
       \end{cases}
       $$
    c. Add contribution to field: $f(\mathbf{x}) \leftarrow f(\mathbf{x}) + C_i(r)$ [S1].
3.  The surface is defined where $f(\mathbf{x}) = T$.

**Negative Key Points:**
If a key point represents a void or subtraction, negate the value returned by $C_i(r)$ before adding it to $f(\mathbf{x})$ [S1].

**Parameters:**
*   $R_i$: Radius of influence for point $i$. Typical values depend on scene scale.
*   $T$: Threshold for surface extraction.

**Edge Cases and Pitfalls:**
*   **Precision:** The cubic polynomial is continuous but has discontinuous derivatives at $r=R_i$ if not carefully handled, though the function itself is $C^1$ continuous at the boundary $R_i$ (value and first derivative are zero).
*   **Boundaries:** Points far from $\mathbf{x}$ contribute 0, allowing for efficient culling.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_metablobby_sdf(x, points, radii, threshold):
    """
    Evaluates the soft metablobby implicit field.
    
    Args:
        x: Query point (vec3)
        points: List of key points (list of vec3)
        radii: List of radii corresponding to points (list of float)
        threshold: Surface threshold value (float)
        
    Returns:
        field_value: Scalar field value at x (float)
    """
    field_value = 0.0
    
    for i in range(len(points)):
        p_i = points[i]
        R_i = radii[i]
        
        # Calculate distance
        diff = x - p_i
        r = length(diff)
        
        # Evaluate basis function C_i(r)
        if r < R_i:
            # C_i(r) = (r^3 / R_i^2) - (3*r^2 / R_i) + 1
            r_sq = r * r
            r_cu = r_sq * r
            c_i = (r_cu / (R_i * R_i)) - (3.0 * r_sq / R_i) + 1.0
        else:
            c_i = 0.0
            
        # Add contribution (assuming positive key point)
        # Note: For negative key points, negate c_i before adding [S1]
        field_value += c_i
        
    return field_value
```

## Subtopics

### Lipschitz Constant for Soft Metablobby Blend
Calculates the Lipschitz constant for the blending function $C_i(r)$ to bound the distance to the implicit blend [S1]. Differentiate $C_i(r)$ to find $C_i'(r) = 12 \frac{r^2}{R_i^3} - \frac{6r}{R_i^2}$ [S1]. The maximum slope occurs at the midpoint $r=R_i/2$, yielding the Lipschitz constant:
$$
Lip(C_i(r)) = |C_i'(R_i/2)| = \frac{3}{2R_i}
$$
The distance to the implicit blend $B$ is bounded by:
$$
d(x, B) \le f(x) + \sum_{i=1}^{n} \frac{R_i}{3}
$$
This bound allows for sphere tracing optimizations by providing a safe step size [S1].

## Operating characteristics
*   **Time Complexity:** $O(n)$ per query point, where $n$ is the number of key points.
*   **Memory Complexity:** $O(n)$ to store key points and radii.
*   **Performance:** Avoids exponentiation, making it faster than Gaussian-based metaballs on hardware without dedicated exp instructions [S1].
*   **Strengths:** Smooth blending, localized support (sparse evaluation), analytic derivative available for sphere tracing [S1].
*   **Limitations:** The distance bound is loose compared to true SDFs; sphere tracing may require more steps than with exact SDFs [S1].

## Background
The method was introduced by Wyvill et al. (1986) as "soft" objects to approximate Gaussian distributions with piecewise cubics, avoiding exponentiation and iterative root finding [S1]. It builds on Blinn's (1982) blobby model [S1]. The Lipschitz constant analysis for sphere tracing was detailed in later works, including Rockwood and Owen (1987) and Wyvill and Trotman (1990) [S1].

## Sources
[S1] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
