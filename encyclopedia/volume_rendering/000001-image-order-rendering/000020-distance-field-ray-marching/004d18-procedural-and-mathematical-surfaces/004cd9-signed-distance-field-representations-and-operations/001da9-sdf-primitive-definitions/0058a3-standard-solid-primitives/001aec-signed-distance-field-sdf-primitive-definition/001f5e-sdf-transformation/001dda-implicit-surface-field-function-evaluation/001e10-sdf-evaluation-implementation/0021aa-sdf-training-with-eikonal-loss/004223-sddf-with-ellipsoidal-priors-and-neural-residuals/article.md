# SDDF with Ellipsoidal Priors and Neural Residuals

Also called: Signed Directional Distance Function (SDDF), Scene-level SDDF

## Overview
This method learns a scene-level Signed Directional Distance Function (SDDF) by combining an explicit ellipsoidal prior with an implicit neural residual network [S1]. It solves the problem of reconstructing complex 3D environments from sensor data (LiDAR or RGB-D depth) while maintaining differentiability for novel view synthesis and avoiding the iterative sphere tracing required by standard SDFs [S1]. Unlike its siblings, it explicitly models occlusions via ellipsoids to handle discontinuities, while the neural network recovers fine details, satisfying the directional Eikonal constraint by construction [S1].

## How it works
**Inputs:**
- A set of sensor measurements $\{T_t, Z_t\}_{t=1}^T$, where $T_t \in SE(3)$ is the sensor pose and $Z_t$ contains viewing directions $v_i \in S^2$ and range measurements $r_{t,i} \in \mathbb{R}_{>0}$ [S1].
- Query inputs: a 3D position $p \in \mathbb{R}^3$ and a viewing direction $v \in S^2$ [S1].

**Outputs:**
- The predicted SDDF value $\hat{f}(p, v)$, representing the signed distance to the observed surface along direction $v$ from position $p$ [S1].

**Data Structures:**
1.  **Ellipsoid Prior:** A set of $M$ learnable ellipsoids. Each ellipsoid $j$ is defined by parameters $\theta_j$ (center, orientation, scale). The total number of ellipsoids $M$ is typically 128 (256 for Gibson Allensville) [S1].
2.  **Latent Feature Network ($L$):** A neural network that maps the intersection point and viewing direction to a latent feature vector $z \in \mathbb{R}^m$ [S1].
3.  **Residual Decoder ($R$):** A neural network that takes the latent feature $z$ and outputs the residual prediction $\delta_f(p, v)$ [S1].

**Definitions:**
-   **SDDF Definition:** For a query $(p, v)$, the SDDF $f^*(p, v)$ is the signed distance to the first intersection of the ray $p + dv$ with the occupied space boundary $\partial O$. The sign is positive if $p$ is in free space and negative if $p$ is inside the occupied space [S1]. The function is continuous when entering occupied space from free space, but discontinuous when leaving occupied space (inside-out). This discontinuity is handled by assuming sensors remain in free space, observing the scene from outside-in [S1].
-   **Directional Eikonal Constraint:** The gradient of the SDDF with respect to position $p$ in the direction $v$ must have unit magnitude: $\nabla_p f(p, v) \cdot v = 1$ [S1].

**Algorithm Steps:**

1.  **Initialize Ellipsoids:**
    Initialize the $M$ ellipsoids to approximate the coarse shape of the scene. Do not use K-means++ initialization alone, as it causes excessive ellipsoids for planar surfaces and larger errors [S1]. Use a method that provides a balanced initial coverage [S1].

2.  **Compute Ellipsoid Prior $f(p, v)$:**
    For a query $(p, v)$, compute the distance to the nearest ellipsoid surface along $v$.
    Let $d_j(p, v)$ be the distance from $p$ to the surface of ellipsoid $j$ along direction $v$.
    The prior $f(p, v)$ is the minimum of these distances, considering the sign based on whether $p$ is inside or outside the ellipsoid [S1].
    *Note: The exact closed-form solution for $d_j(p, v)$ involves solving a quartic equation for the intersection of a ray and an ellipsoid. The sources state that gradients are derived in closed-form to accelerate training [S1], but do not provide the explicit quartic solver code. [G] The standard approach is to find the smallest positive root $d$ of the quadratic form defining the ellipsoid surface in the local frame transformed to global coordinates.*

3.  **Find Intersection Point:**
    Identify the ellipsoid $j^*$ that provides the minimum distance $f(p, v)$. Calculate the intersection point $p_{int} = p + f(p, v)v$ [S1].

4.  **Construct Latent Feature $z$:**
    Pass $p_{int}$ and $v$ through the Latent Feature Network $L$ to obtain $z = L(p_{int}, v)$ [S1]. The network is designed such that the resulting residual satisfies the directional Eikonal constraint [S1].

5.  **Predict Residual $\delta_f(p, v)$:**
    Pass $z$ through the Residual Decoder $R$ to obtain the residual: $\delta_f(p, v) = R(z)$ [S1].

6.  **Compute Final SDDF:**
    Combine the prior and the residual:
    $$ \hat{f}(p, v) = f(p, v) + \delta_f(p, v) $$
    [S1]

**Training:**
-   **Loss Function:** Minimize the difference between $\hat{f}(p, v)$ and the ground truth SDDF derived from sensor measurements.
-   **Negative Sample Augmentation:** Include negative SDDF samples (points inside the object) in the training data. Without this, ellipsoids regress into the interior of objects, leading to missing ray intersections and wrong ellipsoid selection [S1].
-   **Embedding:** Use polynomial embedding for the neural networks. Fourier embedding is inferior due to the need for scene-specific spatial frequency tuning [S1].
    -   Fourier embedding formula (for ablation comparison):
        $$ \mathbf{m} = [\sin(2\pi\mathbf{B}_{\text{p}}\mathbf{p}), \cos(2\pi\mathbf{B}_{\text{p}}\mathbf{p}), \sin(2\pi\mathbf{B}_{\text{v}}\mathbf{v}), \cos(2\pi\mathbf{B}_{\text{v}}\mathbf{v})] $$
        where $\mathbf{B}_p, \mathbf{B}_v \in \mathbb{R}^{25 \times 3}$ are random Fourier features sampled from $\mathcal{N}(0, 1)$ [S1].

**Parameters:**
-   Number of ellipsoids $M$: 128 (default), 256 for Gibson Allensville [S1].
-   RGB-D Downsampling: 20× for most datasets, 5× for Replica Hotel, to match LiDAR ray density [S1].
-   Fourier Feature Dimensions: 25 (for ablation) [S1].

**Edge Cases and Pitfalls:**
-   **Discontinuity:** The SDDF is discontinuous when a ray leaves the occupied space from inside out. This is handled by assuming sensors remain in free space [S1].
-   **Too Few Ellipsoids:** Prevents accurate approximation of object shapes [S1].
-   **Too Many Ellipsoids:** Causes the model to learn spurious occlusions between ellipsoids and moves redundant ellipsoids out of the scene during testing [S1].
-   **Missing Negative Samples:** Causes ellipsoids to regress into object interiors, leading to missing ray intersections [S1].
-   **K-means++ Initialization:** Causes larger errors due to excessive ellipsoids for planar surfaces [S1].

## Subtopics

### [Ellipsoid SDDF Prior](10639-ellipsoid-sddf-prior.md)
The explicit geometric representation using learnable ellipsoids to approximate coarse shapes and handle occlusions. It provides an analytic, differentiable prior that satisfies the directional Eikonal equation by construction.

## Operating characteristics
-   **Time Complexity:** Inference is a single forward pass through the ellipsoid prior and the neural network, making it faster than iterative sphere tracing methods. It is 1.5× faster than SDF-Instant-NGP sphere tracing and 4× faster than Nerfacto [S1].
-   **Memory Complexity:** Low GPU memory usage. On Replica-Hotel, max training GPU memory is 3.3 GB, significantly lower than RaDe-GS (18.7 GB) and Nerfacto (4.7 GB) [S1].
-   **Performance:** On Replica-Hotel (Intel 14900K CPU, NVIDIA RTX-3090 GPU), training time is 100 minutes, inference time is 69 ms/frame, and the model has 2.7M parameters [S1].
-   **Strengths:** Single-query differentiable novel view synthesis; handles occlusions explicitly; low memory footprint; robust to varying viewing distances due to Eikonal constraint satisfaction; works with LiDAR and depth-only data [S1].
-   **Limitations:** Slower inference than highly optimized Gaussian Splatting methods like RaDe-GS (69 ms vs 7 ms); requires careful tuning of the number of ellipsoids; sensitive to initialization [S1].
-   **When to Use:** When geometric reconstruction accuracy with low memory usage is required, and differentiable depth rendering is needed without RGB supervision.
-   **When Not to Use:** When maximum inference speed is critical (use RaDe-GS) or when photometric rendering with RGB is the primary goal (use Nerfacto) [S1].

## Background
The method was introduced in "Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals" [S1]. It addresses the limitations of previous SDDF formulations: object-level SDDF ignores objects behind the viewing position, and PDDF lacks sign information and has discontinuities. This method defines a scene-level SDDF that is continuous when entering occupied space and retains sign information, enabling accurate scene reconstruction from sensor data [S1].

## Sources
[S1] Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals., https://doi.org/10.1109/tpami.2026.3688658, fetched 2026-10-26
