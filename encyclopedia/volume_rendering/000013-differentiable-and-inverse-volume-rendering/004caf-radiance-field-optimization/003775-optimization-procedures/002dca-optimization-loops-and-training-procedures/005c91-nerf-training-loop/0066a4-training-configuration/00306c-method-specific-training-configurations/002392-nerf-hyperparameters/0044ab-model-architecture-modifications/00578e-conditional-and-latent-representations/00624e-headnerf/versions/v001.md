# HeadNerf

Also called: HeadNeRF, NeRF-based parametric head model

## Overview
HeadNeRF is a NeRF-based parametric head model that integrates neural radiance fields into a parametric representation of the human head to render high-fidelity images in real-time [S1]. It supports direct control over rendering pose and semantic attributes: identity, expression, albedo, and illumination [S1]. Unlike standard NeRF, which treats geometry and appearance as monolithic implicit functions, HeadNeRF disentangles these factors by conditioning the density prediction on shape codes (identity, expression) and the feature prediction on appearance codes (albedo, illumination), while removing viewing direction dependence to mitigate dataset bias [S1]. It accelerates rendering from seconds to milliseconds by combining volumetric rendering with a 2D neural refinement network [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Latent codes: $z_{id}$ (identity), $z_{exp}$ (expression), $z_{alb}$ (albedo), $z_{ill}$ (illumination).
    *   Camera parameters $P$: extrinsic and intrinsic matrices.
    *   3D points $x \in \mathbb{R}^3$ sampled along rays $r(t)$ emitted from the camera center.
*   **Outputs:**
    *   Final rendered image $I \in \mathbb{R}^{3 \times 256 \times 256}$.

**Data Structures:**
1.  **Implicit Function $h_\theta$:** A Multi-Layer Perceptron (MLP) with parameters $\theta$.
2.  **Positional Encoding $\gamma(\cdot)$:** The standard NeRF positional encoding function [S1].
3.  **Feature Vector $F(x)$:** A high-dimensional vector $F(x) \in \mathbb{R}^{256}$ predicted by the MLP [S1].
4.  **Density $\sigma$:** A scalar value predicted by the MLP [S1].
5.  **2D Neural Renderer $\pi_\psi$:** A 2D convolutional network with parameters $\psi$, composed of 1x1 Conv2D layers and Leaky ReLU activations [S1].
6.  **Low-Resolution Feature Map $I_F$:** A tensor of shape $\mathbb{R}^{256 \times 32 \times 32}$ [S1].

**Numbered Steps:**

1.  **Conditioned Implicit Function Evaluation:**
    For a 3D point $x$ sampled along a ray $r(t)$, compute the positional encoding $\gamma(x)$. The MLP $h_\theta$ takes the concatenation of $\gamma(x)$, $z_{id}$, and $z_{exp}$ to predict the density $\sigma$ and an intermediate feature representation. The albedo $z_{alb}$ and illumination $z_{ill}$ codes are concatenated with the intermediate feature to predict the final high-dimensional feature vector $F(x)$ [S1]. The viewing direction $\mathbf{d}$ is explicitly excluded from the input to avoid capturing inconsistencies caused by dataset bias [S1].
    
    The mapping is defined as:
    $$
    h_\theta : (\gamma(x), z_{id}, z_{exp}, z_{alb}, z_{ill}) \rightarrow (\sigma, F)
    $$
    where $\sigma$ is primarily affected by $z_{id}$ and $z_{exp}$, while $F$ is affected by $z_{alb}$ and $z_{ill}$ [S1].

2.  **Volumetric Rendering of Feature Map:**
    Compute a low-resolution 2D feature map $I_F \in \mathbb{R}^{256 \times 32 \times 32}$ by integrating the feature vector $F(r(t))$ along the ray $r(t)$ using alpha compositing weights [S1].
    
    The formula for the feature map at ray $r$ is:
    $$
    I_F(r) = \frac{\int_0^\infty w(t) F(r(t)) dt}{\int_0^\infty w(t) dt}
    $$
    where the weight function $w(t)$ is:
    $$
    w(t) = \exp\left(-\int_t^\infty \sigma(r(s)) ds\right) \sigma(r(t))
    $$
    Here, $r(t)$ represents a ray emitted from the camera center [S1].

3.  **2D Neural Rendering Refinement:**
    Map the low-resolution feature map $I_F$ to the final high-resolution image $I \in \mathbb{R}^{3 \times 256 \times 256}$ using the 2D neural rendering module $\pi_\psi$ [S1].
    
    The module $\pi_\psi$ consists of 1x1 Conv2D layers and Leaky ReLU activation layers [S1]. The resolution is gradually increased through a series of upsampling layers, similar to the strategy in StyleNeRF [S1]. This step alleviates multi-view inconsistent artifacts [S1].

    The final image is formulated as:
    $$
    I = R(z_{id}, z_{exp}, z_{alb}, z_{ill}, P)
    $$
    where $R$ denotes the entire HeadNeRF rendering process [S1].

**Parameters:**
*   Feature vector dimension: 256 [S1].
*   Low-resolution feature map size: $32 \times 32$ [S1].
*   Final image size: $256 \times 256$ [S1].
*   Activation function in 2D renderer: Leaky ReLU [S1].

**Edge Cases and Pitfalls:**
*   **Dataset Bias:** Including viewing direction in the MLP input can capture undesired inconsistencies present in the training dataset. HeadNeRF removes viewing direction to prevent this [S1].
*   **Multi-view Inconsistency:** The coarse volumetric rendering may produce artifacts. The 2D refinement network is specifically designed to alleviate these inconsistencies [S1].

## Operating characteristics
*   **Speed:** Rendering time per frame is reduced from 5 seconds (standard NeRF) to 25 milliseconds [S1]. The method can exceed 40 frames per second (fps) on modern GPUs without sacrificing rendering quality [S1].
*   **Quality:** Generates high-fidelity head images, capturing fine details such as gaps between teeth, wrinkles, and beards [S1].
*   **Control:** Supports direct semantic control over identity, expression, albedo, and illumination [S1].
*   **Limitations:** The model is specialized for human heads/face images. It relies on the assumption that geometry is controlled by identity/expression and appearance by albedo/illumination [S1].

## Background
HeadNeRF was proposed as a novel NeRF-based parametric head model [S1]. It addresses the limitation of traditional parametric head models (like 3DMM) which use textured meshes and may lack high-fidelity details, and the limitation of standard NeRF which is computationally expensive for real-time applications [S1]. By integrating 2D neural rendering, it achieves real-time performance [S1]. The trained parametric model is available at https://github.com/CrisHY1995/headnerf [S1].

## Sources
[S1] HeadNeRF: A Realtime NeRF-based Parametric Head Model, https://doi.org/10.1109/cvpr52688.2022.01973, fetched 2026-10-26
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
