# Neural Network RTE Simplification via Albedo Series

Also called: Albedo power series for neural prediction, Radiation Transfer Equation Simplification for Neural Network, Scattering term approximation

## Overview
This method simplifies the classical Radiative Transfer Equation (RTE) by expressing the co-scattering term as an infinite power series of the single-scattering albedo $\eta$, enabling efficient prediction via a neural network [S1, unofficial]. It solves the computational bottleneck of evaluating multiple scattering events in participating media by decoupling the complex integral operator from the albedo parameter, allowing the network to infer the total scattering contribution from pre-calculated geometric and density features [S1, unofficial]. It sets itself apart from [Full Propagator Approximation for Multiply-Scattered Radiance](1298-full-propagator-approximation-for-multiply-scattered-radianc.md) by explicitly leveraging the algebraic structure of the albedo series, and from [Monte Carlo Method for Multiple Scattering](1940-monte-carlo-method-for-multiple-scattering.md) by providing a deterministic, differentiable approximation rather than a stochastic estimate.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Position $p$, direction $\omega$, density features, transmittance features, and phase function features. These are pre-calculated and stored in a lookup table based on particle coordinates and sampling templates [S1, unofficial].
*   **Outputs:** The backbone neural network outputs the approximation of the co-scattering term $F(p, \omega)$ or the final radiance $L_o(p, \omega)$ [S1, unofficial].

**Data Structures:**
*   **Pre-calculation Table:** Stores sampling features (density, transmittance, phase) for discrete template points. Access is direct via particle position coordinates [S1, unofficial].
*   **Neural Network Backbone:** Three identical network branches are used, corresponding to density, transmittance, and phase features, though the specific input/output mappings for each branch are not fully detailed in the source [S1, unofficial].

**Mathematical Formulation:**
The classical RTE is given by [S1, unofficial]:
$$ \frac{\partial L_o(p, \omega)}{\partial \omega} = L_e(p, \omega) - \mu_a(p)L_i(p, \omega) - \mu_s(p)L_i(p, \omega) + \frac{\mu_s(p)}{\Omega} \int_{\Omega} f_p(\omega_i \to \omega)L_i(p, \omega_i)d\omega_i $$
where $L_o$ is output radiance, $L_i$ is incident radiance, $L_e$ is self-illumination, $\mu_a$ is absorption coefficient, $\mu_s$ is scattering coefficient, $f_p$ is the phase function, and $\Omega$ is the unit ball [S1, unofficial].

The solution for radiance along a path from $p$ to $z$ is [S1, unofficial]:
$$ L_o(p, \omega) = T(p, z)L_i(z, \omega) + \int_p^z T(p, x)\mu_s(x)F(x, \omega)dx $$
where $T(p, y) = e^{-\int_p^y \mu_t(x)dx}$ is the transmittance, $\mu_t = \mu_a + \mu_s$ is the extinction coefficient, and $F(p, \omega)$ is the co-scattering term [S1, unofficial].

The co-scattering term $F(p, \omega)$ is expanded as an infinite series of albedo powers [S1, unofficial]:
$$ F(p, \omega) = \sum_{i=0}^{\infty} \eta^i F_i(p, \omega) $$
where $\eta = \mu_s / \mu_t$ is the single-scattering albedo [S1, unofficial]. The terms are defined recursively [S1, unofficial]:
$$ F_0(p, \omega) = f_p \cdot T \cdot I $$
$$ F_{i+1}(p, \omega) = \Pi F_i(p, \omega) $$
with the operator $\Pi$ defined as [S1, unofficial]:
$$ \Pi F(p, \omega) = \int_{\Omega} \int_p^z f_p T \mu_t F(p, \omega) dx d\omega_i $$
For co-directional scattering, $f_p$ is constant with respect to $\omega_i$, simplifying the operator [S1, unofficial].

**Algorithm Steps:**
1.  **Pre-calculation:** Compute sampling features (density, transmittance, phase) for all relevant template points and store them in a pre-calculation table [S1, unofficial].
2.  **Feature Extraction:** For a query point $p$ and direction $\omega$, retrieve the corresponding sampling features from the table [S1, unofficial].
3.  **Neural Prediction:** Input the features into the backbone neural network. The network is trained to predict the value of the infinite series sum $F(p, \omega)$ or the final radiance $L_o(p, \omega)$ directly, bypassing the iterative calculation of individual $F_i$ terms [S1, unofficial].
4.  **Rendering:** Use the predicted radiance for image synthesis [S1, unofficial].

**Parameters:**
*   $\eta = \mu_s / \mu_t$: Single-scattering albedo [S1, unofficial].
*   $l = 1/\mu_t$: Mean free path [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The source assumes the volume medium does not emit light ($L_e = 0$) for the simplified transmission equation [S1, unofficial].
*   The source mentions the medium albedo is assumed constant for the simplified equation, but the general series formulation allows for spatial variation in $\mu_t$ and $\mu_s$ within the integral [S1, unofficial].
*   Precision issues may arise if the series converges slowly for high albedo values, though the neural network is intended to mitigate this by learning the global mapping [S1, unofficial].

## Subtopics

### [Participating Media Regularization (Prediction Function)](15081-participating-media-regularization-prediction-function.md)
Estimates target radiance $R_t$ given source radiance $R_s$, gradient $G_s$, and albedos $\alpha_s, \alpha_t$ using a two-exponential formulation to handle non-linear radiance behavior.

## Operating characteristics
*   **Time Complexity:** Depends on the neural network inference time and table lookup, which is significantly faster than iterative multiple scattering solvers [S1, unofficial].
*   **Memory Complexity:** Requires storage for the pre-calculation table and the neural network weights [S1, unofficial].
*   **Strengths:** Real-time rendering capability for high albedo anisotropic materials; avoids complex iterative calculations [S1, unofficial].
*   **Limitations:** Requires pre-calculation and training; accuracy depends on the neural network's ability to generalize across different media properties [S1, unofficial].
*   **When to use:** Real-time rendering of participating media with high albedo and anisotropic scattering [S1, unofficial].
*   **When not to use:** Scenes with significant self-illumination or where exact physical accuracy is required over performance [S1, unofficial].

## Background
This method originates from research on real-time rendering of high albedo anisotropic materials with multiple scattering [S1, unofficial]. It addresses the difficulty of simulating multiple scattering events efficiently by leveraging the mathematical properties of the albedo series and modern neural network approximation capabilities [S1, unofficial].

## Sources
[S1] A real-time rendering method for high albedo anisotropic materials with multiple scattering, https://doi.org/10.48550/arxiv.2401.14051, fetched 2026-10-26 (unofficial)
