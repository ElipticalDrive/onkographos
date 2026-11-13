# Specular ABSDF Factorization

Also called: Specular Aggregation

## Overview
Specular ABSDF Factorization is a method for aggregating the specular component of the Aggregated Bidirectional Scattering Distribution Function (ABSDF) [10872-specular-absdf-factorization.md]. It solves the problem of efficiently rendering heterogeneous materials with varying roughness and orientation by factorizing the complex integral of the microfacet BRDF into separable terms involving Fresnel mix, microfacet distribution, and shadowing-masking [S1, unofficial]. It is distinguished from its sibling, [Phys3DGS BRDF Function](11071-phys3dgs-brdf-function.md), by explicitly handling the statistical aggregation of roughness parameters via moment-based approximation rather than a direct physical simulation [S1, unofficial].

## How it works
The method factorizes the specular integral of the ABSDF by separating the Fresnel term, the microfacet distribution, and the shadowing-masking term, allowing for precomputation and efficient runtime evaluation.

**Inputs and Outputs:**
*   **Inputs:** Incident direction $\omega_i$, outgoing direction $\omega_o$, material parameter distribution $p_Y(\gamma)$, where $\gamma$ includes Fresnel mix parameters $(\beta_c, \beta_m, \beta_s)$ and microfacet parameters $(n, \alpha)$.
*   **Outputs:** Aggregated specular radiance contribution.

**Data Structures:**
*   **SGGX Mixture Model:** The surface Normal Distribution Function (NDF) $p_N(n)$ is represented as a mixture of $k$ SGGX lobes:
    $$ p_N(n) = \sum_{i=1}^{k} w_i D_{i,sggx}(n) $$
    where weights $w_i$ are positive and sum to 1 [S1, unofficial].
*   **Convolved NDF Table:** Precomputed tables for the convolution of SGGX lobes with an isotropic spherical distribution $g(\omega; n)$, resulting in roughened SGGX lobes $D_{conv}$ [S1, unofficial].
*   **Shadowing-Masking Table:** Precomputed or analytic values for the shadowing term $S$ and masking term $G$ [S1, unofficial].

**Numbered Steps:**

1.  **Substitute and Expand:** Substitute the specular BRDF $f_s(\omega_i, \omega_o; \gamma)$ into the ABSDF integral (Eq. 7 of the parent context). Expand the integral into terms involving the Fresnel mix $R$ and the microfacet distribution $D$ [S1, unofficial]:
    $$ \int_{\Gamma} f_s(\omega_i, \omega_o; \gamma) \langle n \cdot \omega_i \rangle \langle n \cdot \omega_o \rangle p_Y(\gamma) d\gamma = \frac{1}{4} \int_{\Gamma} (1 - F_c) R D p_Y(\gamma) d\gamma + F_c \int_{\Gamma} R D p_Y(\gamma) d\gamma $$
    where $F_c$ is the constant Fresnel term, and $R = \beta_m \beta_c + (1 - \beta_m)\beta_s$ [S1, unofficial].

2.  **Split Probability Distribution:** Split the joint probability distribution $p_Y(\gamma)$ into independent components for Fresnel mix and microfacet parameters:
    $$ p_Y(\gamma) \approx p_{Y1}(\beta_c, \beta_m, \beta_s) p_{Y2}(n, \alpha) $$
    This allows the integral to be separated into a parameter integral (over $\Gamma_1$) and a microfacet integral (over $\Gamma_2$) [S1, unofficial]:
    $$ \int_{\Gamma} R D p_Y(\gamma) d\gamma \approx \left( \int_{\Gamma_1} R p_{Y1}(\gamma) d\gamma \right) \left( \int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma \right) $$

3.  **Evaluate Parameter Integral (Moments):** Evaluate the Fresnel mix integral as statistical moments of the distribution $p_{Y1}$:
    $$ \int_{\Gamma_1} R p_{Y1}(\gamma) d\gamma = E[\beta_m \beta_c] + E[\beta_s] - E[\beta_m \beta_s] $$
    These moments ($E[\beta_m \beta_c]$, $E[\beta_s]$, $E[\beta_m \beta_s]$) are precomputed or queried based on the material aggregation [S1, unofficial].

4.  **Evaluate Microfacet Integral (Convolution):** Evaluate the microfacet integral $\int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma$. Since $p_N(n)$ is a mixture of SGGX lobes, the convolution is performed per lobe. The convolution of an SGGX lobe $D_{sggx}$ with an isotropic distribution $g$ is approximated by a roughened SGGX:
    $$ \int g(\omega; n) D_{sggx}(n; R, \alpha) dn \approx D_{sggx}(\omega; R, \alpha + \Delta\alpha) $$
    where $R$ is the eigenbasis (fixed) and $\alpha$ is the anisotropic roughness (increased by the convolution) [S1, unofficial].

5.  **Apply Shadowing-Masking:** Multiply the shadowing-masking term $G$ to each lobe post-convolution, as $G$ is very smooth [S1, unofficial]. The final expression for the microfacet integral is:
    $$ \int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma \approx \sum_{i=1}^{k} \sum_{j=1}^{2} w_i m_j D_{conv}(\omega_h)_{ij} S_{ij}(\omega_i, \omega_o) G_{ij}(\omega_i, \omega_o) $$
    where $S_{ij} = S(\omega_i, \omega_o; \alpha_{ij})$ is the shadowing term and $G_{ij}$ is the masking term [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def specular_absdf_factorization(omega_i, omega_o, material_aggregation):
    """
    Computes the aggregated specular BRDF using ABSDF factorization.
    
    Args:
        omega_i: Incident light direction.
        omega_o: Outgoing view direction.
        material_aggregation: Object containing precomputed moments and SGGX mixture parameters.
            - moments: dict with keys 'E_beta_m_beta_c', 'E_beta_s', 'E_beta_m_beta_s'
            - sggx_mixture: list of k SGGX lobes, each with weights w_i, eigenbasis R_i, roughness alpha_i
            - convolved_ndf_table: lookup table for D_conv(omega_h, R, alpha)
            - shadowing_table: lookup table for S(omega_i, omega_o, alpha)
            - masking_table: lookup table for G(omega_i, omega_o, R, alpha)
            
    Returns:
        float: Aggregated specular radiance contribution.
    """
    # Step 1: Calculate Fresnel mix moments
    # R_avg = E[beta_m * beta_c] + E[beta_s] - E[beta_m * beta_s]
    R_avg = (material_aggregation.moments['E_beta_m_beta_c'] + 
             material_aggregation.moments['E_beta_s'] - 
             material_aggregation.moments['E_beta_m_beta_s'])
    
    # Step 2: Calculate the microfacet integral part
    # Sum over SGGX mixture lobes
    microfacet_integral = 0.0
    for i in range(len(material_aggregation.sggx_mixture)):
        lobe = material_aggregation.sggx_mixture[i]
        w_i = lobe.weight
        R_i = lobe.eigenbasis
        alpha_i = lobe.roughness
        
        # Calculate half-vector
        omega_h = normalize(omega_i + omega_o)
        
        # Get convolved NDF value from precomputed table
        # D_conv depends on omega_h, R_i, and roughened alpha_i
        D_conv = material_aggregation.convolved_ndf_table.query(omega_h, R_i, alpha_i)
        
        # Get shadowing term S from precomputed table
        S_ij = material_aggregation.shadowing_table.query(omega_i, omega_o, alpha_i)
        
        # Get masking term G from precomputed table
        G_ij = material_aggregation.masking_table.query(omega_i, omega_o, R_i, alpha_i)
        
        # Accumulate contribution
        # Term: w_i * m_j * D_conv * S * G
        # Note: m_j is part of the microfacet parameter distribution moments, 
        # assumed to be incorporated into the precomputed D_conv or S/G tables 
        # or handled as a separate moment if not constant. 
        # Based on Eq 22, m_j is a coefficient from p_Y2. 
        # For simplicity, assuming m_j is 1 or absorbed in w_i for this generic implementation.
        # If m_j varies, it should be queried similarly to moments.
        microfacet_integral += w_i * D_conv * S_ij * G_ij
        
    # Step 3: Combine Fresnel and Microfacet parts
    # Formula: 1/4 * (1 - F_c) * R_avg * microfacet_integral + F_c * R_avg * microfacet_integral
    # Note: The original formula has R inside the integral. 
    # The factorization splits R and D. 
    # The term R_avg is the expectation of R.
    # The term microfacet_integral is the expectation of D (including G).
    # The formula in [S1, unofficial] Eq 15 is:
    # 1/4 * int((1-Fc) R D p) + Fc * int(R D p)
    # = 1/4 * (1-Fc) * int(R D p) + Fc * int(R D p)
    # = (1/4 * (1-Fc) + Fc) * int(R D p)
    # = (1/4 + 3/4 * Fc) * int(R D p)
    # And int(R D p) = R_avg * microfacet_integral
    
    Fc = material_aggregation.constant_fresnel # F_c is a constant parameter
    
    factor = 0.25 * (1 - Fc) + Fc
    result = factor * R_avg * microfacet_integral
    
    return result
```

**Parameters:**
*   **SGGX Mixture Size ($k$):** Not given in the sources.
*   **Spherical Domain Partition Size:** Coarse partitioning such as $3 \times 3$ is sufficient for orientation-varying parameters [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The convolution approximation relies on the assumption that the roughened SGGX is a valid approximation of the convolved distribution [S1, unofficial].
*   **Boundaries:** The shadowing-masking term $G$ is assumed to be very smooth, allowing it to be multiplied post-convolution. This may not hold for extremely rough or anisotropic surfaces [S1, unofficial].
*   **Degenerate Input:** If the material aggregation has zero weight for all SGGX lobes, the result is zero [G].

## Subtopics

### [Aggregated Microfacet Distribution Approximation](10880-aggregated-microfacet-distribution-approximation.md)
This method approximates the aggregated microfacet distribution $\hat{D}_{mic}(\omega_h)$, which represents the statistical expectation of microfacet distributions over a distribution of roughness values $\alpha$. It solves the problem of efficiently representing complex, heterogeneous material roughness in a scene by fitting a parametric roughness distribution (Beta) to a weighted sum of two GGX lobes. It is distinguished from its parent, Specular ABSDF Factorization, by providing the specific mathematical mechanism for aggregating roughness statistics into a tractable BRDF approximation [10872-specular-absdf-factorization.md].

### Final Specular ABSDF Expression
This variant finalizes the specular ABSDF expression by multiplying the shadowing-masking term $G$ to each lobe post-convolution, as it is very smooth [S1, unofficial]. The expression is:
$$ \int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma \approx \sum_{i=1}^{k} \sum_{j=1}^{2} w_i m_j D_{conv}(\omega_h)_{ij} S_{ij}(\omega_i, \omega_o) G_{ij}(\omega_i, \omega_o) $$
where $S_{ij} = S(\omega_i, \omega_o; \alpha_{ij})$ and $G_{ij}$ is the shadowing-masking term [S1, unofficial].

### Orientation-Varying Parameters
This extension handles orientation-varying material parameters by computing directional moments weighted by a spherical kernel [S1, unofficial]. It allows the model to capture view-dependent appearance changes due to orientation-varying material parameters [S1, unofficial].

**Steps:**
1.  For a direction $\omega$, weight each surface sample by a spherical function $s(T_x \omega)$ as a kernel [S1, unofficial].
2.  Compute directional moments $E_\omega[\beta]$ and $E_\omega[\beta \beta']$ by integrating over the surface area $A$:
    $$ E_\omega[\beta] = \frac{\int_A \beta_x s(T_x \omega) dx}{\int_A s(T_x \omega) dx}, \quad E_\omega[\beta \beta'] = \frac{\int_A \beta_x \beta'_x s(T_x \omega) dx}{\int_A s(T_x \omega) dx} $$
    where $T_x$ is the local transform at $x$ [S1, unofficial].
3.  For the specular component, set the kernel $s(\omega)$ to the microfacet distribution. For the diffuse component, set the kernel $s(\omega)$ to the SG (Spherical Gaussian) from Eq. 14 [S1, unofficial].
4.  Query the directional moments at $\omega_h$ when evaluating Eq. 13 and Eq. 17 [S1, unofficial].
5.  During precomputation, splat each surface sample to the spherical partition [S1, unofficial].

**Parameters:**
*   **Spherical Domain Partition Size:** Coarse partitioning such as $3 \times 3$ is sufficient because angular frequency decreases as the scale of aggregation becomes larger [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The runtime cost is dominated by the evaluation of the SGGX mixture and the lookup of precomputed tables for $D_{conv}$, $S$, and $G$. The number of lobes $k$ in the SGGX mixture directly affects performance.
*   **Memory Complexity:** Memory usage is proportional to the size of the precomputed tables for $D_{conv}$, $S$, and $G$, and the storage of the SGGX mixture parameters and moments. For orientation-varying parameters, memory increases with the number of spherical partitions (e.g., $3 \times 3$) [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Efficiently handles heterogeneous materials with varying roughness and orientation. The factorization allows for precomputation and reduces the complexity of the integral evaluation [S1, unofficial].
*   **Limitations:** Relies on approximations (convolution of SGGX, smoothness of $G$) which may not be accurate for all material types. The accuracy depends on the quality of the SGGX mixture fit [S1, unofficial].
*   **When to Use:** Suitable for level-of-detail rendering where material heterogeneity needs to be aggregated efficiently [S1, unofficial].
*   **When Not to Use:** Not suitable for highly anisotropic or complex microfacet distributions that cannot be well-approximated by a mixture of SGGX lobes [S1, unofficial].

## Background
The method originates from the paper "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" [S1, unofficial]. It builds on the ABSDF framework and extends it to handle specular components with heterogeneous material properties. The use of SGGX distributions and moment-based aggregation is inspired by previous work on microfacet models and statistical aggregation [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
