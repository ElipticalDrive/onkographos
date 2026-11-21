# Volume Rendering Prior (VRPrior)

Also called: learnable differentiable renderer, neural weight function

## Overview
VRPrior replaces handcrafted, 1D ray-centric equations for mapping unsigned distance fields (UDF) to opaque density with a learnable neural network $f_w$ that incorporates 3D context from neighboring points along the ray [S1]. It solves the bias and sensitivity to outliers inherent in standard volume rendering formulations by learning a robust mapping from local geometric structures to rendering weights, enabling accurate surface reconstruction from multi-view RGB images without requiring explicit depth supervision during the final optimization stage [S1].

## How it works
VRPrior operates in two distinct phases: **Prior Training** (learning the mapping function $f_w$) and **Generalization** (optimizing the scene's UDF and color using the fixed prior) [S1].

### Inputs and Outputs
*   **Inputs:**
    *   During Prior Training: Ground truth meshes $\{S_h\}_{h=1}^H$, random camera views.
    *   During Generalization: Multi-view RGB images $\{I_j\}_{j=1}^J$, camera poses.
*   **Outputs:**
    *   Prior Training: Optimized network parameters $\theta_w$ for $f_w$.
    *   Generalization: Optimized UDF field $f_u$, color network $f_c$, and rendered RGB/Depth images.

### Data Structures and Network Layout
The core component is the neural weight function $f_w$, which predicts opaque density $\sigma_n$ for a query point $q_n$ [S1].

1.  **Neighbor Selection:** For each query point $q_n$ along a ray, identify $M$ nearest neighbors $\{q'_m\}_{m=1}^M$ along the same ray [S1].
2.  **Feature Grouping:** The $M$ neighbors are divided into three groups based on their distance to $q_n$:
    *   Group 1: $M_1 = 10$ nearest neighbors.
    *   Group 2: $M_2 = 20$ next nearest neighbors.
    *   Group 3: $M_3 = 30$ furthest neighbors.
    *   Total $M = M_1 + M_2 + M_3 = 60$ [S1].
3.  **Input Features:** For each neighbor $q'_m$, the input features are:
    *   Unsigned distance: $u'_m = f_{u_{gt}}(q'_m)$ (during prior training) or $f_u(q'_m)$ (during generalization).
    *   Sampling interval: $\delta_m$, the distance between $q'_m$ and $q'_{m+1}$ [S1].
4.  **Network Architecture $f_w$:**
    *   Three separate Multi-Layer Perceptrons (MLPs), denoted $f_{w1}^{(k)}$ for $k \in \{1,2,3\}$, process each group independently.
    *   Each $f_{w1}^{(k)}$ takes the concatenated features (distances and intervals) of its group as input.
    *   The parameters of the three $f_{w1}$ MLPs are **non-shared** [S1].
    *   The output feature vectors from the three groups are summed.
    *   The sum is passed to a final MLP $f_{w2}$ with skip connections to predict the opaque density $\sigma_n$ [S1].

### Numbered Steps

#### Phase 1: Prior Training
1.  **Dataset Preparation:** For each ground truth shape $S_h$ in the training set:
    *   Compute the ground truth UDF $f_{u_{gt}}$ for all points in space [S1].
    *   Sample $A=100$ random camera views around the shape [S1].
2.  **Rendering:** For each view $a$ of shape $h$:
    *   Shoot rays from the camera.
    *   Hierarchically sample $N=128$ points $\{q_n\}_{n=1}^N$ along each ray [S1].
    *   For each query $q_n$, retrieve the ground truth unsigned distances $\{f_{u_{gt}}(q'_m)\}_{m=1}^M$ of its $M$ nearest neighbors along the ray and the sampling intervals $\{\delta_m\}_{m=1}^M$ [S1].
    *   Compute opaque density $\sigma_n = f_w(\{\delta_m\}, \{f_{u_{gt}}(q'_m)\})$ using the current network $f_w$ [S1].
    *   Compute rendering weights $w_n = \sigma_n \times \prod_{n'=1}^{n-1} (1 - \sigma_{n'})$ [S1].
    *   Render the depth image $\tilde{D}_a^h = \sum_{n'=1}^N w_{n'} \times d_{n'}$, where $d_{n'}$ is the depth of point $q_{n'}$ [S1].
3.  **Optimization:** Update network parameters $\theta_w$ by minimizing the L2 error between the rendered depth $\tilde{D}_a^h$ and the ground truth depth $D_a^h$ derived from $f_{u_{gt}}$:
    $$ \min_{\theta_w} \sum_{h=1}^H \sum_{a=1}^A \| D_a^h - \tilde{D}_a^h \|_2^2 $$
    *   Note: RGB images are **not** used in this phase to prevent the color network from interfering with the geometric prior learning [S1].

#### Phase 2: Generalization (Unseen Scene)
1.  **Initialization:** Freeze $f_w$ parameters $\theta_w$. Initialize a UDF field $f_u$ (e.g., spherical initialization) and a color network $f_c$ [S1].
2.  **Sampling:** For each pixel in the target RGB images, shoot a ray. Hierarchically sample $N=128$ points $\{q_n\}_{n=1}^N$ along the ray [S1].
3.  **Density Prediction:** For each query $q_n$:
    *   Find $M$ nearest neighbors $\{q'_m\}_{m=1}^M$ along the ray.
    *   Retrieve current UDF values $\{f_u(q'_m)\}$ and sampling intervals $\{\delta_m\}$.
    *   Compute opaque density $\sigma_n = f_w(\{\delta_m\}, \{f_u(q'_m)\})$ using the frozen $f_w$ [S1].
4.  **Rendering:**
    *   Compute weights: $w_n = \sigma_n \times \prod_{n'=1}^{n-1} (1 - \sigma_{n'})$ [S1].
    *   Compute rendered color: $I' = \sum_{n'=1}^N w_{n'} \times f_c(q_{n'})$ [S1].
    *   Compute rendered depth: $D' = \sum_{n'=1}^N w_{n'} \times d_{n'}$ [S1].
5.  **Optimization:** Update $f_u$ and $f_c$ parameters by minimizing the total loss:
    $$ \mathcal{L} = \mathcal{L}_{rgb} + \lambda \mathcal{L}_e $$
    Where $\mathcal{L}_{rgb} = \sum_{j=1}^J \| I_j - \tilde{I}_j \|$ (photometric loss) and $\mathcal{L}_e$ is the Eikonal loss [S1].
6.  **Surface Extraction:** Extract the zero-level set of the optimized $f_u$ as the final mesh surface [S1].

### Formulas
*   **Opaque Density:**
    $$ \sigma_n = f_w(\{\delta_m\}_{m=1}^M, \{f_{u_{gt}}(q'_m)\}_{m=1}^M) $$
    where $q'_m \in NN(q_n)$ are the $M$ nearest neighbors of $q_n$ along the ray, and $\delta_m$ is the sampling interval [S1].
*   **Rendering Weight:**
    $$ w_n = \sigma_n \times \prod_{n'=1}^{n-1} (1 - \sigma_{n'}) $$
    [S1]
*   **Rendered Color:**
    $$ I' = \sum_{n'=1}^N w_{n'} \times f_c(q_{n'}) $$
    [S1]
*   **Rendered Depth:**
    $$ D' = \sum_{n'=1}^N w_{n'} \times d_{n'} $$
    [S1]
*   **Prior Training Loss:**
    $$ \min_{\theta_w} \sum_{h=1}^H \sum_{a=1}^A \| D_a^h - \tilde{D}_a^h \|_2^2 $$
    [S1]
*   **Generalization Loss:**
    $$ \mathcal{L} = \mathcal{L}_{rgb} + \lambda \mathcal{L}_e $$
    [S1]

### Parameters
*   $N=128$: Number of hierarchical samples along each ray [S1].
*   $A=100$: Number of ground truth depth images rendered per shape during prior training [S1].
*   $M_1=10, M_2=20, M_3=30$: Sizes of the three neighbor groups [S1].
*   $\lambda=0.1$: Balance weight for the Eikonal loss [S1].

### Edge Cases and Pitfalls
*   **Handcrafted Bias:** Standard methods using only $M=2$ neighbors are biased at ray-surface intersections and sensitive to unsigned distance outliers due to small receptive fields and lack of 3D awareness [S1].
*   **RGB Interference:** Using RGB images during the prior training phase reduces the generalization ability of the learned prior. The prior must be trained solely on depth errors to learn the UDF-to-density mapping [S1].
*   **Sampling Bias:** Hierarchical sampling can introduce bias in non-intersection regions; an auxiliary point sampling prior may be required to correct sampling density [S1].

## Operating characteristics
*   **Time Complexity:** The prior training phase requires rendering $H \times A$ depth images, which is computationally expensive but performed once. The generalization phase has similar complexity to standard NeRF/UDF methods but adds the overhead of querying the neighbor-based MLP $f_w$ for each sample.
*   **Memory Complexity:** Requires storing the parameters of $f_w$ (three $f_{w1}$ MLPs and one $f_{w2}$ MLP) and the UDF field $f_u$.
*   **Strengths:** Robust to unsigned distance outliers; 3D aware due to multi-scale neighborhood aggregation; adaptive to various UDF variants (e.g., sharp corners, large near-surface regions) [S1].
*   **Limitations:** Requires ground truth meshes for prior training; does not generalize to scenes without corresponding training shapes if the prior is not sufficiently diverse [S1].
*   **When to use:** When reconstructing surfaces from multi-view RGB images where standard UDF methods suffer from bias or noise, and ground truth meshes are available for prior training [S1].
*   **When not to use:** When real-time inference is critical and the overhead of the neighbor-based MLP is prohibitive, or when no ground truth meshes are available for prior training [S1].

## Background
VRPrior was introduced in the paper "VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors" [S1]. It addresses limitations in previous differentiable renderers that used handcrafted equations (e.g., NeUDF [2], 2S-UDF [28]) which were biased and not 3D aware. The method builds on the idea of learnable neural rendering frameworks [66–68] and hierarchical volume sampling [69]. It was developed to provide a more robust and scalable way to map UDFs to opaque densities for alpha blending [S1].

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
