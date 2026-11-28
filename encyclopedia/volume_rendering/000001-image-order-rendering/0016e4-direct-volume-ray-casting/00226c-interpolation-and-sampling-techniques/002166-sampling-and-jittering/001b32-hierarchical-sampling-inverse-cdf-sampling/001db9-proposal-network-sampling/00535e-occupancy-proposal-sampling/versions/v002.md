# Occupancy Proposal Sampling

Also called: Occupancy Proposal Network Sampling, Proposal Network Sampler

## Overview
Occupancy Proposal Sampling is a ray sampling strategy that queries a lightweight occupancy proposal network at uniform intervals along each camera ray, then selects additional samples based on the initial samples to accelerate rendering [S2]. It differs from its parent, Proposal Network Sampling, by specifically utilizing an occupancy-based signal and employing separate hash table-backed static and dynamic branches for temporal scenes [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray $r$, current time $t$ (for dynamic scenes), and the lightweight proposal network parameters.
*   **Output:** A set of sample positions $\{t_i\}$ and associated weights $\{w_i\}$ along the ray $r$ for volume rendering integration [S2].

**Data Structures:**
The proposal network is modeled with separate hash table-backed static and dynamic branches [S2].
*   **Static Branch:** Stores geometry that does not change over time [S2].
*   **Dynamic Branch:** Stores geometry that changes over time [S2].
*   **Hash Table:** Used for efficient spatial hashing of features in both branches [S2].

**Steps:**
1.  **Uniform Query:** Query the lightweight occupancy proposal network at uniform intervals along the camera ray $r$ within the viewing range (typically $t_0 \in [-1, 1]$) [S2].
2.  **Sample Selection:** Pick additional samples based on the initial samples, guided by occupancy values to concentrate samples in the area where the first surface intersects [S1, S2].
3.  **Branch Separation:** Maintain separate static and dynamic branches, each backed by a hash table [S2].
4.  **Histogram Loss Training:** Train each branch using histogram loss with the weights of the respective branch of the main model as the target distribution [S2]. The loss is computed as $L_f(r) = \sum_{t_0 \in [-1,1]} |F(r) - \hat{F}_{t_0}(r)|$, where $F(r)$ is the cumulative opacity from the main model and $\hat{F}_{t_0}(r)$ is the cumulative opacity from the proposal network [S2].
5.  **Distortion Loss Regularization:** Regularize the resulting sample distances and weights using distortion loss [S2].

**Parameters:**
*   **Speedup Factor:** 2-4x compared to uniform sampling [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** Hash table collisions may affect accuracy in high-density regions [G].
*   **Boundaries:** The uniform query range must cover the entire visible scene; clipping errors can occur if the frustum is not correctly defined [G].
*   **Degenerate Input:** If the scene is empty or fully transparent, the proposal network may return uniform occupancy, degrading to uniform sampling [G].

## Subtopics

### Learned Sampler for Neural Volume Rendering
A generalizable learned sampler conditioned on cheap low-resolution information to enable full-resolution rendering during training, replacing dense uniform sampling [S3]. The sampler is a scene-conditional proposal network that generalizes across scenes, inspired by density estimators, instead of being category-specific or optimized per-scene [S3]. It operates on a per-ray basis to adaptively accelerate rendering [S3]. The sampler allows the method to operate with just 20 samples per ray, compared to existing 3D GANs which must use at least 96 samples per ray [S3]. The approach uses up to 5 times fewer depth samples than dense sampling methods [S3]. Unlike acceleration structures like octrees which are limited to coarse or category-specific acceleration, this sampling-based method generalizes to every new scene [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of uniform query points plus the number of additional samples [G].
*   **Memory Complexity:** Proportional to the size of the hash tables for the static and dynamic branches [S2].
*   **Measured Performance:** Provides a 2-4x speedup compared to uniform sampling [S2].
*   **Scaling:** Scales well with dynamic scenes due to the separation of static and dynamic branches [S2].
*   **Strengths:** Significant speedup; handles dynamic scenes effectively via branch separation [S2].
*   **Limitations:** Requires training the proposal network; may introduce artifacts if the proposal network is poorly trained [G].
*   **When to Use:** When rendering speed is critical and the scene has significant empty space or occluders [S2].
*   **When Not to Use:** When the scene is uniformly dense, as the overhead of the proposal network may not be justified [G].

## Background
Originated in the context of scalable urban dynamic scenes, specifically in the SUDS method [S2]. It builds on the idea of using lightweight networks to guide sampling, extending it to dynamic scenes with separate static and dynamic branches [S2].

## Sources
[S1] Neural Radiance Fields-Based 3D Reconstruction of Power Transmission Lines Using Progressive Motion Sequence Images., https://doi.org/10.3390/s23239537, fetched 2026-10-26
[S2] SUDS: Scalable Urban Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.01191, fetched 2026-10-26
[S3] What You See is What You GAN: Rendering Every Pixel for High-Fidelity Geometry in 3D GANs, https://doi.org/10.1109/cvpr52733.2024.02148, fetched 2026-10-26
