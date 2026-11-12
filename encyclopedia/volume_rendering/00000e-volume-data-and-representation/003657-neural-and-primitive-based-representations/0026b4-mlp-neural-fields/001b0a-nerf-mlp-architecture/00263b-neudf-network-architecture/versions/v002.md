# NeUDF Network Architecture

Also called: NeUDF MLP, NeUDF encoder

## Overview
The NeUDF network architecture is a neural field method that represents 3D geometry as an Unsigned Distance Field (UDF) and color as a separate radiance field, using two distinct Multi-Layer Perceptrons (MLPs) [S1]. It differs from standard NeRF by modeling unsigned distance rather than volume density, allowing for explicit surface extraction via the zero-level set of the UDF, and by decoupling geometry and appearance into separate networks [S1]. This approach builds on the principles of [NeuS](2444-view-dependent-mlp.md) and [IDR](2444-view-dependent-mlp.md) but specifically utilizes the UDF formulation to handle non-manifold geometries and occlusions without prior assumptions about surface intersections [S1].

## How it works
The method employs two separate MLP networks: a UDF network $\Psi$ for geometry and a color network $c$ for appearance [S1].

**Inputs and Outputs:**
*   **UDF Network Input:** Spatial location $p(t) \in \mathbb{R}^3$ [S1].
*   **UDF Network Output:** Scalar UDF value and a 256-dimensional feature vector [S1].
*   **Color Network Inputs:** Spatial location $p(t)$, view direction $v \in \mathbb{R}^3$, surface normal $n$ (gradient of $\Psi$ at $p(t)$), and the 256-dim feature vector [S1].
*   **Color Network Output:** Directional emitted color $c \in \mathbb{R}^3$ [S1].

**Data Structures and Layout:**
1.  **UDF Network $\Psi(x)$:**
    *   **Structure:** 8 hidden layers, hidden size 256 [S1].
    *   **Activation:** Softplus with $\beta = 100$ for all hidden layers and the output layer [S1].
    *   **Skip Connections:** A skip connection concatenates the input $x$ to the output of the 4th hidden layer [S1].
    *   **Output:** The network outputs the UDF value along with the 256-dimensional feature vector [S1].
2.  **Color Network $c(x, v)$:**
    *   **Structure:** 4 hidden layers, hidden size 256 [S1].
    *   **Inputs:** Concatenation of positional encoding of $p(t)$, positional encoding of $v$, the surface normal $n$, and the feature vector [S1].
    *   **Regularization:** Normal regularization is applied to the gradient $n$ before it is used as input [S1].
    *   **Normalization:** Weight normalization is adopted [S1].
    *   **Positional Encoding:** Uses the same positional encoding scheme as NeuS [S1].

**Numbered Steps:**
1.  Sample a spatial location $p(t)$ along a ray.
2.  Forward $p(t)$ through the UDF network $\Psi$ to obtain the UDF value and the 256-dim feature vector [S1].
3.  Compute the surface normal $n$ by taking the gradient of the UDF network output with respect to the input $p(t)$: $n = \nabla \Psi(p(t))$ [S1].
4.  Apply normal regularization to $n$ [S1].
5.  Forward $p(t)$, view direction $v$, regularized gradient $n$, and the feature vector through the color network to obtain the emitted color [S1].

**Pitfalls:**
*   **Normal Regularization:** The gradient $n$ must be regularized before being fed into the color network to ensure stability and correct shading [S1].
*   **UDF vs SDF:** The use of Softplus activation on the output layer enforces non-negative values, which is consistent with an Unsigned Distance Field but constrains the representational capacity compared to signed fields [S1].

## Subtopics

### D-NeuS Network Architecture
A variant that uses a similar two-MLP structure (geometry and radiance) but initializes the geometry network weights to approximate the SDF of a unit sphere [S2, unofficial]. It employs hierarchical sampling (512 rays, 64 coarse, 64 fine, plus 32 points outside the unit sphere) and uses positional encoding with 6 frequencies for position and 4 for direction [S2, unofficial]. It also incorporates multi-view consistency by comparing reference views with neighboring source views using 32 feature channels [S2, unofficial].

## Operating characteristics
*   **Complexity:** The UDF network has 8 hidden layers (256 units) and the color network has 4 hidden layers (256 units) [S1].
*   **Performance:** The architecture was trained for 300k iterations on a single NVIDIA Titan RTX [S2, unofficial].
*   **Strengths:** The method maintains occlusion-aware properties for more than two surface intersections along a ray [S1]. It is suitable for reconstructing non-manifold geometries [S1].
*   **Limitations:** Requires computing the gradient of the UDF network for normals, which adds computational overhead during training and inference [S1].

## Background
The NeUDF architecture was introduced in the paper "NeUDF: Learning Neural Unsigned Distance Fields with Volume Rendering" [S1]. It builds upon the foundations laid by [IDR](2444-view-dependent-mlp.md) and [NeuS](2444-view-dependent-mlp.md), adapting the signed distance field approach to an unsigned distance field to better handle complex geometries and occlusions [S1].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S2] WACV 2023 Open Access Repository, https://openaccess.thecvf.com/content/WACV2023/html/Chen_Recovering_Fine_Details_for_Neural_Implicit_Surface_Reconstruction_WACV_2023_paper.html, fetched 2026-10-29 (unofficial)
