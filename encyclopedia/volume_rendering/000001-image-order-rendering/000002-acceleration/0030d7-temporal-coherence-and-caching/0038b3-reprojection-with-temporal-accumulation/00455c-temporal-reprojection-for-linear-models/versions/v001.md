# Temporal Reprojection for Linear Models

Also called: Linear Model Reprojection, wRLS with Temporal Features

## Overview
Temporal Reprojection for Linear Models exploits temporal coherence by reprojecting linear model coefficients and features rather than raw pixel colors, allowing the model to react immediately to abrupt changes in camera, light, or transfer functions [S1]. Unlike standard temporal reprojection which caches constant values per pixel, this method uses linear regression to predict gradual changes while maintaining robustness against sudden shading variations, thereby avoiding the need for Monte Carlo (MC) integration resets [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current frame's noisy MC estimates $\tilde{I}_j(t)$, camera view matrix, per-pixel world coordinates, and the history buffer from the previous frame.
*   **Outputs:** Denoised color values derived from the linear model, updated feature vectors $z_j(t)$, and linear model coefficients.

**Data Structures:**
*   **History Buffer:** Stores the feature vector $z_j \in \mathbb{R}^3$ for each pixel $j$. The vector corresponds to pixel color (RGB) and is constructed using all three channels simultaneously to avoid color-shift effects, even though denoising is performed per channel [S1].
*   **Linear Model:** Maintains coefficients for linear regression at each pixel. The specific coefficient update logic is handled by Weighted Recursive Least Squares (wRLS), which allocates high weights to samples with lower noise variance [S1].

**Steps:**
1.  **Estimate Velocities:** Calculate per-pixel velocity vectors $\upsilon_j = (\upsilon_x, \upsilon_y)$ using the camera view matrix and per-pixel world coordinates. This effectively computes optical flow for the linear model [S1].
2.  **Reproject Coordinates:** Define the reprojection operation $\pi(\upsilon_j)$ to map the current pixel index $j$ to the corresponding coordinate $q$ in the previous frame:
    $$ q \leftarrow \pi(\upsilon_j) = j + \upsilon_j $$
    If $\upsilon_j = 0$ (e.g., static camera), $q = j$ [S1].
3.  **Retrieve History:** Access the history feature $z_q(t-1)$ from the reprojected coordinate $q$ in the previous frame's history buffer [S1].
4.  **Rectify History (Clamping):** Apply the neighborhood clamping operator $\Psi[\cdot]$ to the reprojected history $z_q(t-1)$ using the current noisy estimate $\tilde{I}_j(t)$. This rectification reduces artifacts caused by stale history data when the scene content has changed [S1].
5.  **Update Feature Vector:** Construct the new feature vector $z_j(t)$ by combining the rectified history and the current estimate. The update uses an exponentially weighted history buffer:
    $$ z_j(t) = \alpha \Psi[z_q(t-1), \tilde{I}_j(t)] + (1 - \alpha)\tilde{I}_j(t) $$
    where $\alpha$ is a weight controlling the balance between history and the current estimate [S1].
6.  **Linear Regression:** Use the updated feature vector $z_j(t)$ as input to the linear model (specifically Weighted RLS) to predict the denoised output. The wRLS variant handles temporally varying noise inherent in DVR images generated with small sample counts [S1].

**Parameters:**
*   $\alpha$: Exponential decay weight for the history buffer. Typical values are not explicitly given in the sources, but it controls the trade-off between temporal stability and responsiveness [S1].

**Edge Cases and Pitfalls:**
*   **Abrupt Changes:** Standard reprojection of pixel colors fails to adapt to changes in lighting or transfer functions. Linear model reprojection reacts immediately to these changes, avoiding MC integration resets [S1].
*   **Color Shifts:** Denoising is performed per channel, but the feature $z_j(t)$ must be constructed using all three color channels simultaneously to avoid undesired color-shift effects [S1].
*   **Stale History:** Without the clamping operator $\Psi[\cdot]$, reprojected history can introduce artifacts if the underlying scene has changed significantly between frames [S1].

## Subtopics

### Temporal Feature History Buffering
A feature vector $z_j(t)$ constructed from an exponentially weighted history buffer of MC estimates, used as input to the linear model to provide temporal stability and reduce noise variance [S1]. It updates via $z_j(t) = \alpha\Psi[z_q(t-1), \tilde{I}_j(t)] + (1-\alpha)\tilde{I}_j(t)$, where $\Psi[\cdot]$ is a neighborhood clamping operator that relies on the current-frame input estimate $\tilde{I}_j(t)$ to reduce artifacts from stale history data [S1]. It is chosen when temporal stability is required without sacrificing responsiveness to lighting changes. It differs from simple color accumulation by integrating clamping and multi-channel construction to prevent color shifts [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method involves per-pixel velocity estimation, reprojection, clamping, and weighted RLS updates.
*   **Memory Complexity:** Requires storing the history buffer $z_j \in \mathbb{R}^3$ per pixel, in addition to the linear model coefficients [S1].
*   **Strengths:** Reacts immediately to abrupt changes in shading, lighting, or transfer functions, avoiding MC integration resets. More robust than caching constant values per pixel [S1].
*   **Limitations:** Requires estimation of per-pixel velocities (optical flow) and additional storage for history buffers [S1].
*   **Performance:** Demonstrated on real-time VPT. Specific hardware or resolution metrics are not given in the sources [S1].

## Background
This method was introduced in the context of real-time volumetric path tracing (VPT) for direct volume rendering. It addresses the limitations of conventional temporal reprojection, which reprojects pixel colors and struggles with dynamic scene changes [S1]. The approach builds on exponentially weighted history buffers and weighted recursive least squares (wRLS) to handle heterogeneous noise in DVR images [S1].

## Sources
[S1] Real-Time Denoising of Volumetric Path Tracing for Direct Volume Rendering, https://doi.org/10.1109/tvcg.2020.3037680, fetched 2026-10-26
