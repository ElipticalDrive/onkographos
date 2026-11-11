# Specular ABSDF Factorization

Also called: Specular Aggregation

## Overview
Specular ABSDF Factorization is the specular component of the Aggregated Bidirectional Surface Distribution Function (ABSDF) [10860-modified-disney-principled-brdf.md], designed to efficiently render complex, orientation-varying microfacet materials in large-scale scenes. It solves the problem of integrating the specular BRDF over a distribution of surface orientations and material parameters by factorizing the integral into separable terms: a precomputable convolution of the Normal Distribution Function (NDF) and a runtime-evaluated moment term [S1, unofficial]. Unlike [Phys3DGS BRDF Function](11071-phys3dgs-brdf-function.md), which may rely on different aggregation strategies, this method explicitly handles the coupling between Fresnel terms, roughness, and microfacet geometry through a split of the integral into "clear coat" and "base" contributions, allowing for real-time performance via precomputed tables [S1, unofficial].

## How it works
The method computes the aggregated specular response by substituting the specular BRDF $f_s$ into the ABSDF integral and expanding it into terms involving the Fresnel mix $R$ and the microfacet distribution $D$ [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** Incident direction $\omega_i$, outgoing direction $\omega_o$, material parameters $\gamma = (\beta_c, \beta_m, \beta_s, n, \alpha)$, where $\beta_c, \beta_m, \beta_s$ are Fresnel/roughness mix parameters, $n$ is the normal, and $\alpha$ is roughness.
- **Outputs:** Aggregated specular radiance contribution.

**Data Structures:**
- **SGGX Mixture Model:** The surface Normal Distribution Function (NDF) $p_N(n)$ is represented as a mixture of $k$ SGGX lobes:
  $$ p_N(n) = \sum_{i=1}^k w_i D_{i,sggx}(n) $$
  where $w_i$ are positive weights summing to 1 [S1, unofficial].
- **Precomputed Tables:**
  - $D_{conv}(\omega_h)_{ij}$: Convolved NDF values for each lobe pair $(i,j)$, stored in a 1D or 2D table depending on the approximation.
  - $S_{ij}(\omega_i, \omega_o)$: Shadowing term lookup table, parameterized by $\alpha_{ij}$ [S1, unofficial].
  - $M_5(\alpha)$: Normalization term for the GGX in the spherical domain [S1, unofficial].

**Numbered Steps:**

1.  **Substitute and Expand:** Substitute the specular BRDF $f_s(\omega_i, \omega_o; \gamma)$ into the ABSDF integral (Eq. 7 of [S1, unofficial]). Expand the integral into two terms based on the Fresnel mix $R = \beta_m \beta_c + (1 - \beta_m)\beta_s$ [S1, unofficial]:
    $$ \int_{\Gamma} f_s(\omega_i, \omega_o; \gamma) \langle n \cdot \omega_i \rangle \langle n \cdot \omega_o \rangle p_Y(\gamma) d\gamma = \frac{1}{4} \int_{\Gamma} (1 - F_c) R D p_Y(\gamma) d\gamma + F_c \int_{\Gamma} R D p_Y(\gamma) d\gamma $$
    where $F_c$ is the clear coat Fresnel term, $D = D(\omega_h; n, \alpha)G(\omega_i, \omega_o; n, \alpha)1(n \cdot \omega_o)1(n \cdot \omega_i)$, and $1(\cdot)$ is the Heaviside step function [S1, unofficial].

2.  **Split Probability Density:** Split the joint probability density $p_Y(\gamma)$ into independent components for the Fresnel/roughness mix parameters and the geometric parameters:
    $$ p_Y(\gamma) \approx p_{Y1}(\beta_c, \beta_m, \beta_s) p_{Y2}(n, \alpha) $$
    This allows the integral over the product space $\Gamma = \Gamma_1 \times \Gamma_2$ to be separated [S1, unofficial]:
    $$ \int_{\Gamma} R D p_Y(\gamma) d\gamma \approx \left( \int_{\Gamma_1} R p_{Y1}(\gamma) d\gamma \right) \left( \int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma \right) $$
    where $\Gamma_1$ is the space of $(\beta_c, \beta_m, \beta_s)$ and $\Gamma_2$ is the space of $(n, \alpha)$ [S1, unofficial].

3.  **Evaluate Parameter Integral (Moments):** Evaluate the integral over $\Gamma_1$ as moments of the distribution $p_{Y1}$. The term $\int_{\Gamma_1} R p_{Y1}(\gamma) d\gamma$ becomes:
    $$ E[\beta_m \beta_c] + E[\beta_s] - E[\beta_m \beta_s] $$
    These moments are precomputed or queried based on the material's statistical distribution [S1, unofficial].

4.  **Evaluate Microfacet Integral (Convolution):** Evaluate the integral over $\Gamma_2$ involving $D$. Since $p_N(n)$ is a mixture of SGGX lobes, the convolution of each lobe $D_{i,sggx}$ with the isotropic spherical distribution $g(\omega; n)$ is approximated by a roughened SGGX:
    $$ \int g(\omega; n) D_{i,sggx}(n; R, \alpha) dn \approx D_{i,sggx}(\omega; R, \alpha + \Delta \alpha) $$
    where $R$ stays fixed but $\alpha$ gains additional values representing the aggregation blur [S1, unofficial].

5.  **Apply Shadowing-Masking:** The shadowing-masking term $G$ is very smooth. It is multiplied to each lobe post-convolution. The final expression for the microfacet integral is:
    $$ \int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma \approx \sum_{i=1}^k \sum_{j=1}^2 w_i m_j D_{conv}(\omega_h)_{ij} S_{ij}(\omega_i, \omega_o) G_{ij}(\omega_i, \omega_o) $$
    where $S_{ij}(\omega_i, \omega_o) = S(\omega_i, \omega_o; \alpha_{ij})$ is the shadowing term and $G_{ij}$ is the shadowing-masking term [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def specular_absdf_factorization(omega_i, omega_o, material_params):
    """
    Computes the specular ABSDF contribution.
    
    Args:
        omega_i: Incident light direction.
        omega_o: Outgoing view direction.
        material_params: Object containing:
            - beta_c, beta_m, beta_s: Fresnel/roughness mix parameters.
            - moments: Precomputed E[beta_m * beta_c], E[beta_s], E[beta_m * beta_s].
            - n_samples: List of surface samples with normals n and roughness alpha.
            - weights: Weights w_i for the SGGX mixture model.
            - k: Number of SGGX lobes.
            - partition: Spherical domain partition (e.g., 3x3) for orientation-varying params.
    
    Returns:
        float: Aggregated specular radiance.
    """
    # 1. Calculate Fresnel Mix Moments (Gamma 1 integral)
    # E[R] = E[beta_m * beta_c] + E[beta_s] - E[beta_m * beta_s]
    moment_beta_m_beta_c = material_params.moments['beta_m_beta_c']
    moment_beta_s = material_params.moments['beta_s']
    moment_beta_m_beta_s = material_params.moments['beta_m_beta_s']
    
    expected_R = moment_beta_m_beta_c + moment_beta_s - moment_beta_m_beta_s
    
    # 2. Calculate Geometric Integral (Gamma 2 integral)
    # Sum over SGGX lobes and partition bins
    geometric_integral = 0.0
    
    # Determine half-vector
    omega_h = normalize(omega_i + omega_o)
    
    for i in range(material_params.k):
        w_i = material_params.weights[i]
        
        # For each lobe, sum over partition bins (j)
        # Note: The source mentions sum_{j=1}^2, likely referring to a coarse partition or 
        # specific approximation steps. We iterate over the partition bins.
        for j in range(material_params.partition.num_bins):
            # Query precomputed convolved NDF for lobe i and bin j
            # D_conv depends on omega_h and the aggregated roughness for this bin
            d_conv = query_precomputed_table_D_conv(omega_h, i, j)
            
            # Query shadowing term S
            # S depends on omega_i, omega_o, and the roughness alpha_ij for this bin
            alpha_ij = get_roughness_for_bin(i, j, material_params)
            s_ij = query_shadowing_table_S(omega_i, omega_o, alpha_ij)
            
            # Query shadowing-masking term G
            # G depends on omega_i, omega_o, and the transformed roughness
            g_ij = compute_shadowing_masking(omega_i, omega_o, alpha_ij)
            
            # Get the weight m_j for the partition bin (from splatting process)
            m_j = get_partition_weight(i, j, material_params)
            
            geometric_integral += w_i * m_j * d_conv * s_ij * g_ij

    # 3. Combine terms
    # The full expression involves F_c (clear coat Fresnel)
    # Eq 15: 1/4 * (1 - F_c) * R * D + F_c * R * D
    # Factored: R * D * [ 1/4 * (1 - F_c) + F_c ]
    
    f_c = compute_clear_coat_fresnel(omega_i, omega_o) # Simplified Fresnel calculation
    
    term_inside_bracket = 0.25 * (1.0 - f_c) + f_c
    
    result = expected_R * geometric_integral * term_inside_bracket
    
    return result
```

**Parameters:**
- **Spherical Domain Partition Size:** Coarse partitioning such as $3 \times 3$ is sufficient because angular frequency decreases as the scale of aggregation becomes larger [S1, unofficial].
- **SGGX Mixture Weights:** $w_i$ must be positive and sum to 1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The Heaviside functions $1(n \cdot \omega_o)$ and $1(n \cdot \omega_i)$ must be handled carefully to avoid discontinuities at grazing angles [S1, unofficial].
- **Degenerate Input:** If the surface samples are degenerate (e.g., all normals aligned), the SGGX mixture may collapse. The fitting process in §7.1 of [S1, unofficial] handles this, but runtime queries must handle empty bins.
- **Orientation-Varying Parameters:** If parameters vary significantly with orientation, a coarse $3 \times 3$ partition may introduce artifacts. However, the source states this is usually sufficient [S1, unofficial].

## Subtopics

### Final Specular ABSDF Expression
The complete specular ABSDF expression is the sum of the two terms derived from the Fresnel mix expansion. It consists of a "clear coat" term weighted by $(1-F_c)/4$ and a "base" term weighted by $F_c$, both multiplied by the aggregated geometric term [S1, unofficial]. The geometric term is a sum over SGGX lobes and partition bins, where each term includes the convolved NDF, shadowing, and shadowing-masking terms [S1, unofficial].

### Orientation-Varying Parameters
An extension of the factorization to handle material parameters that vary with surface orientation. Instead of global moments, directional moments $E_\omega[\beta]$ and $E_\omega[\beta \beta']$ are computed by splatting surface samples to the spherical domain using a kernel function $s(T_x \omega)$ [S1, unofficial].
- **Kernel Selection:** For the specular component, the kernel $s(\omega)$ is the microfacet distribution. For the diffuse component, it is a Spherical Gaussian (SG) [S1, unofficial].
- **Computation:** Each surface sample is weighted by the kernel and integrated over the surface area $A$ [S1, unofficial].
- **Querying:** Directional moments are queried at $\omega_h$ when evaluating the ABSDF equations [S1, unofficial].
- **Precomputation:** Surface samples are splatted to the spherical partition during precomputation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(k \cdot P)$ per pixel, where $k$ is the number of SGGX lobes and $P$ is the number of spherical partition bins (typically $3 \times 3 = 9$). This is constant time relative to scene size, making it suitable for real-time rendering [S1, unofficial].
- **Memory Complexity:** Depends on the size of the precomputed tables for $D_{conv}$ and $S$. The tables are small (1D or 2D) and depend on roughness and view angles, not scene resolution [S1, unofficial].
- **Measured Performance:** Validated in the context of Level-of-Detail (LOD) rendering, showing efficient aggregation of complex microfacet appearances [S1, unofficial].
- **Strengths:** Handles arbitrary orientation distributions and complex material parameter variations efficiently. Separates precomputation from runtime evaluation.
- **Limitations:** Requires precomputation of moments and tables. Approximation errors may occur if the SGGX mixture does not fit the underlying NDF well or if the partition is too coarse for high-frequency variations [S1, unofficial].

## Background
The method originates from the paper "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" by Yang Zhou, Tao Huang, Ravi Ramamoorthi, Pradeep Sen, and Ling-Qi Yan [S1, unofficial]. It builds upon the ABSDF framework and the SGGX distribution model [Heitz et al. 2015] to provide a compact representation for aggregated microfacet appearances. The factorization technique allows for the efficient handling of orientation-varying parameters, which is crucial for realistic rendering of large-scale scenes with complex materials [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
