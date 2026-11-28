# Differentiable Regularization Framework for Participating Media

Also called: Differentiable Regularization for Participating Media, Differentiable Regularization-based Approach, Radiance Prediction via Gradient

## Overview
This method reduces variance in high-order scattering dominant participating media by rendering a low-albedo version of the scene, computing the gradient of radiance with respect to albedo, and predicting the high-albedo result using a novel prediction function [S1]. It is designed for scenarios with low time budgets where standard path tracing is too noisy [S1]. Unlike path space regularization methods which cannot correct regularization error, this method extrapolates from the regularized result to predict the original configuration using gradient information [S1]. It builds on [Differentiable volume rendering](43-differentiable-volume-rendering.md) by utilizing differentiable path tracing to compute gradients with respect to material parameters.

## How it works
The method operates in three main steps: differentiable volumetric path tracing on a modified low-albedo configuration, temporal denoising of the computed gradient, and prediction of the final high-albedo radiance.

### Inputs and Outputs
**Inputs:**
*   Target albedo $\alpha_t$ and extinction coefficient $\sigma_t$ for the participating media.
*   Phase function $p(\omega, \omega')$ (typically Henyey–Greenstein) [S1].
*   Camera parameters and scene geometry.
*   Source albedo $\alpha_s$, chosen as $0.9 - 0.95 \times \alpha_t$ [S1].
*   A sequence of frames (for temporal denoising) [S1].

**Outputs:**
*   Predicted radiance image $\hat{L}_{target}$ for the target albedo $\alpha_t$ [S1].

### Data Structures
The method relies on standard volumetric path tracing data structures but extends them to store gradients.
*   **Radiance Image:** A 2D array of color values $L(x, \omega)$ [S1].
*   **Gradient Image:** A 2D array of values $\frac{\partial L}{\partial \alpha}$ representing the derivative of radiance with respect to the albedo parameter [S1].
*   **Path Contribution:** For a path $\tilde{p}$ with $K$ scattering events, the contribution is defined as:
    $$
    f(\tilde{p}) = \prod_{k=1}^K w_k g(\omega_{k-1}, \omega_k)
    $$
    where $w_k$ is the weight of the path segment $x_{k-1} \leftrightarrow x_k$:
    $$
    w_i = \begin{cases} \alpha \sigma_t \text{Tr}(x_{i-1} \leftrightarrow x_i) & \text{if } x_i \in \text{medium} \\ \text{Tr}(x_{i-1} \leftrightarrow x_i) & \text{if } x_i \in \text{surface} \end{cases}
    $$
    and $g(\omega_{k-1}, \omega_k)$ is the phase function evaluation [S1].

### Steps

1.  **Differentiable Volumetric Path Tracing:**
    Render the scene using a modified media configuration with source albedo $\alpha_s < \alpha_t$. Compute the radiance $L_s$ and the gradient $\frac{\partial L_s}{\partial \alpha}$ with respect to the albedo. This is performed using automatic differentiation tools (e.g., Eigen library) integrated into the renderer (e.g., Mitsuba) [S1]. The radiative transfer equation is modeled as:
    $$
    L(x, \omega) = \text{Tr}(x \leftrightarrow x_s)L(x_s, \omega) + \int_0^s \text{Tr}(x \leftrightarrow x_t)\sigma_s(x_t)L_i(x_t, \omega)dt
    $$
    where $\text{Tr}(x \leftrightarrow x_s) = \exp(-\sigma_t \|x - x_s\|)$ [S1]. The in-scattering radiance is:
    $$
    L_i(x_t, \omega) = \int_{\Omega_{4\pi}} p(\omega, \omega_t)L(x_t, \omega_t)d\omega_t
    $$
    Monte Carlo estimation is used:
    $$
    L(x, \omega) = \frac{1}{M} \sum_{i=1}^M \frac{f(\tilde{p}_i)}{\text{pdf}(\tilde{p}_i)}
    $$
    [S1].

2.  **Temporal Denoising:**
    Apply temporal denoising to the gradient image $\frac{\partial L_s}{\partial \alpha}$ using information from neighboring frames to produce a noise-free gradient estimate [S1]. The specific denoising algorithm is not detailed in the source, but it is noted that the cost is negligible compared to rendering [S1].

3.  **Radiance Prediction:**
    Predict the final radiance $\hat{L}_{target}$ for the target albedo $\alpha_t$ using the rendered low-albedo radiance $L_s$ and the denoised gradient $\frac{\partial L_s}{\partial \alpha}$. The prediction function is novel and designed to fit the behavior of media rendering, differing from linear or log-linear functions used in prior work [S1]. The exact formula for the prediction function is not provided in the source text, but it utilizes the gradient to extrapolate from $\alpha_s$ to $\alpha_t$ [S1].

### Parameters
*   **Source Albedo ($\alpha_s$):** The albedo used for the differentiable path tracing step. The optimal value is $0.9 - 0.95 \times \alpha_t$ [S1]. Using a source albedo far from the target (e.g., $0.5 \times \alpha_t$) results in large bias [S1].
*   **Phase Function:** Henyey–Greenstein is used in all test scenes [S1].
*   **Extinction Coefficient ($\sigma_t$):** The method is robust across a range of $\sigma_t$ values (tested from 5 to 50) [S1].

### Edge Cases and Pitfalls
*   **Bias-Variance Trade-off:** The method introduces bias as a trade-off for variance reduction. The error increases compared to path tracing when the sample rate is high [S1].
*   **Low-Albedo Failure:** The method does not benefit low-albedo media. A failure case is shown for $\alpha_t = 0.5$ [S1].
*   **High Source Albedo:** Increasing the source albedo decreases bias but increases variance [S1].

## Operating characteristics
*   **Time Complexity:** The method is significantly faster than reference path tracing for equivalent quality. In tested scenes, it achieves lower MSE in less time than equal-time path tracing [S1]. The cost of prediction and denoising is negligible (e.g., 0.2s for a 512x512 image) compared to rendering time [S1].
*   **Memory Complexity:** Not explicitly stated, but relies on storing radiance and gradient images per frame for temporal denoising [S1].
*   **Performance:**
    *   **Bunny (500x500):** MSE $1.1 \times 10^{-4}$ in 0.49 min vs. PT MSE $2.3 \times 10^{-4}$ in 0.50 min. Reference MSE $1.1 \times 10^{-4}$ in 45.61 min [S1].
    *   **Candle (256x256):** MSE $6.1 \times 10^{-5}$ in 1.21 min vs. PT MSE $1.3 \times 10^{-4}$ in 1.17 min. Reference MSE $6.1 \times 10^{-5}$ in 15.89 min [S1].
    *   **Lucy (512x512):** MSE $1.5 \times 10^{-5}$ in 1.14 min vs. PT MSE $4.4 \times 10^{-5}$ in 1.16 min. Reference MSE $1.5 \times 10^{-5}$ in 60.00 min [S1].
*   **Strengths:** Reduces noise in high-order scattering dominant media; robust across varying extinction coefficients; suitable for low time budgets [S1].
*   **Limitations:** Introduces bias; ineffective for low-albedo media; less accurate than high-sample path tracing when time is not constrained [S1].
*   **When to use:** High-albedo participating media with strict time constraints [S1].
*   **When not to use:** Low-albedo media or when high sample counts are available [S1].

## Background
The method was proposed by authors publishing in 2023 [S1]. It builds on the observation that lower-albedo media have less noise than higher-albedo media in Monte Carlo path tracing [S1]. It differs from previous differentiable regularization approaches by using a novel prediction function and temporal denoising of the gradient [S1]. It is implemented inside the Mitsuba renderer using automatic differentiation tools from the Eigen library [S1].

## Sources
[S1] Efficient Participating Media Rendering with Differentiable Regularization, https://iccvm.org/2023/papers/s8-3-284-CVMJ.pdf, fetched 2026-10-29
