# Neural Depth Fields (NeDF)


## Overview
Neural Depth Fields (NeDF) is an implicit surface representation that maps a ray to the distance from its origin to the intersection point with an object's surface, enabling fast depth estimation and spatial occlusion computation for NeRF composition [S1]. It differs from standard NeRF depth regression by predicting a local distance parameter $\mu$ relative to a tangency point rather than the absolute depth, which avoids the impossibility of learning an infinite space and stabilizes training [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $\mathbf{r}$, defined by its origin $\mathbf{o}$ and direction. The ray is represented by a tuple of $N=16$ points sampled along the ray within a relaxed bounding box [S1].
*   **Output:** A tuple $(\mu, \hat{\alpha})$, where $\mu$ is the distance from the tangency point to the intersection, and $\hat{\alpha}$ is a binary indicator of intersection [S1].

**Data Structures:**
*   **Intersection Network:** A Multilayer Perceptron (MLP) denoted as $\mathcal{F}_\Phi$ [S1].
*   **Ray Representation:** A fixed tuple of 16 points sampled along the ray [S1].

**Steps:**
1.  **Ray Sampling:** For a given ray $\mathbf{r}$, sample 16 points along the ray within a specific local range (relaxed bounding box) to approximate the main body of the ray [S1].
2.  **Network Forward Pass:** Feed the tuple of sampled points into the intersection network $\mathcal{F}_\Phi$.
3.  **Prediction:** The network outputs $\mu$ and $\hat{\alpha}$:
    $$ \mathcal{F}_\Phi : \mathbf{r} \mapsto (\mu, \hat{\alpha}) \quad (1) $$
    where $\mu$ is the distance between the ray-object intersection point and the tangency point $\mathbf{p}_\perp$, and $\hat{\alpha}$ indicates if an intersection exists [S1].
4.  **Depth Calculation:** Compute the final depth $D(\mathbf{r})$ using the predicted $\mu$ and the pre-computed tangency point $\mathbf{p}_\perp$:
    $$ D(\mathbf{r}) = |\mathbf{o} - \mathbf{p}_\perp| - \mu \quad (2) $$
    where $\mathbf{o}$ is the ray origin and $\mathbf{p}_\perp$ is the tangency point where the ray is tangent to a sphere centered at the origin in the object's local space [S1].

**Parameters:**
*   Number of sampled points per ray: 16 [S1].

**Edge Cases and Pitfalls:**
*   **Infinite Space:** Directly learning the absolute depth $D(\mathbf{r})$ is impossible because it involves an infinite space [S1]. NeDF learns $\mu$ instead, which is defined in local space and has a limited range of values [S1].
*   **Tangency Point:** The tangency point $\mathbf{p}_\perp$ must be uniquely and quickly determined for each ray relative to the object's local coordinate system [S1].

## Subtopics

### [NeRF Composition Framework](21239-nerf-composition-framework.md)
A general implicit pipeline for composing multiple NeRF objects that enables dynamic shadow casting and arbitrary rigid transformations without querying complex spatial structures.

### [NeDF Intersection Network](21629-nedf-intersection-network.md)
A multi-level classifier that predicts the signed distance parameter $\mu$ and a binary intersection indicator $\hat{\alpha}$ for a given ray, replacing direct regression to avoid depth discontinuity artifacts and floating artifacts at object edges.

## Operating characteristics
Not given in the sources.

## Background
Neural Depth Fields were introduced as part of a general implicit framework for fast NeRF composition and rendering [S1]. The method addresses the limitations of direct depth regression in neural fields by utilizing a local distance parameter $\mu$ [S1].

## Sources
[S1] A General Implicit Framework for Fast NeRF Composition and Rendering, https://doi.org/10.1609/aaai.v38i3.27952, fetched 2026-10-29
