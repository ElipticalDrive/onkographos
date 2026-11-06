# Probabilistic Prediction of Pixel-Aligned Gaussians

Also called: Discrete Depth Bucketing for Gaussians, Probabilistic Gaussian Prediction

## Overview
This method predicts 3D Gaussian primitive parameters (mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, spherical harmonics $\mathbf{S}$) from per-pixel features by modeling the depth distribution as a discrete probability density over depth buckets, rather than regressing a point estimate. It solves the problem of local minima inherent in direct regression of Gaussian centers and enables the use of differentiable training where non-differentiable spawning/pruning heuristics from 3D-GS cannot be applied. It sets itself apart from direct regression methods by predicting a probability distribution $\phi$ over depth bins, which provides an approximate 1dB PSNR performance boost [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Per-pixel feature vector $\mathbf{F}[\mathbf{u}]$ at pixel coordinate $\mathbf{u}$, camera origin $\mathbf{o}$, ray direction $\mathbf{d}_{\mathbf{u}}$, depth range $[d_{\text{near}}, d_{\text{far}}]$, and number of depth buckets $Z$.
*   **Output:** A tuple $(\boldsymbol{\mu}, \boldsymbol{\Sigma}, \alpha, \mathbf{S})$ representing one Gaussian primitive per pixel (simplified case $M=1$) [S1].

**Data Structures:**
*   **Depth Buckets $\mathbf{b} \in \mathbb{R}^Z$:** A vector representing discrete depth values in disparity space. The $z$-th element $\mathbf{b}_z$ is defined as:
    $$
    \mathbf{b}_z = \left( \left(1 - \frac{z}{Z}\right) \left(\frac{1}{d_{\text{near}}} - \frac{1}{d_{\text{far}}}\right) + \frac{1}{d_{\text{far}}} \right)^{-1}
    $$
    where $z \in \{0, \dots, Z-1\}$ is the bucket index [S1].
*   **Probability Vector $\boldsymbol{\phi} \in \mathbb{R}^Z$:** Discrete probabilities predicted by the network, normalized via softmax such that $\sum_z \phi_z = 1$. $\phi_z$ represents the probability that a surface exists in depth bucket $\mathbf{b}_z$ [S1].
*   **Offset Vector $\boldsymbol{\delta} \in \mathbb{R}^Z$:** Per-bucket center offsets $\delta_z \in [0, 1]$ that adjust the depth between bucket boundaries [S1].

**Parameters:**
*   $Z$: Number of depth bins/buckets discretizing the depth range. Not given in the sources.
*   $d_{\text{near}}, d_{\text{far}}$: Near and far planes defining the depth range. Not given in the sources.
*   $M$: Number of Gaussian primitives predicted per pixel. The text simplifies to $M=1$ for the explanation [S1].

**Steps:**
1.  **Prediction:** A neural network $f$ processes the per-pixel feature $\mathbf{F}[\mathbf{u}]$ to predict the depth probability vector $\boldsymbol{\phi}$, the offset vector $\boldsymbol{\delta}$, the covariance $\boldsymbol{\Sigma}$, and the spherical harmonics coefficients $\mathbf{S}$:
    $$
    (\boldsymbol{\phi}, \boldsymbol{\delta}, \boldsymbol{\Sigma}, \mathbf{S}) = f(\mathbf{F}[\mathbf{u}])
    $$
    [S1]
2.  **Sampling:** Sample a depth bucket index $z$ from the discrete probability distribution parameterized by $\boldsymbol{\phi}$:
    $$
    z \sim p_{\boldsymbol{\phi}}(z)
    $$
    [S1]
3.  **Mean Computation:** Compute the Gaussian mean $\boldsymbol{\mu}$ by unprojecting the sampled depth bucket $\mathbf{b}_z$ adjusted by the offset $\delta_z$ along the ray direction $\mathbf{d}_{\mathbf{u}}$ from the camera origin $\mathbf{o}$:
    $$
    \boldsymbol{\mu} = \mathbf{o} + (\mathbf{b}_z + \delta_z)\mathbf{d}_{\mathbf{u}}
    $$
    [S1]
4.  **Opacity Assignment:** Set the Gaussian opacity $\alpha$ equal to the probability of the sampled bucket:
    $$
    \alpha = \phi_z
    $$
    [S1]
5.  **Return:** Return the tuple $(\boldsymbol{\mu}, \boldsymbol{\Sigma}, \alpha, \mathbf{S})$ [S1].

**Backpropagation (Reparameterization Trick):**
To make sampling differentiable, gradients of the loss $\mathcal{L}$ with respect to the opacity $\alpha$ are assigned to the gradients of the depth probability buckets $\boldsymbol{\phi}$:
$$
\nabla_{\boldsymbol{\phi}} \mathcal{L} = \nabla_{\alpha} \mathcal{L}
$$
This allows gradient descent to increase the opacity of correctly sampled depths (concentrating probability mass) and decrease it for incorrect samples [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_prediction(F_u, b, o, d_u):
    """
    Predicts a single Gaussian primitive per pixel using probabilistic depth bucketing.
    
    Args:
        F_u: Per-pixel feature vector.
        b: Depth bucket vector in disparity space, shape (Z,).
        o: Camera origin vector.
        d_u: Ray direction vector.
        
    Returns:
        mu: Gaussian mean vector.
        Sigma: Gaussian covariance matrix.
        alpha: Gaussian opacity.
        S: Spherical harmonics coefficients.
    """
    # 1. Predict parameters from features
    # f is the neural network defined elsewhere
    phi, delta, Sigma, S = f(F_u)
    
    # 2. Sample depth bucket index z from discrete distribution phi
    # z is an integer index
    z = sample_discrete(phi)
    
    # 3. Compute mean mu
    # b_z is the depth value of the sampled bucket
    # delta_z is the offset for the sampled bucket
    b_z = b[z]
    delta_z = delta[z]
    mu = o + (b_z + delta_z) * d_u
    
    # 4. Set opacity alpha to the probability of the sampled bucket
    alpha = phi[z]
    
    return mu, Sigma, alpha, S
```

**Edge Cases and Pitfalls:**
*   **Local Minima:** Directly regressing the Gaussian center $\boldsymbol{\mu}$ is susceptible to local minima [S1].
*   **Non-differentiable Heuristics:** The method avoids reliance on spawning and pruning heuristics from 3D-GS, which are not differentiable [S1].
*   **Precision:** The depth buckets are defined in disparity space to handle non-linear depth distributions, but precision depends on the number of buckets $Z$ [S1].

## Subtopics

### [Occlusion-Aware IBR-based Color Prediction](8441-occlusion-aware-ibr-based-color-prediction.md)
This method predicts the color of a 3D Gaussian primitive by aggregating information from multiple nearby reference frames, using monocular depth priors to filter out occluded regions. It addresses the problem of retrieving inaccurate color information from 2D reference images due to occlusions, which degrades rendering quality in generalizable Gaussian Splatting. It sets itself apart from direct regression methods by leveraging Inter-View Consistency and explicit visibility estimation to distinguish visible from invisible features, allowing the model to focus on visible views without manual thresholding.

## Operating characteristics
*   **Performance:** The probabilistic prediction method leads to an approximate 1dB PSNR performance boost compared to the baseline of directly regressing the depth [S1].
*   **Complexity:** Not given in the sources.
*   **Strengths:** Overcomes local minima associated with direct regression; enables differentiable training without non-differentiable heuristics [S1].
*   **Limitations:** Requires sampling during the forward pass, which introduces stochasticity; relies on the reparameterization trick for gradient flow [S1].
*   **When to use:** When direct regression of Gaussian centers leads to local minima or when differentiable training is required without relying on 3D-GS heuristics [S1].
*   **When not to use:** When deterministic output is strictly required without stochastic sampling [S1].

## Background
The method was introduced in "PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction" [S1]. It addresses limitations in predicting Gaussian parameters from reference views, specifically the susceptibility of direct regression to local minima and the inability to use non-differentiable spawning/pruning heuristics [S1]. The approach predicts Gaussians from both reference views, with the final set being the union of predicted Gaussians [S1].

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
