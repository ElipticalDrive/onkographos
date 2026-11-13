# Specular ABSDF Factorization

Also called: Specular Aggregation

## Overview
Specular ABSDF Factorization is a method for computing the aggregated specular response of a surface with varying material parameters (roughness, orientation) by factorizing the integral of the microfacet BRDF over a probability distribution of these parameters [S1, unofficial]. It solves the problem of efficiently evaluating the expectation of a complex microfacet distribution when the roughness $\alpha$ and normal $n$ are not single values but follow a joint probability density function $p_Y(\gamma)$ [S1, unofficial]. It sets itself apart from its siblings by explicitly separating the Fresnel mix terms from the microfacet distribution convolution, allowing the shadowing-masking term $G$ to be applied post-convolution due to its smoothness, and supporting orientation-varying parameters via spherical kernel splatting [S1, unofficial].

## How it works
The method computes the aggregated specular term by substituting the specular BRDF $f_s$ into the Aggregated BSDF (ABSDF) integral and expanding it into terms involving the Fresnel mix $R$ and the microfacet distribution $D$ [S1, unofficial].

**Inputs and Outputs:**
- **Input:** The probability density function $p_Y(\gamma)$ of material parameters $\gamma = (n, \alpha, \beta_c, \beta_m, \beta_s)$, where $n$ is the normal, $\alpha$ is the roughness, and $\beta$ terms are Fresnel coefficients [S1, unofficial].
- **Output:** The aggregated specular radiance contribution.

**Data Structures:**
- **SGGX Mixture:** The surface Normal Distribution Function (NDF) $p_N(n)$ is represented as a mixture of $k$ SGGX lobes: $p_N(n) = \sum_{i=1}^k w_i D_{i,sggx}(n)$, where weights $w_i$ are positive and sum to 1 [S1, unofficial].
- **Precomputed Tables:** Moments of the microfacet distribution $M_5(\alpha)$ and convolved distributions $D_{conv}$ are stored in lookup tables [S1, unofficial].

**Steps:**
1.  **Substitution and Expansion:** Substitute $f_s(\omega_i, \omega_o; \gamma)$ into the ABSDF integral (Eq. 7) and expand into terms involving $R$ (Fresnel mix) and $D$ (microfacet distribution) [S1, unofficial]:
    $$
    \int_{\Gamma} f_s(\omega_i, \omega_o; \gamma) \langle n \cdot \omega_i \rangle \langle n \cdot \omega_o \rangle p_Y(\gamma) d\gamma = \frac{1}{4} \int_{\Gamma} (1 - F_c) R D p_Y(\gamma) d\gamma + F_c \int_{\Gamma} R D p_Y(\gamma) d\gamma
    $$
    where $R = \beta_m \beta_c + (1 - \beta_m)\beta_s$ and $D = D(\omega_h; n, \alpha) G(\omega_i, \omega_o; n, \alpha) 1(n \cdot \omega_o) 1(n \cdot \omega_i)$ [S1, unofficial].

2.  **Splitting the PDF:** Split the joint PDF $p_Y(\gamma)$ into independent components for the Fresnel coefficients and the geometric parameters: $p_Y(\gamma) \approx p_{Y1}(\beta_c, \beta_m, \beta_s) p_{Y2}(n, \alpha)$ [S1, unofficial].

3.  **Evaluate Parameter Integral:** Evaluate the integral over the Fresnel parameters $\Gamma_1$ as moments:
    $$
    \int_{\Gamma_1} R p_{Y1}(\gamma) d\gamma = E[\beta_m \beta_c] + E[\beta_s] - E[\beta_m \beta_s]
    $$
    [S1, unofficial].

4.  **Evaluate Microfacet Integral:** Evaluate the integral over the geometric parameters $\Gamma_2$ using precomputed tables. The convolution of the SGGX mixture with an isotropic spherical distribution $g(\omega; n)$ is approximated by a roughened SGGX:
    $$
    \int g(\omega; n) D_{sggx}(n; R, \alpha) dn \approx D_{sggx}(\omega; R, \alpha + \Delta)
    $$
    where $R$ stays fixed but $\alpha$ gains additional values [S1, unofficial].

5.  **Final Expression:** Multiply the shadowing-masking term $G$ to each lobe post-convolution, as it is very smooth [S1, unofficial]:
    $$
    \int_{\Gamma_2} D p_{Y2}(\gamma) d\gamma \approx \sum_{i=1}^{2} \sum_{j=1}^{k} w_i m_j D_{conv}(\omega_h)_{ij} S_{ij}(\omega_i, \omega_o) G_{ij}(\omega_i, \omega_o)
    $$
    where $S_{ij} = S(\omega_i, \omega_o; \alpha_{ij})$ and $G_{ij}$ is the shadowing-masking term [S1, unofficial].

**Parameters:**
- **Spherical Domain Partition Size:** Coarse partitioning such as $3 \times 3$ is sufficient for orientation-varying parameters because angular frequency decreases as the scale of aggregation becomes larger [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The approximation of the convolution as a roughened SGGX relies on the validity of the eigenbasis parameterization [S1, unofficial].
- **Boundaries:** The Heaviside step functions $1(n \cdot \omega_o)$ and $1(n \cdot \omega_i)$ must be handled correctly to ensure clamped dot products [S1, unofficial].

## Subtopics

### [Aggregated Microfacet Distribution Approximation](10880-aggregated-microfacet-distribution-approximation.md)
This method approximates the statistical aggregation of microfacet distributions over a distribution of roughness values, enabling efficient rendering of scenes with material variation. It solves the problem of computing the expectation of a microfacet distribution $D_{mic}$ when the roughness parameter $\alpha$ is not a single value but follows a probability density function $p_A(\alpha)$. Unlike single-lobed approximations that simply average roughness, this method uses a two-lobed weighted sum to capture the statistical spread, providing higher accuracy while maintaining computational efficiency.

### Final Specular ABSDF Expression
The final expression for the specular component involves multiplying the shadowing-masking term $G$ to each lobe post-convolution. The expression consists of contributions from all convolved NDF lobes, where each term is multiplied by its scaling factor to account for the specific roughness and orientation [S1, unofficial].

### Orientation-Varying Parameters
This is an extension of the factorization to handle orientation-varying material parameters by computing directional moments weighted by a spherical kernel. It allows the model to capture view-dependent appearance changes due to orientation-varying material parameters [S1, unofficial].

**Steps:**
1.  For a direction $\omega$, weight each surface sample by a spherical function $s(T_x \omega)$ as a kernel [S1, unofficial].
2.  Compute directional moments $E_\omega[\beta]$ and $E_\omega[\beta \beta']$ by integrating over the surface area $A$:
    $$
    E_\omega[\beta] = \frac{\int_A \beta_x s(T_x \omega) dx}{\int_A s(T_x \omega) dx}, \quad E_\omega[\beta \beta'] = \frac{\int_A \beta_x \beta'_x s(T_x \omega) dx}{\int_A s(T_x \omega) dx}
    $$
    where $T_x$ is the local transform at $x$ [S1, unofficial].
3.  For the specular component, set the kernel $s(\omega)$ to the microfacet distribution. For the diffuse component, set the kernel $s(\omega)$ to the Spherical Gaussian (SG) from Eq. 14 [S1, unofficial].
4.  Query the directional moments at $\omega_h$ when evaluating the specular and diffuse components [S1, unofficial].
5.  During precomputation, splat each surface sample to the spherical partition [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method reduces the complexity of aggregating microfacet distributions by precomputing moments and convolutions, allowing for efficient runtime evaluation using lookup tables [S1, unofficial].
- **Memory Complexity:** Requires storage for precomputed tables of moments and convolved distributions, as well as the spherical partition for orientation-varying parameters [S1, unofficial].
- **Strengths:** Efficiently handles material variation and orientation changes without requiring per-pixel integration of complex microfacet distributions [S1, unofficial].
- **Limitations:** The accuracy depends on the approximation of the convolution as a roughened SGGX and the coarse partitioning of the spherical domain [S1, unofficial].

## Background
Specular ABSDF Factorization was introduced in the paper "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" by Yang Zhou, Tao Huang, Ravi Ramamoorthi, Pradeep Sen, and Ling-Qi Yan [S1, unofficial]. The method builds on previous work in microfacet theory and aggregation, such as the SGGX distribution [Heitz et al. 2015] and the aggregation of roughness distributions [Xu et al. 2013] [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
