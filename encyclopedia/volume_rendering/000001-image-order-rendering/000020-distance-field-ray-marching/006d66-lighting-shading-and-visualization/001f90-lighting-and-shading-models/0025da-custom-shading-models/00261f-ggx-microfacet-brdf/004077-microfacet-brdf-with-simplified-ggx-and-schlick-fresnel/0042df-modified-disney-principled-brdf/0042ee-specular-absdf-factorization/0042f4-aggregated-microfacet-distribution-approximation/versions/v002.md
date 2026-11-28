# Aggregated Microfacet Distribution Approximation

Also called: Beta Distribution Fitting, Roughness Aggregation

## Overview
This method approximates the aggregated microfacet distribution $\hat{D}_{mic}(\omega_h)$, which represents the expectation of microfacet distributions over a statistical distribution of roughness values $\alpha$ [S1, unofficial]. It solves the problem of rendering assets with heterogeneous roughness by replacing the intractable integral of infinite lobes with a weighted sum of two GGX lobes [S1, unofficial]. It is distinguished from its parent [Specular ABSDF Factorization](10872-specular-absdf-factorization.md) by explicitly modeling the roughness variance using a Beta distribution to handle the bounded nature of $\alpha \in [0, 1]$, avoiding the invalid range errors associated with Gaussian approximations [S1, unofficial].

## How it works
**Inputs:**
- Shape parameters $a, b$ of the Beta distribution $B(\alpha; a, b)$ modeling the roughness marginal density $p_A(\alpha)$ [S1, unofficial].
- The microfacet distribution function $D_{mic}(\omega_h; \alpha)$ (typically GGX) [S1, unofficial].

**Outputs:**
- Approximated aggregated distribution $\hat{D}_{mic}(\omega_h)$ defined by weights $m_1, m_2$ and roughness values $\alpha_1, \alpha_2$ [S1, unofficial].

**Data Structures:**
- A 2D lookup table storing the mapping $(m_1, \alpha_1, \alpha_2) = M_2(a, b)$ for discrete values of $a$ and $b$ [S1, unofficial].

**Steps:**
1. Model the roughness distribution $p_A(\alpha)$ using a Beta distribution $B(\alpha; a, b)$ [S1, unofficial].
2. Define the aggregated distribution as the expectation:
   $$ \hat{D}_{mic}(\omega_h) = \int_0^1 D_{mic}(\omega_h; \alpha) p_A(\alpha) d\alpha $$
   Since no closed-form solution exists, approximate this integral using a 2-lobe weighted sum:
   $$ \hat{D}_{mic}(\omega_h) \approx m_1 D_{mic}(\omega_h; \alpha_1) + m_2 D_{mic}(\omega_h; \alpha_2) $$
   where $m_1 + m_2 = 1$ [S1, unofficial].
3. Perform a nonlinear least-square fit to determine the mapping $(m_1, \alpha_1, \alpha_2) = M_2(a, b)$ that minimizes the error between the integral and the 2-lobe approximation for given $a, b$ [S1, unofficial].
4. Store the resulting mapping in a 2D table indexed by $a$ and $b$ [S1, unofficial].

**Parameters:**
- $a, b$: Shape parameters of the Beta distribution. Typical values depend on the asset data statistics; the sources do not provide specific default ranges but note they are estimated from asset data [S1, unofficial].

**Edge Cases and Pitfalls:**
- The Beta distribution is chosen specifically because $\alpha$ is bounded in $[0, 1]$. Using a Gaussian distribution is invalid as its tails exceed this range, leading to errors [S1, unofficial].
- While the approximation can be extended to $k$ lobes, $k=2$ is selected as the balance between cost and accuracy [S1, unofficial].

## Subtopics

### [Aggregated Microfacet Distribution via Beta Distribution](10888-aggregated-microfacet-distribution-via-beta-distribution.md)
This variant explicitly uses the Beta distribution $B(\alpha; a, b)$ to model the statistical spread of roughness values $\alpha$. It is chosen over Gaussian approximations because the Beta distribution is naturally bounded within the valid $[0, 1]$ range of $\alpha$, preventing invalid range errors that occur when Gaussian tails exceed these bounds.

### 2-Lobe Microfacet Distribution Fit
This method fits the aggregated microfacet distribution using a 2-lobe weighted sum [S1, unofficial]. It is compared against ground-truth aggregated distributions and single-lobe fits with simple averaged roughness. The 2-lobe fit is overall more accurate and guaranteed to be never worse than simple fits [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Pre-computation requires a nonlinear least-square fit to populate the 2D lookup table for $(m_1, \alpha_1, \alpha_2)$ given $a, b$ [S1, unofficial].
- Runtime evaluation is $O(1)$ per pixel/vertex, involving two evaluations of $D_{mic}$ and a weighted sum, plus a table lookup for the parameters [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Handles heterogeneous roughness distributions accurately; avoids invalid roughness values; more accurate than single-lobe simple averaging [S1, unofficial].
- **Limitations:** Requires pre-computation and storage of the 2D table; the fitting process is non-trivial (nonlinear least-square) [S1, unofficial].

**When to Use:**
- Use when rendering assets with a distribution of roughness values (e.g., Level-of-Detail rendering where roughness varies) and higher accuracy than simple averaging is required [S1, unofficial].

## Background
The method originates from work on efficient scene appearance aggregation for Level-of-Detail rendering [S1, unofficial]. It addresses the limitation of standard microfacet models which assume a single roughness value, by aggregating the distribution over a statistical spread of roughness [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
