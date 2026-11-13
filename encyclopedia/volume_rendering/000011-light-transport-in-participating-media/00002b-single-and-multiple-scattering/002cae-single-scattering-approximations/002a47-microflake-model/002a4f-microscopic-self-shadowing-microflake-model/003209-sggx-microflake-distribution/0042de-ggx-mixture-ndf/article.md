# GGX Mixture NDF

Also called: SGGX Mixture, Surface NDF, Surface Normal Distribution Function (NDF) Mixture with SGGX

## Overview
The GGX Mixture NDF is a compact representation of the marginal distribution of surface normals, $p_N(n)$, used to model the glossiness and anisotropy of aggregated microflake appearances [S1, unofficial]. It solves the problem of representing complex, arbitrarily oriented surface normal distributions by combining multiple SGGX lobes [S1, unofficial]. It is distinguished from single-lobe SGGX by its ability to capture multi-modal or highly anisotropic normal distributions through a weighted sum, building on the [SGGX Microflake Distribution](6660-sggx-microflake-distribution.md) [S1, unofficial].

## How it works
**Inputs:**
- A set of $k$ SGGX lobes, each defined by an eigenbasis $R_i$ and anisotropic roughness parameters $\alpha_i = (\alpha_{x,i}, \alpha_{y,i})$ [S1, unofficial].
- A set of positive weights $w_i$ for each lobe [S1, unofficial].

**Outputs:**
- The composite Normal Distribution Function (NDF) $p_N(n)$ evaluated for any surface normal $n$ [S1, unofficial].

**Data Structures:**
- **Lobes:** An array of size $k$. Each element $i$ contains:
  - `R_i`: A $3 \times 3$ rotation matrix (eigenbasis) representing the orientation of the $i$-th SGGX lobe [S1, unofficial].
  - `alpha_i`: A 2D vector $(\alpha_{x,i}, \alpha_{y,i})$ representing the anisotropic roughness [S1, unofficial].
  - `w_i`: A scalar weight [S1, unofficial].
- **Weights:** The weights $w_i$ must satisfy $\sum_{i=1}^{k} w_i = 1$ and $w_i > 0$ [S1, unofficial].

**Steps:**
1. For a given surface normal $n$, evaluate the SGGX distribution $D_{i\text{sggx}}(n)$ for each lobe $i$ from $1$ to $k$ [S1, unofficial].
2. Multiply each lobe's distribution value by its corresponding weight $w_i$ [S1, unofficial].
3. Sum the weighted results to obtain the final NDF value:
   $$ p_N(n) = \sum_{i=1}^{k} w_i D_{i\text{sggx}}(n) $$
   [S1, unofficial]

**Parameters:**
- $k$: The number of SGGX lobes in the mixture. Typical value not specified in the sources [S1, unofficial].
- $w_i$: Positive weights summing to 1 [S1, unofficial].
- $R_i$: Eigenbasis of the $i$-th SGGX distribution [S1, unofficial].
- $\alpha_i$: Anisotropic roughness $(\alpha_{x,i}, \alpha_{y,i})$ of the $i$-th SGGX distribution [S1, unofficial].

**Edge Cases and Pitfalls:**
- The weights must be strictly positive and sum to 1 [S1, unofficial].
- The fitting process for the mixture parameters is described in §7.1 of the source, but specific algorithms or constraints for the fitting are not detailed in the provided text [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ggx_mixture_ndf(n, lobes):
    """
    Evaluate the GGX Mixture NDF for a given normal n.
    
    Args:
        n: A 3D vector representing the surface normal.
        lobes: A list of k lobes. Each lobe is a tuple (R, alpha, w), where:
               R is a 3x3 rotation matrix (eigenbasis).
               alpha is a 2D tuple (alpha_x, alpha_y).
               w is a scalar weight.
               
    Returns:
        The value of the NDF p_N(n).
    """
    total = 0.0
    for R, alpha, w in lobes:
        # Evaluate SGGX distribution D_sggx(n; R, alpha)
        # Implementation follows the SGGX Microflake Distribution method [G]
        d_sggx = sggx_distribution(n, R, alpha)
        total += w * d_sggx
    return total

def sggx_distribution(n, R, alpha):
    """
    Standard SGGX NDF evaluation [G].
    Computes D_sggx(n; R, alpha) = (alpha_x * alpha_y) / (pi * ((n_x*alpha_x)^2 + (n_y*alpha_y)^2 + n_z^2)^2)
    where (n_x, n_y, n_z) = R @ n.
    """
    nx, ny, nz = R @ n
    denom = (nx * alpha[0])**2 + (ny * alpha[1])**2 + nz**2
    return (alpha[0] * alpha[1]) / (3.141592653589793 * denom**2)
```

## Subtopics

### SGGX Convolution Approximation
An approximation of the convolution between an SGGX lobe $D_{sggx}(n; R, \alpha)$ and an isotropic spherical distribution $g(\omega; n)$ by another SGGX with increased roughness [S1, unofficial]. It is chosen when efficient evaluation of the convolved distribution is required, avoiding numerical integration [S1, unofficial]. The method parameterizes the SGGX by its eigenbasis $R$ and anisotropic roughness $\alpha = (\alpha_x, \alpha_y)$ [S1, unofficial]. It performs a nonlinear least-square fit to find a mapping $\alpha^+ = M(\alpha, g)$ such that $\int_{S^2} g(\omega; n) D_{sggx}(n; R, \alpha) dn \approx D_{sggx}(\omega; R, \alpha^+)$ [S1, unofficial]. The eigenbasis $R$ remains fixed, while the roughness $\alpha$ is updated to $\alpha^+$ [S1, unofficial]. The mapping $M$ is scene-independent and smooth, allowing it to be precomputed and stored in a table [S1, unofficial]. For a mixture, the convolved distribution is $D_{conv}(\omega) \approx \sum_{i=1}^{k} w_i D_{i\text{sggx}}(\omega; R_i, \alpha_i^+)$ [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Evaluation of the mixture NDF is $O(k)$, where $k$ is the number of lobes [S1, unofficial].
- The convolution approximation requires precomputation of the mapping $M(\alpha, g)$, which is scene-independent and has negligible storage overhead if stored as a precomputed table [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- The accuracy of the convolution approximation depends on the target distributions $g$ and the fitting process [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Compact representation of complex normal distributions; efficient evaluation; convolution approximation avoids numerical integration [S1, unofficial].
- **Limitations:** Requires fitting process for mixture parameters; convolution approximation is an approximation [S1, unofficial].

**When to Use:**
- Use when modeling aggregated microflake appearances with complex glossiness and anisotropy [S1, unofficial].

**When Not to Use:**
- Not given in the sources.

## Background
The GGX Mixture NDF was introduced by Heitz et al. in the paper "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" [S1, unofficial]. It builds on the SGGX distribution introduced by Heitz et al. [HDCD15] [S1, unofficial]. The convolution approximation is inspired by Xu et al. [2013] for anisotropic Spherical Gaussians [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
