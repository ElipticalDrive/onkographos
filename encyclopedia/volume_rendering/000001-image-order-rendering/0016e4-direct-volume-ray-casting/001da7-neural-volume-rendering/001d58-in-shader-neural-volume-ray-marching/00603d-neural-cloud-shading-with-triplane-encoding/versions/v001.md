# Neural Cloud Shading with Triplane Encoding

Also called: Environmental Volumetric Neural Shading, Mesh-based Neural Volumetric Rendering, Neural Shading of Clouds, Neural Volumetric Shading for Clouds

## Overview
Neural Cloud Shading with Triplane Encoding is a real-time rendering method that replaces explicit volumetric ray marching with a neural shading model applied to mesh geometry [S1]. It solves the problem of high-fidelity cloud relighting by approximating complex light scattering and atmospheric illumination using a multilayer perceptron (MLP) evaluated per pixel on a rasterized cloud mesh [S1]. Unlike density-field approaches such as [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md) or traditional ray marching, this method eliminates volume integration entirely, relying instead on a rasterization-based pipeline with triplane feature encoding to capture view-dependent opacity and parallax [S1].

## How it works
**Inputs and Outputs:**
The method takes a mesh representation of a volumetric cloud, a sky illumination model (Nishita sky model), and a sun direction defined by spherical coordinates $(\theta, \phi)$ [S1]. The output is a final color $c_{tilde}$ and transparency $\alpha$ for each pixel, composited over the background [S1].

**Data Structures:**
1.  **Triplane Feature Maps:** Six 32-bit floating-point RGBA textures at $512 \times 512$ resolution. Three planes (XY, XZ, YZ) are used, with 8 features per plane (4 channels per texture) [S1]. These are stored as learnable Parameter tensors in PyTorch during training and as textures during inference [S1].
2.  **MLP Weights:** Stored in 16-bit floating-point precision to support tensor core operations [S1].
3.  **Depth/Position Attachments:**
    *   Back-face position: 16-bit RGB color attachment storing normalized positions of the furthest visible surface [S1].
    *   Front-face depth: Standard depth buffer storing the closest visible surface [S1].
    *   Shadow map: Generated from the sun's perspective [S1].

**Numbered Steps:**

1.  **Mesh Conversion:** Convert the volumetric cloud asset into a mesh representation [S1].
2.  **Training Data Generation:** Render the mesh in Blender under diverse viewing and lighting conditions using path tracing to generate ground-truth RGBA values [S1].
3.  **Thickness Estimation (Training):** Render the mesh back faces (front-face culling, greater-than depth test) and front faces (back-face culling, less-or-equal depth test) to compute per-pixel thickness $\delta$ as the difference between front and back positions [S1].
4.  **Visibility Computation (Training):** Compute a diffuse direct light pass and apply a smoothstep function with lower bound 0.0 and upper bound 0.1 to produce soft visibility values [S1].
5.  **Model Training:** Train an MLP (4 hidden layers, 64 neurons, ReLU activations) using Adam optimizer (initial LR $10^{-3}$, halved every 50 epochs, batch size $1024^2$, 250 epochs) [S1]. The input is the concatenation of triplane features, sun direction $(\theta, \phi)$, and thickness $\delta$. The output is two RGB colors ($c_D, c_I$) and $\alpha$. The loss is the L2 distance between $\log(\tilde{c} + 1)$ and $\log(c_{GT} + 1)$ [S1].
6.  **Back-Face Position Pre-Pass (Runtime):** Render back faces with front-face culling and a greater-than depth test. Store normalized positions in a 16-bit RGB color attachment [S1].
7.  **Front-Face Depth Pre-Pass (Runtime):** Render front faces with back-face culling and a less-or-equal depth test to a depth map (Scene Depth) [S1].
8.  **Shadow Pre-Pass (Runtime):** Generate a shadow map from the light source's point of view (sun direction) using back-face culling and disabled depth test [S1].
9.  **Final Cloud Color Pass (Runtime):**
    *   Use a depth test set to 'equal' against the front-face depth map to identify the visible surface [S1].
    *   Reconstruct front-face position in the vertex shader and normalize it to the bounding box [S1].
    *   Compute thickness $\delta$ as the difference between the front-face position and the back-face position (from Step 6) [S1].
    *   Apply a smoothstep function to $\delta$ with bounds 0.0 and 0.4 to prevent contouring artifacts [S1].
    *   Project the normalized position onto XY, XZ, and YZ planes to obtain texture coordinates. Sample the 6 triplane textures using bilinear interpolation and sum the features [S1].
    *   Concatenate summed triplane features, sun direction $(\theta, \phi)$, and smoothed thickness $\delta$ [S1].
    *   Evaluate the MLP to output $c_D$, $c_I$, and $\alpha$ [S1].
    *   Interpolate between $c_D$ and $c_I$ based on precomputed visibility values (from shadow map using PCF) to obtain final color $\tilde{c}$ [S1].
    *   Blend $\tilde{c}$ and $\alpha$ with the background [S1].

**Formulas:**
*   **Triplane Encoding:** For a point $p$, features $F_{XY}, F_{YZ}, F_{XZ}$ are sampled via bilinear interpolation and summed: $F_{total} = F_{XY} + F_{YZ} + F_{XZ}$ [S1].
*   **Output Activation:** $g(x) = e^{x-3}$ applied to the output layer [S1].
*   **Loss Function:** $L = || \log(\tilde{c} + 1) - \log(c_{GT} + 1) ||_2^2$ [S1].
*   **Visibility Interpolation:** $\tilde{c} = V \cdot c_D + (1 - V) \cdot c_I$, where $V$ is the smoothstep visibility value [S1].
*   **Thickness Smoothing:** $\delta_{smooth} = \text{smoothstep}(0.0, 0.4, \delta)$ [S1].

**Parameters:**
*   **MLP Architecture:** 4 hidden layers, 64 neurons per layer ($4 \times 64$) [S1].
*   **Triplane Resolution:** $512 \times 512$ [S1].
*   **Features per Plane:** 8 (stored in 2 RGBA textures per plane) [S1].
*   **Sun Direction:** 2D vector $(\theta, \phi)$ [S1].
*   **Thickness Bounds:** 0.0 to 0.4 [S1].
*   **Visibility Thresholds:** 0.0 to 0.1 [S1].
*   **Mipmapping:** Mipmap level 4 applied to triplane features for final rendering to reduce noise [S1].

**Edge Cases and Pitfalls:**
*   **Missing Thickness:** Without the thickness parameter, the model fails to account for accumulated density, leading to inconsistent transparency and erroneous semi-transparent regions [S1].
*   **Opacity Saturation:** Opacity does not saturate sufficiently toward the cloud interior without smoothing, causing contouring artifacts [S1].
*   **High-Frequency Noise:** High-frequency components in triplane features introduce noise; mitigated by mipmapping at the cost of PSNR [S1].
*   **View Direction:** View direction is excluded from inputs as it was found less critical than thickness for view dependency [S1].

## Subtopics

### Relightable Neural Assets (RNA)
The foundational method for neural shading of assets using MLPs. Neural Cloud Shading adapts RNA by adding thickness and sky illumination. [RNA](7049-radiance-predicting-neural-networks-rpnn.md)

### In-Shader Neural Volume Ray Marching
The parent method where the MLP is evaluated during ray marching. Neural Cloud Shading differs by using rasterization and mesh geometry instead of ray marching. [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md)

## Operating characteristics
*   **Time Complexity:** Real-time performance achieved. Frame times are 3.7 ms for high-poly meshes and 0.41 ms for low-poly meshes on NVIDIA GeForce RTX 4080 [S1].
*   **Memory Complexity:** Requires storage for 6 triplane textures ($512 \times 512 \times 4 \times 32$-bit) and MLP weights [S1].
*   **Performance:** Training takes ~20 minutes on NVIDIA A100 GPU [S1].
*   **Quality:** 22.315 dB PSNR and 0.059 FLIP error for the $4 \times 64$ architecture with thickness [S1].
*   **Strengths:** Eliminates ray marching, enabling real-time performance; preserves volumetric appearance with full parallax; supports real-time relighting via sky model inputs [S1].
*   **Limitations:** Requires mesh conversion of volumetric assets; relies on rasterization shadows which may not capture complex volumetric self-shadowing as accurately as ray tracing; quality depends on training data diversity [S1].
*   **When to Use:** Real-time applications requiring high-fidelity, relightable clouds where ray marching is too expensive [S1].
*   **When Not to Use:** When exact volumetric integration is required or when assets cannot be converted to meshes [S1].

## Background
Neural Cloud Shading with Triplane Encoding was introduced by Rikard Olajos, Michael Doggett, and Prashant Goswami in 2026 in the paper "Environmental Volumetric Neural Shading of Clouds for Real-Time Rendering" [S1]. It builds upon the Relightable Neural Assets (RNA) framework, adapting it for volumetric clouds by incorporating per-pixel thickness and sky illumination models [S1]. The method addresses the computational cost of traditional density-field ray marching approaches like those by Hillaire (2016) and Schneider (2023) [S1].

## Sources
[S1] Environmental Volumetric Neural Shading of Clouds for Real-Time Rendering, https://doi.org/10.1145/3820020, fetched 2026-10-28
