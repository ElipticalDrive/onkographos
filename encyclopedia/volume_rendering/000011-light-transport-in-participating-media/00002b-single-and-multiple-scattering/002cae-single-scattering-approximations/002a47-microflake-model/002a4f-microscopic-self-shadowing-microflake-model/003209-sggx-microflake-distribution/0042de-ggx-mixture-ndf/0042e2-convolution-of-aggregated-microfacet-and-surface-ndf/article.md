# Convolution of Aggregated Microfacet and Surface NDF

Also called: Microfacet-NDF Convolution

## Overview
This method computes the effective normal distribution function (NDF) for a microflake model by convolving an aggregated microfacet distribution $\hat{D}_{mic}$ with a surface NDF $p_N(n)$ represented as a mixture of SGGX distributions. It solves the problem of accurately modeling specular reflection from rough surfaces where the microgeometry is complex and arbitrarily oriented, avoiding the computational cost of full integration. It differs from standard microfacet models by explicitly accounting for the statistical orientation of the surface normals via convolution, rather than assuming a fixed local frame.

## How it works
**Inputs:**
- Aggregated microfacet distribution $\hat{D}_{mic}$, modeled as a mixture of $k$ lobes with weights $m_j$ and roughness parameters $\alpha_j$.
- Surface NDF $p_N(n)$, modeled as a mixture of 2 SGGX lobes with weights $w_i$ and roughness parameters $\alpha_i$.
- The mapping table $M_3$ for convolving roughness with SGGX.

**Outputs:**
- Convolved NDF $D_{conv}(\omega_h)$, represented as a mixture of SGGX distributions.

**Data Structures:**
- $M_3$: A 3D lookup table storing the mapping $\alpha^+ = M_3(\alpha, \alpha_{ref})$ for convolving a roughness $\alpha$ with a reference roughness $\alpha_{ref}$. The table is precomputed and stored.

**Steps:**
1. **Decompose Mixtures:** Represent the surface NDF as a sum of 2 SGGX lobes: $p_N(n) = \sum_{i=1}^{2} w_i D_{sggx}(n; R_i, \alpha_i)$. Represent the aggregated microfacet distribution as a sum of $k$ lobes with weights $m_j$ and roughness $\alpha_j$.
2. **Pairwise Convolution:** Compute the convolution for each pair of lobes $(i, j)$ from the surface NDF and microfacet mixtures. The convolution of an SGGX distribution with another isotropic spherical distribution is approximated by a roughened SGGX with the same eigenbasis $R$ but updated roughness $\alpha^+$.
3. **Roughness Mapping:** For each pair, compute the effective roughness $\alpha^+_{ij}$ using the mapping table $M_3$:
   $$ \alpha^+_{ij} = M_3(\alpha_i, M_3(\alpha_j)) $$
   Here, $M_3(\alpha_j)$ is a shorthand for fitting a beta distribution for $\alpha_j$ and querying $M_3$ for the roughness of the $j$-th microfacet lobe. The outer $M_3$ combines the surface roughness $\alpha_i$ with the microfacet roughness.
4. **Reconstruct NDF:** Sum the contributions of all pairs, weighted by the product of their respective weights $w_i$ and $m_j$:
   $$ D_{conv}(\omega_h) = \sum_{i=1}^{2} \sum_{j=1}^{k} w_i m_j D_{sggx}(\omega_h; R_i, \alpha^+_{ij}) $$
   where $D_{conv}(\omega_h)_{ij} \approx D_{sggx}(\omega; R_i, \alpha^+_{ij})$.
5. **Shadowing-Masking:** Apply the microfacet shadowing-masking term $G$ to each lobe post-convolution, as it is smooth and can be multiplied directly.

**Pitfalls:**
- The approximation assumes the convolution result is always an SGGX. This may not hold for extreme roughness values or highly anisotropic cases not covered by the $M_3$ table.
- The mapping table $M_3$ must be sufficiently dense to avoid interpolation artifacts.

## Subtopics

### Correction for Conditioned Angular Domain
An approximation technique for the conditioned angular domain in SGGX convolution, which assumes constant contribution in the shape term. It uses a 1D table $T_{LTC}^{-1}$ for the inverse LTC transform (varying only with roughness) and a 1D table $M_5(\alpha)$ for the normalization term (integral of GGX over full sphere). The constant contribution assumption can cause under- or overestimates, resulting in slightly darker or brighter results, and stretched lobe shapes compared to references [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel if the mapping table $M_3$ is used, as it involves simple table lookups and summation over a small number of lobes (2 surface lobes $\times$ $k$ microfacet lobes).
- **Memory Complexity:** $O(1)$ for the mapping tables $M_3$, $T_{LTC}^{-1}$, and $M_5$.
- **Strengths:** Efficient approximation of complex microfacet-surface interactions; avoids expensive numerical integration.
- **Limitations:** Approximation errors in lobe shape and intensity due to constant contribution assumption in conditioned angular domain.
- **When to use:** Real-time rendering or LOD rendering where performance is critical and slight inaccuracies are acceptable.
- **When not to use:** High-fidelity offline rendering requiring exact integration over conditioned angular domains.

## Background
Originated in the context of efficient scene appearance aggregation for level-of-detail rendering, building on SGGX distributions [Heitz et al. 2015] and normal map filtering techniques [Olano and Baker 2010; Kaplanyan et al. 2016]. The method was proposed to handle arbitrarily oriented surfaces in microflake models by convolving the microfacet distribution with the surface NDF.

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
