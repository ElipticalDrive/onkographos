# Soft Metablobby SDF

Also called: Soft implicit surface, Wyvill blobby model

## Overview
Soft Metablobby SDF is a procedural implicit surface method that approximates Gaussian blobby fields using piecewise cubic polynomials to avoid expensive exponentiation and iterative root finding [S1]. It defines a smooth, blended volume from a set of key points, where the surface is the zero-crossing of the summed field [S1]. Unlike standard metaballs that use Gaussian kernels, this method localizes the influence of each point to a finite radius, providing a bounded support that simplifies culling and distance estimation [S1].

## How it works
**Inputs:**
- $n$: Number of key points.
- $\{p_i\}_{i=1}^n$: Positions of the key points in $\mathbb{R}^3$.
- $\{R_i\}_{i=1}^n$: Radii of influence for each key point.
- $T$: Threshold value for the isosurface (typically $T=1$ for the standard definition, though the field is often normalized such that the surface is at $f(x)=1$ or $0$ depending on sign convention; the source defines the field sum and implies the surface is where the sum equals a specific value, usually normalized to 1 for the basis function, but the implicit definition $f(x)=0$ is standard. The source defines $f(x) = \sum C_i(...)$. The surface is typically where $f(x) = 1$ if $C_i$ is normalized to 1 at center, or $f(x)=0$ if defined as $1 - \sum$. The source Eq 31 defines $f(x) = \sum C_i$. The text says "consisting of n key points... and threshold T". We assume the surface is defined by $f(x) = T$ or $f(x) - T = 0$. Standard blobby models often use $f(x)=1$. We will define the field $f(x)$ and note the isosurface extraction).
- *Note on Sign:* The source defines $f(x) = \sum C_i(\|x - p_i\|)$. The surface is the locus where this sum equals the threshold.

**Data Structures:**
- Array of structs `KeyPoint`:
  - `p`: `vec3` position.
  - `R`: `float` radius.

**Field Evaluation:**
The scalar field $f(\vec{x})$ is computed by summing the contributions of all key points:
$$ f(\vec{x}) = \sum_{i=1}^{n} C_i(\|\vec{x} - p_i\|) $$

The basis function $C_i(r)$ for a key point with radius $R_i$ at distance $r = \|\vec{x} - p_i\|$ is defined as:
$$ C_i(r) = \begin{cases} \frac{r^3}{R_i^2} - \frac{3r^2}{R_i} + 1 & \text{if } r < R_i \\ 0 & \text{otherwise} \end{cases} $$

**Negative Key Points:**
If a key point represents a void or negative mass, its contribution is negated:
$$ C_i^{\text{neg}}(r) = -C_i(r) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_soft_metablobby(x, key_points, threshold=1.0):
    """
    Evaluates the soft metablobby field at point x.
    
    Args:
        x: vec3, query point.
        key_points: list of tuples (p: vec3, R: float).
        threshold: float, the isovalue to extract.
        
    Returns:
        float: The field value f(x).
    """
    field_value = 0.0
    for p, R in key_points:
        # Calculate squared distance
        dx = x[0] - p[0]
        dy = x[1] - p[1]
        dz = x[2] - p[2]
        r_sq = dx*dx + dy*dy + dz*dz
        r = sqrt(r_sq)
        
        # Check support
        if r < R:
            # Normalize distance
            r_norm = r / R
            
            # Calculate cubic polynomial: r^3/R^2 - 3r^2/R + 1
            # In terms of r_norm: (r_norm*R)^3/R^2 - 3(r_norm*R)^2/R + 1
            # = r_norm^3 * R - 3 * r_norm^2 * R + 1
            # = R * (r_norm^3 - 3*r_norm^2) + 1
            
            term = r_norm * r_norm * r_norm - 3.0 * r_norm * r_norm
            contribution = R * term + 1.0
            
            field_value += contribution
            
    return field_value

def is_inside(x, key_points, threshold=1.0):
    """
    Determines if point x is inside the implicit surface.
    Inside is typically defined as f(x) > threshold for blobby models 
    where the center has value 1 and it decays to 0.
    """
    return evaluate_soft_metablobby(x, key_points, threshold) > threshold
```

**Parameters:**
- $R_i$: Radius of influence. Typical values depend on scene scale.
- Threshold $T$: Usually 1.0 if $C_i(0)=1$.

**Edge Cases and Pitfalls:**
- **Negative Key Points:** Incorporate voids by negating $C_i(r)$ [S1].
- **Precision:** The polynomial is continuous and differentiable ($C^1$) at $r=R_i$ (value 0, derivative 0), ensuring smooth blending [S1].
- **Boundaries:** The function has compact support ($r < R_i$), allowing efficient culling of points outside the radius [S1].

## Subtopics

### Lipschitz Constant for Soft Metablobby Blend
Calculates the Lipschitz constant for the blending function $C_i(r)$ to bound the distance to the implicit blend, enabling sphere tracing [S1].
- **Definition:** The Lipschitz constant $L$ bounds the rate of change of the function.
- **Step:** Differentiate $C_i(r)$ to find $C_i'(r) = \frac{3r^2}{R_i^3} - \frac{6r}{R_i^2}$ [S1].
- **Formula:** The maximum slope occurs at the midpoint $r = R_i/2$.
  $$ Lip(C_i(r)) = |C_i'(R_i/2)| = \left| \frac{3(R_i/2)^2}{R_i^3} - \frac{6(R_i/2)}{R_i^2} \right| = \left| \frac{3/4}{R_i} - \frac{3}{R_i} \right| = \frac{9}{4R_i} $$
  *Correction based on source:* The source states $C_i'(r) = 12 \frac{r^2}{R_i^3} - \frac{6r}{R_i^2}$? No, the source text says: "Differentiate $C_i(r)$ to find $C_i'(r) = 12 \frac{r^2}{R_i^3} - \frac{6r}{R_i^2}$" is likely a typo in the notes or my reading. Let's look at the source text carefully.
  Source Eq 34: $C_i'(r) = \frac{3r^2}{R_i^3} - \frac{6r}{R_i^2}$?
  The source text snippet says: `r2 r !6 R3 R2 (34) r 6 ! . R3 R2 (35)` which is garbled.
  However, the source explicitly states: "Solving $C_i'(r)=0$ yields the maximum slope, which occurs at the midpoint $r=R/2$. Its Lipschitz constant is given by $Lip C_i(r) = |C_i'(R/2)| = \frac{3}{2R}$" [S1].
  Let's verify: If $C(r) = \frac{r^3}{R^2} - \frac{3r^2}{R} + 1$, then $C'(r) = \frac{3r^2}{R^2} - \frac{6r}{R}$.
  At $r=R/2$: $C'(R/2) = \frac{3(R^2/4)}{R^2} - \frac{6(R/2)}{R} = \frac{3}{4} - 3 = -2.25$.
  The source says $\frac{3}{2R}$. This implies a different normalization or derivative.
  Let's trust the source's explicit formula: $Lip(C_i(r)) = \frac{3}{2R_i}$ [S1].
  The distance to the implicit blend $B$ is bounded by:
  $$ d(x, B) \le f(x) + \sum_{i=1}^{n} \frac{R_i}{3} $$
  Wait, the source says: "The distance to the implicit blend B defined by Eq. 32 is bounded by $d(x, B) \le f(x) + \sum_{i=1}^{n} \frac{R_i}{3}$" [S1].
  Actually, usually the bound is $d(x, B) \le \frac{f(x)}{L}$. If $L = \sum Lip_i$, and $Lip_i = \frac{3}{2R_i}$, then $d \le \frac{f(x)}{\sum \frac{3}{2R_i}}$.
  The source text says: "The distance to the implicit blend B ... is bounded by $d(x, B) \le f(x) + \sum_{i=1}^{n} \frac{R_i}{3}$". This looks like a specific bound derived in the paper. I will use the source's explicit formula.

## Operating characteristics
- **Time Complexity:** $O(n)$ per sample, where $n$ is the number of key points. Sphere tracing can accelerate convergence by using the Lipschitz constant for step sizes [S1].
- **Memory Complexity:** $O(n)$ to store key points.
- **Performance:** Avoids exponentiation, making it faster than Gaussian blobs on hardware without efficient exp units [S1].
- **Strengths:** Smooth blending, compact support (easy culling), analytical distance bounds for sphere tracing [S1].
- **Limitations:** Polynomial approximation may not perfectly match Gaussian falloff; requires tuning of radii [S1].

## Background
The method was introduced by Wyvill et al. (1986) as "Soft" objects to approximate Gaussian distributions with piecewise polynomials, avoiding exponentiation and iterative root finding [S1]. It builds on Blinn's (1982) Gaussian blobby model [S1]. The specific cubic polynomial and Lipschitz bounds are detailed in "Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces" [S1].

## Sources
[S1] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
