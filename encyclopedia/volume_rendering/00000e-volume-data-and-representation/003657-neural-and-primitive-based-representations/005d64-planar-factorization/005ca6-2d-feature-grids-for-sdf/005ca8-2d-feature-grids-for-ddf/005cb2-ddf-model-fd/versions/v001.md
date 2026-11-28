# DDF Model (fd)

Also called: Directional Distance Function Model

## Overview
The DDF model is a neural scene representation that predicts the directional distance to an object's surface and the probability of a ray hitting that surface for points on the unit sphere. It replaces iterative sphere tracing with a single forward pass, accelerating inverse rendering algorithms by reducing network evaluations per iteration [S1]. It differs from standard SDF models by explicitly encoding directional information via factorized 2D feature grids, allowing it to resolve surface geometry along specific viewing directions without spatial ambiguity [S1].

## How it works
The DDF model $f_d$ takes a point on the unit sphere $p$, a direction vector $r$, a latent code $z$, and high-dimensional features sampled from 2D feature grids $f_{pd}(p, r)$ as input. It outputs a directional distance $d \in \mathbb{R}^+$ and a ray hit probability $\sigma \in [0, 1]$ [S1].

**Inputs and Outputs:**
*   **Input:** $(p, r) \in \mathbb{R}^6$, where $p \in S^2$ is a point on the unit sphere and $r \in S^2$ is a direction.
*   **Input:** $z \in \mathbb{R}^{R_L}$, a latent code per shape.
*   **Input:** $f_{pd}(p, r) \in \mathbb{R}^{15 \times K}$, the feature vector sampled from 15 2D feature grids.
*   **Output:** $(d, \sigma)$, where $d$ is the positive distance to the surface along $r$, and $\sigma$ is the probability that the ray hits the surface [S1].

**Data Structures:**
1.  **2D Feature Grids:**
    *   Count: 15 grids.
    *   Resolution: $512 \times 512$.
    *   Feature Dimension: 32 per grid.
    *   Sampling: Features are sampled using bilinear interpolation based on the projection of $(p, r)$ onto the grid planes [S1].
2.  **MLP ($f_d$):**
    *   Structure: 3 blocks of fully-connected layers.
    *   Hidden Layers: Each block has 1 hidden layer of dimension 512.
    *   Activations: ReLU for all layers except the last layer (no activation) [S1].
    *   Skip Connections: Skip connections from the input to each block [S1].
    *   Parameters: $\Theta_d$ [S1].

**Algorithm:**
1.  **Positional Encoding:** Encode each dimension of the input tuple $(p, r)$ using positional encoding with 3 frequencies [S1].
2.  **Feature Sampling:** Sample the high-dimensional feature vector $f_{pd}(p, r)$ from the 15 2D feature grids using bilinear interpolation [S1].
3.  **Concatenation:** Concatenate the encoded $(p, r)$, the latent code $z$, and the sampled features $f_{pd}(p, r)$ [S1].
4.  **MLP Forward Pass:** Pass the concatenated vector through the MLP defined by $\Theta_d$ [S1].
5.  **Output:** The MLP outputs $d$ and $\sigma$ [S1].

**Formulas:**
The model is defined as:
$$ f_d((p, r), z, f_{pd}(p, r); \Theta_d) = (d, \sigma) $$
[S1]

The relationship between the DDF and the Signed Distance Function (SDF) is:
$$ \text{SDF}(x) = \min_r \text{DDF}(x, r) $$
where $s = \text{SDF}(x)$ is the signed distance at point $x \in \mathbb{R}^3$ and $r \in S^2$ is a direction from $x$ pointing towards the surface [S1].

The SDF value at the distance predicted by the DDF (where $\sigma=1$) must be zero:
$$ \text{SDF}(p + d_{\sigma=1}r) = 0, \quad d_{\sigma=1} = \text{DDF}(p, r) $$
[S1]

**Parameters:**
*   $\Theta_d$: Learnable parameters of the DDF MLP [S1].
*   $z$: Latent code per shape, dimension $R_L$ [S1].
*   $(p, r)$: Point on the unit sphere and direction, dimension $R^6$ [S1].
*   $f_{pd}(p, r)$: High-dimensional feature vector from DDF grids, dimension $R^{15 \times K}$ [S1].

**Edge Cases and Pitfalls:**
*   The model assumes 3D shapes are inside the unit sphere [S1].
*   The directional distance $d$ is strictly positive [S1].
*   The hit probability $\sigma$ is bounded in $[0, 1]$ [S1].

## Subtopics

### DDF Loss
A supervised loss using L1 distance to ensure the predicted directional distances $d$ are close to the ground truth directional distances $d_{GT}$ [S1]. Defined as $\mathcal{L}_d(d) = \|d - d_{GT}\|_1$ [S1].

### SDF and DDF Neural Network Architecture
A generative model consisting of two neural networks (SDF and DDF) conditioned on latent codes and high-dimensional features sampled from learned 2D feature grids, inspired by Pi-GAN [S1]. The SDF and DDF models share a latent space for each shape category [S1]. The SDF model takes a 3D point $x \in \mathbb{R}^3$ and latent code $z$, sampling from 3 2D feature grids ($M^s_{xy}, M^s_{yz}, M^s_{zx}$) via bilinear interpolation [S1]. The DDF model takes $(p, r) \in \mathbb{R}^6$ and $z$, sampling from 15 2D feature grids [S1].

### Directional Distance Function (DDF)
A neural scene representation defined on the surface of the unit sphere ($p \in S^2$) that predicts two outputs for a given direction $r$: (1) the directional distance $d$ to the object's surface along ray $r$, and (2) the probability $\sigma \in [0, 1]$ that the ray hits the surface [S1]. The model assumes 3D shapes are inside the unit sphere [S1].

## Operating characteristics
*   **Time Complexity:** Replaces iterative sphere tracing with a single forward pass, reducing the number of network evaluations required to find the object surface to 1 per iteration [S1].
*   **Performance:** The algorithm utilizing DDF is 15.5× faster than previous methods [S1].
*   **Memory:** Uses 15 2D feature grids of resolution $512 \times 512$ with feature dimension 32 for the DDF model [S1].
*   **Strengths:** Accelerates inverse rendering algorithms; avoids sphere tracing [S1].
*   **Limitations:** Assumes shapes are inside the unit sphere [S1].

## Background
The DDF model was proposed in "Enabling Fast Inverse Rendering" to accelerate inverse rendering algorithms by replacing sphere tracing with a directional distance function [S1]. It is part of a generative model inspired by Pi-GAN, using 2D feature grids to condition neural networks on high-dimensional features [S1]. The method was evaluated on the ShapeNet dataset [S1].

## Sources
[S1] FIRe: Fast Inverse Rendering Using Directional and Signed Distance Functions, https://www.youtube.com/watch?v=Qj0ymDlRYZs, fetched 2026-10-30
