# Ensemble-averaged transmittance via characteristic functions

Also called: CF-based transmittance, Probabilistic ensemble-averaging

## Overview
This method computes the ensemble-averaged transmittance of non-exponential participating media by treating the micro-scale density as a stochastic process. It solves the problem of modeling light transport in media with spatial correlations (fractal noise) without instantiating discrete particles. It sets itself apart from its siblings by explicitly leveraging the **characteristic function (CF)** of the line-averaged density's probability distribution to derive closed-form transmittance expressions, whereas [Non-exponential transport kernel via macroscopic optical depth](14530-non-exponential-transport-kernel-via-macroscopic-optical-dep.md) uses macroscopic optical depth approximations and [Ensemble Transport Kernel](14536-ensemble-transport-kernel.md) focuses on the kernel formulation directly.

## How it works
**Inputs:**
- Distance $t$ along the ray.
- Medium parameters $\psi = \{\mu_\sigma, C, \beta\}$, where:
  - $\mu_\sigma$: Mean extinction coefficient.
  - $C$: Constant controlling the overall amplitude of density variation.
  - $\beta$: Spectral exponent ($-1 \le \beta \le 1$) controlling spatial correlations (roughness) of the underlying $1/f^\beta$ fractal noise.

**Outputs:**
- Ensemble-averaged transmittance $\text{ff}(t)$.

**Data Structures:**
- The method relies on the Probability Density Function (PDF) of the line-averaged micro-scale density, denoted $\bar{\sigma}_\mu(t)$. The specific layout of this PDF depends on the chosen noise model (e.g., Gaussian or Gamma), but the core method assumes a known Characteristic Function $\varphi_{\bar{\sigma}_\mu(t)}(r)$ for this variable.

**Steps:**
1. **Model the Density Field:** Define the micro-scale density field $\sigma_\mu(x)$ as a random process, typically characterized by $1/f^\beta$ fractal noise [S1].
2. **Determine the PDF:** Determine the probability density function $\text{pd}(\bar{\sigma}_\mu \mid t)$ for the line-averaged density $\bar{\sigma}_\mu(t)$ over distance $t$, given parameters $\psi$. The parameters of this distribution may depend on $t$ [S1].
3. **Identify the Characteristic Function:** Identify the characteristic function $\varphi_{\bar{\sigma}_\mu(t)}(r)$ corresponding to the PDF from Step 2. The characteristic function is defined as the Fourier transform of the PDF:
   $$ \varphi_X(r) = \langle e^{irX} \rangle = \int_{-\infty}^{\infty} e^{irx} \text{pd}(x) \, dx $$
   where $i = \sqrt{-1}$ [S1].
4. **Evaluate Transmittance:** Evaluate the characteristic function at the imaginary argument $r = ti$ to obtain the closed-form ensemble-averaged transmittance:
   $$ \text{ff}(t) = \varphi_{\bar{\sigma}_\mu(t)}(ti) $$
   [S1]

**Formula Derivation:**
The ensemble-averaged transmittance is the expectation of the exponential attenuation weighted by the PDF of the line-averaged density:
$$ \text{ff}(t) = \left\langle e^{-t\bar{\sigma}_\mu(t)} \right\rangle = \int_0^\infty e^{-t\bar{\sigma}_\mu} \text{pd}(\bar{\sigma}_\mu \mid t) \, d\bar{\sigma}_\mu \quad (30) $$
[S1]

Comparing Equation (30) to the definition of the characteristic function $\varphi_X(r) = \int e^{irx} \text{pd}(x) dx$, we see that setting $r = ti$ yields:
$$ \varphi_{\bar{\sigma}_\mu(t)}(ti) = \int e^{i(ti)\bar{\sigma}_\mu} \text{pd}(\bar{\sigma}_\mu \mid t) \, d\bar{\sigma}_\mu = \int e^{-t\bar{\sigma}_\mu} \text{pd}(\bar{\sigma}_\mu \mid t) \, d\bar{\sigma}_\mu $$
Thus:
$$ \text{ff}(t) = \varphi_{\bar{\sigma}_\mu(t)}(ti) \quad (32) $$
[S1]

**Edge Cases and Pitfalls:**
- **Homogeneity:** If the medium has no density fluctuations ($\sigma_\mu(t)$ is constant), the equality $\text{ff}(t) = e^{-t\langle \bar{\sigma}_\mu(t) \rangle}$ holds. Any deviation results in Jensen's inequality: $\langle e^{-t\bar{\sigma}_\mu} \rangle \geq e^{-t\langle \bar{\sigma}_\mu \rangle}$, leading to longer-than-exponential transmittance tails [S1].
- **Parameter Dependence:** The parameters of the distribution $\text{pd}(\bar{\sigma}_\mu \mid t)$ may vary with distance $t$, requiring careful tracking of variance decay as a function of $t$ [S1].
- **Precision:** Evaluating the CF at imaginary arguments requires the CF to be well-defined for complex inputs. Numerical stability depends on the specific closed-form CF used (see Subtopics) [S1].

## Subtopics

### [Gamma-distributed 1/f^β noise transmittance model](14552-gamma-distributed-1-f-noise-transmittance-model.md)
Uses a Gamma distribution for the line-averaged density to ensure strictly non-negative extinction values, addressing artifacts from Gaussian assumptions.

### [Gaussian 1/f^β noise transmittance model](14555-gaussian-1-f-noise-transmittance-model.md)
Assumes the line-averaged density follows a Gaussian distribution derived from $1/f^\beta$ fractal noise, providing a closed-form solution via the Gaussian CF.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample if a closed-form Characteristic Function is available for the chosen distribution. This avoids the need for tabulation or Monte Carlo integration over random realizations [S1].
- **Memory Complexity:** $O(1)$, requiring only the storage of the current distance $t$ and parameters $\psi$.
- **Strengths:** Avoids the complexity of instantiating billions of discrete particles. Provides analytical closed-form solutions for transmittance in correlated media [S1].
- **Limitations:** Parametric control can be cumbersome if the specific PDF's CF is not readily available or if the distribution parameters depend complexly on $t$. The method assumes the line-averaged density can be well-modeled by a specific distribution (e.g., Gaussian or Gamma) [S1].
- **When to Use:** When rendering non-exponential media with spatial correlations (fractal noise) and a closed-form CF for the line-averaged density is known.
- **When Not to Use:** When the density distribution is unknown or does not have a tractable characteristic function.

## Background
This method was introduced by Jensen et al. in "A radiative transfer framework for non-exponential media" (2019) [S1]. It builds on the observation that ensemble averaging over random media realizations can be efficiently computed using probabilistic tools. The approach follows Davis and Mineev-Weinstein [2011] in modeling micro-scale density variability using $1/f^\beta$ fractal noise [S1]. The key insight is the equivalence between the ensemble-averaged transmittance integral and the characteristic function evaluated at an imaginary argument, enabling efficient computation without explicit particle instantiation [S1].

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
