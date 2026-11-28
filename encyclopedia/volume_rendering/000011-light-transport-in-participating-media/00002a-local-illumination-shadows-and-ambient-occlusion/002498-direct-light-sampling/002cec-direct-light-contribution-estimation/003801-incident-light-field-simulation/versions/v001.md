# Incident Light Field Simulation

Also called: Direct and Indirect Light Approximation

## Overview
Incident Light Field Simulation estimates the total incident radiance $L_i$ at a surface point by combining direct illumination from light sources, attenuated direct light, and indirect illumination reflected from other surfaces [S1, unofficial]. It differs from sibling methods like [Direct Illumination Derivative (Spatial Form)](7662-direct-illumination-derivative-spatial-form.md) by explicitly modeling indirect light contributions via neural networks or spherical harmonics, rather than relying solely on spatial derivatives of direct light [S1, unofficial][S2].

## How it works
The method computes incident light $L_i(x, d)$ at a surface point $x$ with view direction $d$ using three components: direct light from sources, direct light modulated by transmittance, and indirect light [S1, unofficial]. Two primary implementations exist: one using Multi-Spherical Gaussians (SGs) and an MLP for indirect light [S1, unofficial], and another using a hybrid of Spherical Harmonics (SH) and environment maps [S2].

### Inputs and Outputs
**Inputs:**
*   Surface point position $x$.
*   View direction $d$ (or outgoing direction $\omega_o$).
*   Appearance feature $a(x)$ (optional, depending on variant).
*   Surface normal $n$ (for PBR integration).
*   Light source parameters (for SG fitting or environment map lookup).

**Outputs:**
*   Incident radiance $L_i(x, d)$ (RGB value).
*   PBR color $c'(\omega_o)$ (in the hybrid SH variant).

### Data Structures
1.  **Multi-Spherical Gaussian (SG) Parameters:** A set of $K=32$ tuples $(\mu_k, \lambda_k, a_k)$ representing the direct light environment map [S1, unofficial].
    *   $\mu_k$: Mean direction (unit vector).
    *   $\lambda_k$: Sharpness parameter.
    *   $a_k$: Amplitude (RGB).
2.  **MLP Network:** A neural network mapping $\{a(x), x, d\}$ to indirect light [S1, unofficial].
3.  **Hybrid Light Representation (Variant):**
    *   Global Environment Map $l_{env}$: 16x32 resolution map for direct light [S2].
    *   Per-Gaussian SH Coefficients $l$: 3-level Spherical Harmonics coefficients for indirect light [S2].

### Formulas
**Spherical Gaussian Function [S1, unofficial]:**
$$ G(v; \mu, \lambda, a) = a e^{\lambda(\mu \cdot \nu - 1)} $$
Where $v$ is the light angle, $\mu, a, \lambda$ are trainable parameters, and $\nu$ is the direction vector.

**Direct Light Approximation [S1, unofficial]:**
$$ L_{dir}(x, \theta_i) = \sum_{k=1}^{32} G(\theta_i; \mu_k, \lambda_k, a_k) $$

**Transmittance [S1, unofficial]:**
$$ T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s)) ds\right) $$
Where $t_n$ is the start of the segment, $t$ is the current point, and $\sigma$ is the volume density. This calculates the receiving rate of direct light for the last sampled point along the ray [S1, unofficial].

**Indirect Light (MLP Variant) [S1, unofficial]:**
$$ L_{idxval} = \text{MLP}(a(x), x, d) $$
Where $L_{idxval}$ is the 3-channel indirect light RGB value [S1, unofficial].

**Total Incident Light (MLP Variant) [S1, unofficial]:**
$$ L_i(x, d) = L_{dir}(x, d) * T(x, d) + L_{idxval} $$

**Hybrid SH/Env Map Variant [S2]:**
$$ L_i(\boldsymbol{\omega}_i) = V(\boldsymbol{\omega}_i) \cdot L_{direct}(\boldsymbol{\omega}_i) + L_{indirect}(\boldsymbol{\omega}_i) $$
Where $V(\omega_i)$ is visibility, $L_{direct}$ is from the global environment map, and $L_{indirect}$ is parameterized by 3-level SH [S2].

**PBR Color Integration (Hybrid Variant) [S2]:**
$$ \boldsymbol{c'}(\boldsymbol{\omega}_o) = \sum_{i=0}^{N_s} (f_d + f_s(\boldsymbol{\omega}_o, \boldsymbol{\omega}_i)) L_i(\boldsymbol{\omega}_i) (\boldsymbol{\omega}_i \cdot \boldsymbol{n}) \Delta \boldsymbol{\omega}_i $$
Where $f_d, f_s$ are diffuse and specular BRDFs, $N_s$ is the number of samples, and $\Delta \boldsymbol{\omega}_i$ is the solid angle [S2].

### Steps
1.  **Represent Direct Light:**
    *   *SG Variant:* Fit 32 SGs to the environment map to approximate direct light [S1, unofficial].
    *   *Hybrid Variant:* Use a globally shared 16x32 environment map for direct light [S2].
2.  **Calculate Transmittance:**
    *   Compute $T(t)$ for the last sampled point along the ray using the volume density integral [S1, unofficial].
3.  **Estimate Indirect Light:**
    *   *SG Variant:* Query an MLP with appearance feature $a$, position $x$, and view direction $d$ to get $L_{idxval}$ [S1, unofficial].
    *   *Hybrid Variant:* Evaluate 3-level SH coefficients $l$ for the specific Gaussian to get $L_{indirect}$ [S2].
4.  **Combine Components:**
    *   *SG Variant:* Compute $L_i = L_{dir} * T + L_{idxval}$ [S1, unofficial].
    *   *Hybrid Variant:* Compute $L_i = V \cdot L_{direct} + L_{indirect}$ [S2].
5.  **Render (Hybrid Variant Only):**
    *   Sample $N_s$ directions over the hemisphere using Fibonacci sampling [S2].
    *   Integrate using the PBR equation to get $\boldsymbol{c'}$ [S2].

### Parameters
*   **SG Count:** 32 [S1, unofficial].
*   **Environment Map Resolution:** 16x32 [S2].
*   **SH Level:** 3 [S2].
*   **Sample Count ($N_s$):** 64 (typical) [S2].
*   **MLP Inputs:** Appearance feature $a$, position $x$, view direction $d$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Visibility:** In the hybrid variant, the visibility term $V(\omega_i)$ must be correctly computed to avoid light leaking through occluders [S2].
*   **Material-Lighting Ambiguity:** Decomposing light and material is ill-posed; regularizations are required to prevent unrealistic results [S2].
*   **Transmittance Boundary:** The transmittance formula in the SG variant only calculates for the last sampled point, which may introduce errors if the ray segment is long [S1, unofficial].

## Subtopics

### Incident Light Modeling with Hybrid SH and Env Map
This variant partitions incident light into a globally shared direct component (16x32 environment map) and individual per-Gaussian indirect components (3-level SH). It avoids the computational burden of querying global neural fields for every Gaussian and aids in decomposing light from appearance. It uses Fibonacci sampling ($N_s=64$) for numerical integration. Choose this when using 3D Gaussians and needing efficient per-Gaussian indirect light without global neural queries. Link: [Incident Light Modeling with Hybrid SH and Env Map](7663-incident-light-modeling-hybrid-sh-env-map.md) (hypothetical link).

## Operating characteristics
*   **Time Complexity:** Depends on the number of SGs (32) or SH coefficients (3-level) and MLP inference cost [S1, unofficial][S2]. The hybrid variant adds $N_s$ samples (64) for integration [S2].
*   **Memory Complexity:** Low for SGs (32 vectors) and SH (fixed coefficients per Gaussian). Higher for MLP if stored globally [S1, unofficial][S2].
*   **Strengths:** Accurately models indirect lighting and shadows [S1, unofficial][S2]. Efficient compared to full path tracing [S2].
*   **Limitations:** Approximation errors in SG/SH fitting [S1, unofficial][S2]. Ambiguity between material and light properties [S2].
*   **When to Use:** Real-time or near-real-time relighting of complex scenes with participating media or surfaces [S1, unofficial][S2].
*   **When Not to Use:** When physically exact global illumination is required without approximations [S1, unofficial].

## Background
The method originates from tensor-based relighting radiance fields, specifically STensoRRF, which adapts incident light simulation from prior work [27, 35, 36] [S1, unofficial]. It was proposed in 2024 [S1, unofficial]. The hybrid SH variant was introduced in Relightable 3D Gaussaims (2024) to address the unconstrained nature of optimizing NeILF for each Gaussian [S2].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
