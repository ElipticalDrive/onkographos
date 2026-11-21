# Final Specular ABSDF Expression

Also called: Specular Aggregation Result

## Overview
This method computes the final aggregated specular term for the Aggregated Bidirectional Scattering Distribution Function (ABSDF) by combining pre-convolved Normal Distribution Function (NDF) lobes with shadowing-masking and geometric terms. It serves as the final evaluation step in the ABSDF pipeline, transforming the factorized components into a single integrand value for ray marching. It differs from sibling methods by applying the shadowing-masking term $G$ as a post-convolution multiplier per lobe, leveraging the smoothness of $G$ to avoid expensive per-sample integration.

## How it works
**Inputs:**
- Incident direction $\omega_i$ and outgoing direction $\omega_o$.
- Surface normal basis $R$ (eigenbasis).
- Anisotropic roughness parameters $\alpha_{ij}$ for each lobe.
- Pre-computed convolved NDF values $D_{conv}(\omega_h)_{ij}$ for each lobe index $i \in \{1, 2\}$ and sample index $j \in \{1, \dots, k\}$.
- Weights $w_i$ and mixture coefficients $m_{ij}$.

**Outputs:**
- The approximated integral value $\int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma$.

**Data Structures:**
- The expression sums over $i=1$ to $2$ and $j=1$ to $k$.
- $D_{conv}(\omega_h)_{ij}$ is a scalar value representing the convolved NDF for the $i$-th lobe at the $j$-th sample.
- $S_{ij}$ and $G_{ij}$ are scalar values computed per lobe/sample pair.

**Steps:**
1. Initialize the result accumulator $Result = 0$.
2. Iterate over lobe index $i$ from 1 to 2.
3. Iterate over sample index $j$ from 1 to $k$.
4. Compute the shadowing-masking term $S_{ij}$ using the geometric parameters $\omega_i, \omega_o$ and roughness $\alpha_{ij}$:
   $$ S_{ij} = S(\omega_i, \omega_o; \alpha_{ij}) $$
   [G] $S$ typically refers to a geometric shadowing function, often approximated by Schlick-GGX or similar, normalized for the spherical domain.
5. Compute the shadowing-masking term $G_{ij}$ using the normal basis $R$ and roughness $\alpha_{ij}$:
   $$ G_{ij} = G(\omega_i, \omega_o, R, \alpha_{ij}) $$
   [G] $G$ is the standard microfacet shadowing-masking term.
6. Retrieve the pre-computed convolved NDF value $D_{conv}(\omega_h)_{ij}$.
7. Calculate the contribution of this term:
   $$ Term = w_i \cdot m_{ij} \cdot D_{conv}(\omega_h)_{ij} \cdot S_{ij} \cdot G_{ij} $$
8. Add $Term$ to $Result$.
9. Return $Result$.

**Formula:**
The final expression is:
$$ \int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma \approx \sum_{i=1}^{2} \sum_{j=1}^{k} w_i m_{ij} D_{conv}(\omega_h)_{ij} S_{ij}(\omega_i, \omega_o) G_{ij}(\omega_i, \omega_o) $$
where:
- $w_i$ are positive weights summing to 1.
- $m_{ij}$ are mixture coefficients.
- $D_{conv}(\omega_h)_{ij}$ is the convolved NDF for lobe $i$, sample $j$.
- $S_{ij}(\omega_i, \omega_o) = S(\omega_i, \omega_o; \alpha_{ij})$ is the shadowing term.
- $G_{ij}(\omega_i, \omega_o) = G(\omega_i, \omega_o, R, \alpha_{ij})$ is the masking term.

**Parameters:**
- $k$: Number of samples per lobe. Not given in the sources.
- $w_i$: Weights for the mixture model. Not given in the sources.
- $m_{ij}$: Mixture coefficients. Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision: The smoothness of $G$ allows it to be multiplied post-convolution, but care must be taken if $G$ varies rapidly in specific degenerate configurations (e.g., grazing angles).
- Boundaries: Ensure $w_i$ sum to 1 to maintain energy conservation relative to the base NDF.

## Operating characteristics
- Time Complexity: $O(k)$ per surface point, where $k$ is the number of samples per lobe. The convolution is pre-computed, so the runtime cost is dominated by the summation and evaluation of $S$ and $G$.
- Memory Complexity: $O(k)$ per lobe to store pre-computed $D_{conv}$ values.
- Strengths: Efficient evaluation due to pre-computation and the separation of $G$.
- Limitations: Accuracy depends on the quality of the pre-computed convolution and the approximation of $G$ as a post-multiplication factor.

## Background
This expression is derived from the ABSDF framework for efficient scene appearance aggregation. It builds on the factorization of the NDF into a mixture of SGGX lobes and their convolution with a spherical distribution. The specific handling of the shadowing-masking term $G$ as a post-convolution multiplier is highlighted in the source.

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
