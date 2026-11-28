# Feature Fusion for Radiance Decoding

Also called: Lidar-Hash Feature Fusion MLP

## Overview
Feature Fusion for Radiance Decoding is a method that predicts volumetric density and view-dependent color by combining sparse LiDAR-derived features with dense hash-encoded spatial features [S1]. It addresses the limitation of relying solely on point features by integrating multi-modal inputs, enabling robust reconstruction in street scenes where LiDAR data is sparse [S1]. Unlike sibling methods such as the [Feature Extraction Network](8151-feature-extraction-network.md) which focuses on generic CNN extraction, or the [Attention-based Multi-View Density Prediction](10674-attention-based-multi-view-density-prediction.md) which models view correlations, this method explicitly fuses geometric priors from LiDAR with appearance priors from hash grids within a single MLP pipeline [S1].

## How it works
**Inputs:**
1. Sample location $x \in \mathbb{R}^3$ [S1].
2. Viewing direction $d \in \mathbb{R}^3$ (normalized) [S1].
3. LiDAR point cloud data (used to compute $\phi_L(x)$) [S1].
4. Hash grid structure (used to compute $\phi_h(x)$) [S1].

**Outputs:**
1. Per-sample density $\alpha \in \mathbb{R}$ [S1].
2. Density embedding $h \in \mathbb{R}^k$ [S1].
3. View-dependent color $c \in \mathbb{R}^3$ [S1].

**Data Structures:**
*   **Hash Grid:** A structure storing feature vectors for spatial locations, queried to produce $\phi_h(x)$ [S1].
*   **LiDAR Feature $\phi_L(x)$:** Computed from $K$ nearest LiDAR points $p_i$ to $x$ [S1]. Distance weight: $w_i = \frac{1}{\|p_i - x\|}$ [S1]. Aggregation: If the set of neighbors $S_{x,K}$ is not empty, $\phi_L(x) = \frac{\sum_{i \in S_{x,K}} w_i f_{i,x}}{\sum_{i \in S_{x,K}} w_i}$ [S1]. If $S_{x,K}$ is empty, $\phi_L(x) = 0$ [S1]. Here $f_{i,x}$ represents the feature vector associated with point $p_i$ at location $x$ [S1].
*   **Spherical Harmonics (SH):** The direction $d$ is encoded into a vector $SH(d)$ using spherical harmonics basis functions [S1].

**Steps:**
1. **Compute LiDAR Encoding:** Identify the $K$ nearest neighbors of $x$ in the LiDAR point cloud [S1]. Calculate weights $w_i$ and aggregate features to produce $\phi_L(x)$ [S1].
2. **Compute Hash Encoding:** Query the hash grid at location $x$ to produce the feature vector $\phi_h(x)$ [S1].
3. **Concatenate Features:** Form the input vector $v_{feat} = [\phi_L(x), \phi_h(x)]$ by concatenating the LiDAR and hash feature vectors along the feature dimension [S1].
4. **Predict Density and Embedding:** Pass $v_{feat}$ through the density MLP $F_\alpha$ to obtain density $\alpha$ and density embedding $h$:
   $$ \alpha, h = F_\alpha([\phi_L(x), \phi_h(x)]) $$
   [S1]
5. **Encode Viewing Direction:** Compute the spherical harmonics encoding $SH(d)$ for the viewing direction $d$ [S1].
6. **Predict Color:** Concatenate the density embedding $h$ with the directional encoding $SH(d)$ to form $v_{color} = [h, SH(d)]$ [S1]. Pass $v_{color}$ through the color MLP $F_c$ to predict the color $c$:
   $$ c = F_c([h, SH(d)]) $$
   [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def feature_fusion_decode(x, d, lidar_points, lidar_feats, hash_query_fn, K, sh_degree, F_alpha_params, F_c_params):
    # 1. Compute LiDAR encoding phi_L(x)
    dists = np.linalg.norm(lidar_points - x, axis=1)
    idx = np.argsort(dists)[:K]
    neighbors = lidar_points[idx]
    neighbor_feats = lidar_feats[idx]
    
    if len(neighbors) == 0:
        phi_L = np.zeros(lidar_feats.shape[1])
    else:
        w = 1.0 / (dists[idx] + 1e-8)
        phi_L = np.sum(w[:, np.newaxis] * neighbor_feats, axis=0) / np.sum(w)

    # 2. Compute Hash encoding phi_h(x)
    # [G] Hash grid query step is not detailed in the sources.
    phi_h = hash_query_fn(x)

    # 3. Concatenate features
    v_feat = np.concatenate([phi_L, phi_h])

    # 4. Predict density alpha and embedding h
    alpha_h = mlp_forward(v_feat, F_alpha_params)
    alpha = alpha_h[0]
    h = alpha_h[1:]

    # 5. Encode viewing direction
    # [G] Standard spherical harmonics basis evaluation for direction d.
    sh_d = sh_basis(d, sh_degree)

    # 6. Predict color
    v_color = np.concatenate([h, sh_d])
    c = mlp_forward(v_color, F_c_params)

    return alpha, c

def mlp_forward(x, params):
    # params = [(W1, b1), (W2, b2), ...]
    for W, b in params:
        x = np.maximum(0, x @ W + b)
    return x

def sh_basis(d, degree):
    # [G] Standard spherical harmonics basis evaluation for direction d
    return np.zeros(3 * (degree + 1) ** 2)
```

**Parameters:**
*   $K$: Number of LiDAR neighbors. Not given in the sources.
*   $B$: Degree of spherical harmonics. Not given in the sources.
*   $F_\alpha, F_c$: MLP architectures. Not given in the sources.

**Edge Cases:**
*   If no LiDAR points are within the search radius for $x$, $\phi_L(x)$ is zero [S1].
*   Precision issues in distance calculation for $w_i$ if $x$ is very close to $p_i$ [G].

## Subtopics

### [Context-aware Radiance Field Decoder](13817-context-aware-radiance-field-decoder.md)
The Context-aware Radiance Field Decoder is a neural network architecture designed to predict volumetric color and density fields from low-resolution feature volumes. It addresses the computational cost of standard 3D convolutions by factorizing 3D spatial-depth operations into separable 2D and 1D convolutions, maintaining performance while improving memory and parameter efficiency. It differs from sibling methods like the [(2+1)D CNN Radiance Decoder](13823-2-1-d-cnn-radiance-decoder.md) by specifically employing a lightweight upsampler for full-resolution output and a specific residual block structure for the low-resolution processing.

### [(2+1)D CNN Radiance Decoder](13823-2-1-d-cnn-radiance-decoder.md)
The (2+1)D CNN Radiance Decoder is a factorized convolutional neural network that predicts the radiance field from a target view frustum volume by applying 2D spatial convolutions followed by 1D depth convolutions. It solves the problem of synthesizing novel views by aggregating multi-view information into a volumetric representation, offering a balance between performance and efficiency. It sets itself apart from sibling methods like the Ray Transformer (which models only depth context) and standard MLPs (point-only) by leveraging spatial context in the 2D plane, while using 50% fewer parameters than a full 3D CNN.

### [Sky Modeling Architecture](19350-sky-modeling-architecture.md)
The Sky Modeling Architecture is an encoder-decoder neural network that compresses a Low Dynamic Range (LDR) sky panorama into a low-dimensional feature vector and decodes it into a High Dynamic Range (HDR) sky dome. It distinguishes itself from sibling methods like the Context-aware Radiance Field Decoder by explicitly predicting the sun's peak intensity and direction as part of the latent representation, enabling precise control and editing of the brightest light source. This explicit parameterization outperforms representations relying solely on latent vectors or explicit azimuths in terms of HDR reconstruction accuracy and peak direction estimation.

### Detector Network Input
The detector $F_\psi$ takes the rendered image $x$ and the classification output $F_{\psi c}(x)$ as input to guide pixel-wise information decoding [S2].

### Foreground-Background Fusion
A module combining foreground and background features to preserve details from the dense surface points and handle large scenes via the bounded domain [S3]. It consists of several layers of multi-layer perceptrons [S3]. It concatenates the 128-dimensional foreground feature and 128-dimensional background feature, then passes them through 4 MLP layers to get the color of the rendered point [S3]. The rendering loss is supervised using an L2 loss between rendered pixels $r_p$ and ground truth $r_g$: $L_R = ||r_p - r_g||^2$ [S3]. The second stage training uses an Adam optimizer with an initial learning rate $5 \times 10^{-4}$ for $2 \times 10^6$ iterations, approximately 20 hours on a single A100 GPU [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Performance depends on the density and accuracy of the LiDAR point cloud. Sparse LiDAR may require accumulation of adjacent frames for sufficient supervision [S1].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method was introduced in "LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes" (2024) [S1]. It builds upon the idea that point features alone are insufficient for high-quality reconstruction in street scenes, particularly in low-texture areas like road surfaces [S1]. By fusing LiDAR-derived features with hash-encoded features, it aims to improve reconstruction quality and robustness [S1].

## Sources
[S1] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
[S2] StegaNeRF: Embedding Invisible Information within Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.00047, fetched 2026-10-26
[S3] PDF: Point Diffusion Implicit Function for Large-scale Scene Neural Representation, https://doi.org/10.48550/arXiv.2311.01773, fetched 2026-10-29
