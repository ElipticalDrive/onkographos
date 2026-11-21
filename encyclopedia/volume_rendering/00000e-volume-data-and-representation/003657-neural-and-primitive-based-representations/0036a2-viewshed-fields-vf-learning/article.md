# Viewshed Fields (VF) Learning

Also called: Normalizing Flow for Viewshed, VF + NF, VF Learning, VF with Normalizing Flows Integration, Viewshed Fields (VF) with Normalizing Flows (NF) Integration

## Overview
Viewshed Fields (VF) Learning is a method that learns an implicit probability distribution over oriented points (3D location $\mathbf{x}$ and viewing direction $\mathbf{d}$) to identify which viewpoints are likely to contain valid surface information for a given scene [S1]. Unlike standard NeRF rendering which traverses rays to compute color and density, VF Learning uses Normalizing Flows (NF) to map the distribution of observed oriented points to a simple latent Gaussian space [S1]. This allows for the generation of novel, geometrically consistent camera poses by sampling from the latent space and inverting the flow, which is particularly useful for NeRF registration and novel view initialization [S1]. It differs from sibling methods like [VoISDF](821-voisdf-volume-rendering-and-adaptive-sampling.md) or [Gaussian Head Avatar (GHA)](1478-gaussian-head-avatar-gha.md) by explicitly modeling the *likelihood of a view* rather than just the radiance or geometry at a point, enabling generative sampling of camera parameters [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained or jointly trained NeRF model (providing density $\sigma$ and color $\mathbf{c}$) and a set of training images with known camera poses.
*   **Output:** A Normalizing Flow model $f$ that maps oriented points $\mathbf{x} \in \mathbb{R}^3, \mathbf{d} \in \mathbb{R}^3$ to a latent vector $\mathbf{z} \in \mathbb{R}^6$, and the inverse mapping $f^{-1}$ for generating novel views.

**Data Structures:**
*   **Normalizing Flow Model ($f$):** Implemented using Real-NVP architecture with 4 layers [S1]. The model takes a 6-dimensional input vector (concatenation of $\mathbf{x}$ and $\mathbf{d}$) and outputs a 6-dimensional latent vector $\mathbf{z}$.
*   **Latent Space ($Z$):** A 6-dimensional diagonal Gaussian distribution $\mathcal{N}(\mathbf{0}, \mathbf{I})$ [S1].

**Numbered Steps:**

1.  **Surface Point Sampling:** During NeRF training, for each ray with origin $\mathbf{o}$ and direction $\mathbf{d}$, sample a single surface point $\tilde{\mathbf{x}}$. The depth is defined as the median of the density weights accumulated along the ray [S1].
    $$ \tilde{\mathbf{x}} = \mathbf{o} + \text{depth} \cdot \mathbf{d} $$
    [S1]

2.  **Oriented Point Formation:** Create the oriented point pair $(\tilde{\mathbf{x}}, \mathbf{d})$. The direction $\mathbf{d}$ is constant along the ray, while $\tilde{\mathbf{x}}$ represents the specific location on the object surface [S1].

3.  **Flow Mapping:** Input the oriented point $(\tilde{\mathbf{x}}, \mathbf{d})$ into the Normalizing Flow model $f$ to obtain the latent representation $\mathbf{z}$:
    $$ \mathbf{z} = f(\tilde{\mathbf{x}}, \mathbf{d}) $$
    [S1]

4.  **Training Objective:** Train the flow model $f$ by maximizing the log-likelihood of the data distribution $p_X(\mathbf{x})$ (where $\mathbf{x}$ here refers to the oriented point) using the change of variables formula [S1]. The optimization minimizes the negative log-likelihood:
    $$ \log(p_X(\mathbf{x})) = \log(p_Z(f(\mathbf{x}))) + \sum_{i=1}^{K} \log \left| \det \left( \frac{df_i}{df_{i-1}} \right) \right| $$
    where $p_Z$ is the probability density of the 6-dimensional diagonal Gaussian in latent space, and $K$ is the number of coupling layers in the Real-NVP architecture [S1].

5.  **Novel View Generation (Inference):**
    a. Sample $K$ points $\mathbf{z}_k$ from the 6-dimensional Gaussian latent space $Z$ [S1].
    b. Invert the flow to recover oriented points with high viewshed scores:
    $$ (\mathbf{x}_k, \mathbf{d}_k) = f^{-1}(\mathbf{z}_k) $$
    [S1]
    c. Reconstruct the camera origin $\mathbf{o}_k$ for each oriented point. Since $\mathbf{x}_k = \mathbf{o}_k + \text{depth}_k \cdot \mathbf{d}_k$, and $\mathbf{d}_k$ is known, the origin can be derived if depth is estimated or assumed, or used directly to define the ray for rendering [S1].
    d. Render the view using the NeRF model at camera position $\mathbf{o}_k$ and direction $\mathbf{d}_k$ [S1].

**Parameters:**
*   **Architecture:** Real-NVP with 4 layers [S1].
*   **Training Schedule:** VF training can occur jointly with NeRF density/RGB training or after NeRF parameters are fixed (working on a pre-trained NeRF) [S1].

**Edge Cases and Pitfalls:**
*   **Depth Estimation:** The accuracy of $\tilde{\mathbf{x}}$ depends on the median depth estimation. If the NeRF density is noisy, the surface point may be inaccurate, leading to poor VF learning [S1].
*   **Noise Robustness:** The method is robust to noise in the input oriented points (up to 20% of scene size), as the goal is plausible initialization rather than exact pose recovery [S1].
*   **Illumination Changes:** VF minimizes a photometric loss. While it works with different illumination conditions, registration errors may arise if COLMAP (used as ground truth) minimizes a geometric objective while VF minimizes a photometric one, leading to ambiguous superiority in low-error scenarios [S1].

## Subtopics

### [Novel View Generation via Viewshed Field](8948-novel-view-generation-via-viewshed-field.md)
This method generates novel views by treating the generation of high-confidence oriented points as a generative process using Normalizing Flows (NF). It samples from a learned latent Gaussian distribution, inverts the mapping to obtain oriented points $(x, d)$ with high Viewshed Field (VF) scores, and reconstructs camera origins to render new perspectives. It differs from standard NeRF rendering by explicitly sampling the latent space of valid viewing configurations rather than traversing a ray, ensuring generated views are geometrically consistent with the learned VF representation.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** VF-generated novel views yield visually indistinguishable registration results compared to using original images, despite slightly higher numerical error due to COLMAP's geometric estimation inaccuracies versus the method's photometric objective [S1].
*   **Strengths:** Can be used to sample novel camera view points, sample a colored 3D point cloud, or guide ray sampling during NeRF registration optimization [S1]. Robust to noise in oriented points [S1].
*   **Limitations:** Performance depends on the quality of the underlying NeRF density estimation for surface point sampling [S1]. Numerical errors may be higher than COLMAP-based methods in scenarios with minimal registration error due to the mismatch between photometric (VF) and geometric (COLMAP) objectives [S1].
*   **When to use:** When NeRF registration or novel view initialization is required, especially in rigid scenes [S1].
*   **When not to use:** When exact geometric pose estimation is prioritized over photometric consistency, or when illumination changes are severe and not handled by the NeRF [S1].

## Background
Viewshed Fields (VF) were introduced in the paper "VF-NeRF: Viewshed Fields for Rigid NeRF Registration" by L. Segre and S. Avidan [S1]. The method addresses the problem of NeRF registration by learning an implicit function that captures the likelihood of 3D surface points being viewed by original cameras [S1]. It combines the standard NeRF representation with Normalizing Flows to enable generative sampling of camera poses [S1]. The method has been evaluated on diverse datasets including front-facing scenarios, object-centric videos, and synthetic objects, achieving state-of-the-art results in many registration tasks [S1].

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
