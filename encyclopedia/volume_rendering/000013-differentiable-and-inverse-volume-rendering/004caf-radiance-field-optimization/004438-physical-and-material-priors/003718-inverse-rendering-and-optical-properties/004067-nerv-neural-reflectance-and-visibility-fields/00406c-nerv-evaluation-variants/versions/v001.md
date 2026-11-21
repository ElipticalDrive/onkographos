# NeRV Evaluation Variants

Also called: NeRV NVF, NeRV Trace

## Overview
NeRV Evaluation Variants are inference-time strategies for the NeRV (Neural Reflectance and Visibility Fields) representation that trade rendering speed for shadow fidelity or material generality. They address the limitation of the base NeRV method, which uses a learned visibility MLP for efficiency but produces soft shadows and assumes diffuse surfaces. The variants allow users to choose between sharp shadows via brute-force tracing or extended material support via deferred queries, while reusing the same trained model.

## How it works
The NeRV model consists of a neural network that outputs reflectance and visibility properties for any 3D point. The evaluation variants modify the rendering loop (ray marching) at test time.

**Inputs:**
*   Trained NeRV model parameters $\theta$.
*   Camera pose $(C, R)$.
*   Light source positions $L_i$ and intensities $I_i$.
*   Scene volume bounds.

**Outputs:**
*   Rendered image pixels.

**Data Structures:**
*   **NeRV Model:** A neural network $f_\theta(x)$ taking a 3D coordinate $x \in \mathbb{R}^3$ and returning a visibility value $v \in [0, 1]$ and a reflectance color $c \in \mathbb{R}^3$.
*   **Ray:** Defined by origin $O$ and direction $D$.

**General Rendering Loop:**
For each pixel, cast a ray. Sample points along the ray. Accumulate color using volume rendering integration. The specific variant determines how the visibility term $v$ is computed for each sample point.

### NeRV NVF (Neural Visibility Fields)
This is the default evaluation mode, consistent with the training procedure.

1.  For each sample point $x$ along the ray:
2.  Query the NeRV model: $(v, c) = f_\theta(x)$.
3.  Use $v$ as the occlusion factor for the light contribution at $x$.
4.  Accumulate color.

**Characteristics:**
*   Uses the same visibility approximations used during training [S1].
*   Fast, but visibility is smoothed, resulting in soft shadows [S1].

### NeRV Trace (Test-time Tracing)
This variant replaces the learned visibility MLP query with brute-force ray tracing to estimate visibility to point light sources.

1.  For each sample point $x$ along the view ray:
2.  For each light source $L_i$:
    a.  Construct a shadow ray from $x$ to $L_i$.
    b.  Perform brute-force ray marching along the shadow ray to check for intersections with the scene geometry (or density field).
    c.  Compute binary visibility $v_{trace} \in \{0, 1\}$ based on whether the shadow ray is occluded.
3.  Use $v_{trace}$ instead of the MLP output $v$ for the lighting calculation at $x$.
4.  Accumulate color.

**Characteristics:**
*   Uses brute-force tracing to estimate visibility to point light sources [S1].
*   Renders sharper shadows at test time [S1].
*   Significantly slower than NVF due to additional ray marching per sample [S1].

### Deferred Diffuse Queries for Glossy Materials
This variant enables non-diffuse material rendering by deferring ray queries.

1.  For each sample point $x$ along the view ray:
2.  If the material at $x$ is non-diffuse:
    a.  Instead of querying reflectance directly at $x$, trace a ray from $x$ to find the nearest diffuse intersection $x_{diffuse}$.
    b.  Query the NeRV model at $x_{diffuse}$ to get reflectance.
    c.  Apply a glossy BRDF approximation using the deferred color.
3.  If the material is diffuse, query directly at $x$.
4.  Accumulate color.

**Characteristics:**
*   Enables NIV rendering for non-diffuse materials by deferring ray queries to diffuse intersections [S2].
*   Accepts sampling-related noise as a trade-off [S2].
*   Maintaining noise-free rendering requires exploring alternative solutions [S2].

**Pitfalls:**
*   The deferred query approach introduces sampling-related noise [S2].
*   Future work suggests incorporating surface roughness as a control parameter in the neural model to natively handle glossy materials [S2].

## Subtopics

### NeRV NVF
A test-time evaluation strategy that uses the trained neural visibility field for occlusion. It is chosen when rendering speed is prioritized over shadow sharpness. It produces soft shadows due to the learned visibility approximation [S1].

### NeRV Trace
A test-time evaluation strategy that replaces the neural visibility query with brute-force ray tracing for shadow rays. It is chosen when sharp, hard shadows are required. It is computationally more expensive than NVF [S1].

### Deferred Diffuse Queries
A material handling strategy for non-diffuse surfaces. It is chosen when rendering glossy or specular materials with a model trained primarily on diffuse data. It trades rendering noise for material flexibility [S2].

## Operating characteristics
*   **NeRV NVF:**
    *   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples per ray. The visibility query is a neural network forward pass.
    *   **Memory Complexity:** Dominated by the storage of the NeRV model weights.
    *   **Performance:** Fast. The authors trained models on 128 TPU cores for 1 million iterations, taking 1 day to converge [S1].
    *   **Strengths:** Efficient, consistent with training.
    *   **Limitations:** Soft shadows, assumes diffuse materials.

*   **NeRV Trace:**
    *   **Time Complexity:** $O(N \cdot M)$ per pixel, where $N$ is view ray samples and $M$ is shadow ray samples. Significantly slower than NVF.
    *   **Strengths:** Sharp shadows.
    *   **Limitations:** High computational cost.

*   **Deferred Diffuse Queries:**
    *   **Time Complexity:** Increases due to additional ray tracing for deferred queries.
    *   **Strengths:** Supports non-diffuse materials.
    *   **Limitations:** Sampling-related noise [S2].

## Background
NeRV was introduced by [S1] (NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, CVPR 2021). The method produces a 3D representation that can be rendered from novel viewpoints under arbitrary lighting conditions, including indirect illumination effects. The evaluation variants were developed to address limitations in shadow quality and material support. The deferred diffuse query approach is discussed in the context of extending Neural Irradiance Volumes (NIV) to non-diffuse materials [S2].

## Sources
[S1] NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, https://doi.org/10.1109/cvpr46437.2021.00741, fetched 2026-10-26
[S2] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
