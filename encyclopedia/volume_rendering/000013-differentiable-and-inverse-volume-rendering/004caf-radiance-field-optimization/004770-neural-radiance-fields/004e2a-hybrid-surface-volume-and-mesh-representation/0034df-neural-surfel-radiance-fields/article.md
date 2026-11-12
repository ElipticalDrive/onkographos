# Neural Surfel Radiance Fields

Also called: SurfelNeRF

## Overview
Neural Surfel Radiance Fields (SurfelNeRF) is an online photorealistic reconstruction method that represents large-scale indoor scenes using a dynamic set of neural surfels, enabling efficient incremental updates and fast rendering without per-scene optimization [S1]. It distinguishes itself from sibling methods like [Unified Neural Implicit Surface and Radiance Field Rendering](9732-unified-neural-implicit-surface-and-radiance-field-rendering.md) by using explicit surfel primitives with learned neural features rather than implicit occupancy fields, and from [Structured Latent Codes Anchored to Mesh](12715-structured-latent-codes-anchored-to-mesh.md) by maintaining a point-cloud-like structure that allows for simpler, rasterization-based fusion and rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of input images $\{I_t\}$, corresponding camera parameters $c_t$, and estimated or sensor depth maps $D_t$.
*   **Outputs:** A global neural surfel radiance field $S_g$ and rendered photorealistic images for novel views.

**Data Structures:**
Each neural surfel $s_i$ in the global set $S_g$ stores the following attributes:
*   Position $\mathbf{p}^i \in \mathbb{R}^3$
*   Normal $\mathbf{n}^i \in \mathbb{R}^3$
*   Radius $r^i \in \mathbb{R}$
*   Weight $w^i \in \mathbb{R}$ (confidence based on depth measurement distance)
*   Neural features $\mathbf{f}^i \in \mathbb{R}^{32}$ (extracted from input images)

**Parameters:**
*   $K=8$: Number of top global surfels considered per pixel during association.
*   $\delta_{depth}=0.1$m: Threshold for merging surfels based on depth distance.
*   $M=80$: Maximum number of shading points (intersecting surfels) per pixel during rendering.
*   $\lambda=0.1$: Balancing weight for depth loss.
*   Image Resolution: $640 \times 480$.
*   Keyframe Sampling: 5% of the sequence.
*   Learning Rate: 0.001 (training), 0.0002 (fine-tuning).

**Step 1: Local Neural Surfel Radiance Field Construction**
For each input frame $I_t$ with camera $c_t$ and depth $D_t$:
1.  **Depth Refinement:** If $D_t$ is from a sensor or MVS, refine it using a U-Net structure to handle noise and incompleteness [S1].
2.  **Unprojection:** Unproject pixels in $I_t$ to 3D space using $c_t$ to obtain initial 3D positions.
3.  **Attribute Estimation:** Estimate normal, weight, and radius for each point. The weight $w_t^i$ measures confidence based on the distance of the depth measurement from the camera center. The radius $r_t^i$ represents the local support area [S1].
4.  **Neural Feature Extraction:** Extract multi-scale view-dependent features from $I_t$ using a modified MnasNet (MasNet) pretrained on ImageNet. Project these 2D pixel features to 3D surfel features $\mathbf{f}_t^{merge}$ using an MLP [S1].

**Step 2: Neural Surfel Association**
Integrate local surfels $\{s_t^i\}$ into the global set $S_g$:
1.  **Projection:** Project each global surfel $s_j \in S_g$ as an ellipse onto the image plane of the current view.
2.  **Rasterization:** For each pixel, obtain the top-$K$ global surfels that overlap with it. Discard surfels failing a normal filter [S1].
3.  **Matching:** Compare the depth distance between the local surfel and the top-$M$ global candidates (sorted near to far).
4.  **Decision:** If the minimum depth distance between a local surfel and its global candidates is $< \delta_{depth}$, mark it for **Merge**. Otherwise, mark it for **Add** [S1].

**Step 3: Neural Surfel Fusion**
1.  **Addition:** For surfels marked "Add", insert them directly into $S_g$ with their estimated attributes and features [S1].
2.  **Update (Merge):** For surfels marked "Merge", update the corresponding global surfel $s_j$ using a GRU network for features and weighted sums for geometry [S1].
    *   **Feature Update:**
        $$ \mathbf{f}_t^{corrs} = \mathtt{GRU}(\mathbf{f}_t^{merge}, \mathbf{f}_{t-1}^{corrs}) $$
        Where $\mathbf{f}_t^{merge}$ is the input feature and $\mathbf{f}_{t-1}^{corrs}$ is the hidden state (previous global feature). The GRU is implemented as:
        $$ \mathbf{z}_t = \sigma(M_z([\mathbf{f}_t^{merge}, \mathbf{f}_{t-1}^{corrs}])) $$
        $$ \mathbf{r}_t = \sigma(M_r([\mathbf{f}_t^{merge}, \mathbf{f}_{t-1}^{corrs}])) $$
        $$ \tilde{\mathbf{f}}_t^{corrs} = \tanh(M_t([\mathbf{r}_t * \mathbf{f}_{t-1}^{corrs}, \mathbf{f}_t^{merge}])) $$
        $$ \mathbf{f}_t^{corrs} = (1 - \mathbf{z}_t) * \mathbf{f}_{t-1}^{corrs} + \mathbf{z}_t * \tilde{\mathbf{f}}_t^{corrs} $$
        Where $M_z, M_r, M_t$ are one-layer MLPs followed by sigmoid, sigmoid, and tanh activations, respectively, and $[\cdot, \cdot]$ denotes concatenation [S1].
    *   **Geometry Update:**
        $$ \mathbf{p}_t^j = \frac{w_t^i \mathbf{p}_t^i + w_{t-1}^j \mathbf{p}_{t-1}^j}{w_t^i + w_{t-1}^j} $$
        $$ \mathbf{n}_t^j = \frac{w_t^i \mathbf{n}_t^i + w_{t-1}^j \mathbf{n}_{t-1}^j}{w_t^i + w_{t-1}^j} $$
        $$ r_t^j = \frac{w_t^i r_t^i + w_{t-1}^j r_{t-1}^j}{w_t^i + w_{t-1}^j} $$
        $$ w_t^j = w_t^i + w_{t-1}^j $$
        Where $i$ is the local surfel index and $j$ is the global surfel index [S1].

**Step 4: Rendering**
To render a target view with camera $c_{tar}$:
1.  **Rasterization:** Project all global surfels onto the target image plane. For each pixel $p_x$, gather the set of intersecting surfels $\{s_i\}_{i=1}^M$ (up to $M=80$) [S1].
2.  **Shading Point Intersection:** For each surfel $s_i$, compute the intersection point $\mathbf{x}_i$ between the ray from $c_{tar}$ through $p_x$ and the surfel surface.
3.  **Feature Interpolation:** Compute the interpolated feature $\mathbf{f}^i(\mathbf{x}_i)$ at $\mathbf{x}_i$:
    $$ \mathbf{f}^i(\mathbf{x}_i) = \frac{r^i - \|\mathbf{x}^i - \mathbf{p}^i\|}{r^i} \mathtt{F}(\mathbf{f}^i, \mathbf{d}, \mathbf{n}^i, w^i) $$
    Where $\mathbf{d}$ is the view direction. The network $\mathtt{F}$ is:
    $$ \mathtt{F}(\mathbf{f}^i, \mathbf{d}, \mathbf{n}^i, w^i) = F_f([\mathbf{f}^i, \gamma(\mathbf{d}), \gamma(w^i), \gamma(\mathbf{n}^i), \gamma(\mathbf{d}-\mathbf{n})]) $$
    $F_f$ is a two-layer MLP with ReLU activation and hidden size 256. $\gamma(\cdot)$ is positional embedding with max frequency 5 [S1].
4.  **Density and Radiance Prediction:** Predict volume density $\sigma_i$ and radiance $c_i$ using an MLP $\mathtt{Render}$:
    $$ \sigma_i, c_i = \mathtt{Render}(\mathbf{x}_i, \mathbf{f}^i(\mathbf{x}_i), \mathbf{d}) $$
5.  **Volume Rendering:** Compute pixel color $C(p_x)$:
    $$ C(p_x) = \sum_{i=1}^M \tau_i (1 - \exp(-\sigma_i \Delta_i)) c_i $$
    $$ \tau_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \Delta_j\right) $$
    Where $\Delta_j$ is the distance between adjacent shading points [S1].

**Training Loss:**
$$ \mathcal{L} = \mathcal{L}_{render} + \lambda \cdot \mathbbm{1}(\mathcal{L}_d) $$
$$ \mathcal{L}_{render} = \| C_{gt} - C_r \|_2^2 $$
$$ \mathcal{L}_d = \| D_i^{sen} \odot M_i - D^i \odot M_i \|_1 $$
Where $D_i^{sen}$ is sensor depth, $M_i$ is a binary mask for valid depths, $D^i$ is the refined depth output, and $\mathbbm{1}$ is an indicator function (1 if sensor depth is available) [S1].

**Pitfalls:**
*   Inaccurate or noisy depth estimates cause geometry discontinuities or missing parts in the surfel representation, leading to rendering artifacts [S1].
*   The method relies on depth refinement; without it, incomplete sensor depths result in missing geometry [S1].

## Operating characteristics
*   **Speed:** SurfelNeRF achieves a 10× speedup in training and inference compared to conventional NeRF methods. Rendering time is approximately 0.2s per image on a V100 GPU, compared to 30s for volume rendering [S1].
*   **Complexity:** The major computational cost is rasterization. Surfel construction and integration costs are negligible. The number of surfels grows sub-linearly with input frames due to fusion [S1].
*   **Memory:** Memory footprint grows linearly with the number of neural surfels. On ScanNet, the average number of rasterized surfels per pixel is 2.92, with a maximum of 10 [S1].
*   **Performance:** On ScanNet, SurfelNeRF achieves 23.82 PSNR in feedforward inference and 29.58 PSNR with per-scene fine-tuning [S1].
*   **Strengths:** Efficient online reconstruction, scalable fusion, and fast rendering via rasterization-guided volume rendering.
*   **Limitations:** Sensitive to depth quality; requires depth refinement for noisy/incomplete inputs. Performance drops slightly (0.5% PSNR) if depth resolution is halved [S1].

## Background
SurfelNeRF was introduced in 2023 by Gao et al. [S1]. It builds upon classical surfel-based geometric reconstruction methods [43] and neural radiance fields [22], aiming to combine the efficiency of explicit geometry with the appearance modeling of NeRFs. It is designed for online photorealistic reconstruction of large-scale indoor scenes, such as those in the ScanNet dataset [S1].

## Sources
[S1] SurfelNeRF: Neural Surfel Radiance Fields for Online Photorealistic Reconstruction of Indoor Scenes, https://doi.org/10.1109/cvpr52729.2023.00019, fetched 2026-10-26
