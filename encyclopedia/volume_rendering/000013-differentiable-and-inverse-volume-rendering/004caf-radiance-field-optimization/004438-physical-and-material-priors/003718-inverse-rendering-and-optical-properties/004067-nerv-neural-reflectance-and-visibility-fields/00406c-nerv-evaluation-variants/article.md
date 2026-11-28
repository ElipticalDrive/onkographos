# NeRV Evaluation Variants

Also called: NeRV NVF, NeRV Trace

## Overview
NeRV Evaluation Variants are inference-time strategies for the NeRV representation that modify how visibility and material properties are queried during rendering [S1, S2]. NeRV with Neural Visibility Fields (NVF) uses the same visibility approximations learned during training at test time, while NeRV with Test-time Tracing (Trace) replaces the learned visibility with brute-force ray tracing to estimate visibility to point light sources for sharper shadows [S1]. Both variants reuse the same trained model [S1]. A third approach defers ray queries to diffuse intersections to enable rendering for non-diffuse materials, accepting sampling-related noise as a trade-off [S2].

## How it works
**Inputs:**
*   Trained NeRV model parameters $\theta$ [G].
*   Camera pose $(C, R)$ and light source positions $L_i$ with intensities $I_i$ [G].
*   Scene volume bounds [G].

**Outputs:**
*   Rendered image pixels [G].

**Data Structures:**
*   **NeRV Model:** A neural network $f_\theta(x)$ taking a 3D coordinate $x \in \mathbb{R}^3$ and returning a visibility value $v \in [0, 1]$ and a reflectance color $c \in \mathbb{R}^3$ [G].
*   **Ray:** Defined by origin $O$ and direction $D$ [G].
*   Exact field layouts and bit packing are Not given in the sources. [G]

**Numbered Steps & Formulas:**
For each pixel, cast a ray through the scene volume. [G] Sample points along the ray. Accumulate color using standard volume rendering integration [G]. The specific variant determines how the visibility term $v$ is computed for each sample point. [G]

### NeRV NVF (Neural Visibility Fields)
This is the default evaluation mode, consistent with the training procedure [S1].
1. For each sample point $x$ along the ray:
2. Query the NeRV model: $(v, c) = f_\theta(x)$ [G].
3. Use $v$ as the occlusion factor for the light contribution at $x$ [S1].
4. Accumulate color using volume rendering integration [G].

**Characteristics:**
*   Uses the same visibility approximations used during training [S1].

### NeRV Trace (Test-time Tracing)
This variant replaces the learned visibility MLP query with brute-force ray tracing to estimate visibility to point light sources [S1].
1. For each sample point $x$ along the view ray:
2. For each light source $L_i$:
    a. Construct a shadow ray from $x$ to $L_i$ [G].
    b. Perform brute-force ray marching along the shadow ray to check for intersections with the scene geometry or density field [S1].
    c. Compute visibility based on whether the shadow ray is occluded [S1].
3. Use the traced visibility for the lighting calculation at $x$ [S1].
4. Accumulate color [G].

**Characteristics:**
*   Uses brute-force tracing to estimate visibility to point light sources [S1].
*   Renders sharper shadows at test time [S1].

### Deferred Diffuse Queries for Glossy Materials
This variant enables non-diffuse material rendering by deferring ray queries [S2].
1. For each sample point $x$ along the view ray:
2. If the material at $x$ is non-diffuse:
    a. Trace a ray from $x$ to find the nearest diffuse intersection $x_{\text{diffuse}}$ [S2].
    b. Query the NeRV model at $x_{\text{diffuse}}$ to get reflectance [S2].
    c. Apply a glossy BRDF approximation using the deferred color [S2].
3. If the material is diffuse, query directly at $x$ [S2].
4. Accumulate color [G].

**Characteristics:**
*   Enables NIV rendering for non-diffuse materials by deferring ray queries to diffuse intersections [S2].
*   Accepts sampling-related noise as a trade-off [S2].
*   Maintaining noise-free rendering requires exploring alternative solutions [S2].

**Pitfalls:**
*   The deferred query approach introduces sampling-related noise [S2].
*   Future work suggests incorporating surface roughness as a control parameter in the neural model to natively handle glossy materials [S2].

## Subtopics

### NeRV NVF
A test-time evaluation strategy that uses the trained neural visibility field for occlusion [S1]. It is chosen when rendering speed is prioritized over shadow sharpness [G]. It reuses the visibility approximations from training [S1].

### NeRV Trace
A test-time evaluation strategy that replaces the neural visibility query with brute-force ray tracing for shadow rays [S1]. It is chosen when sharp, hard shadows are required [S1]. It estimates visibility to point light sources directly [S1].

### Deferred Diffuse Queries
A material handling strategy for non-diffuse surfaces [S2]. It is chosen when rendering glossy or specular materials with a model trained primarily on diffuse data [G]. It trades rendering noise for material flexibility [S2].

## Operating characteristics
*   **NeRV NVF:**
    *   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples per ray. The visibility query is a neural network forward pass [G].
    *   **Memory Complexity:** Dominated by the storage of the NeRV model weights [G].
    *   **Performance:** Fast inference. The base model training requires 1 million iterations over 128 TPU cores, taking 1 day to converge [S1].
    *   **Strengths:** Efficient, consistent with training [S1].
    *   **Limitations:** Visibility is approximated by the MLP [S1].
*   **NeRV Trace:**
    *   **Time Complexity:** Increases with the number of light sources and shadow ray samples due to brute-force marching [S1].
    *   **Strengths:** Renders sharper shadows at test time [S1].
    *   **Limitations:** Higher computational cost per sample due to ray marching [S1].
*   **Deferred Diffuse Queries:**
    *   **Time Complexity:** Increases due to additional ray tracing for deferred queries [S2].
    *   **Strengths:** Supports non-diffuse materials [S2].
    *   **Limitations:** Sampling-related noise [S2].

## Background
NeRV was introduced by [S1] (NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, CVPR 2021). The method produces a 3D representation that can be rendered from novel viewpoints under arbitrary lighting conditions, including indirect illumination effects [G]. The evaluation variants were developed to address limitations in shadow quality and material support [G]. The deferred diffuse query approach is discussed in the context of extending Neural Irradiance Volumes (NIV) to non-diffuse materials [S2].

## Sources
[S1] NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, https://doi.org/10.1109/cvpr46437.2021.00741, fetched 2026-10-26
[S2] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
