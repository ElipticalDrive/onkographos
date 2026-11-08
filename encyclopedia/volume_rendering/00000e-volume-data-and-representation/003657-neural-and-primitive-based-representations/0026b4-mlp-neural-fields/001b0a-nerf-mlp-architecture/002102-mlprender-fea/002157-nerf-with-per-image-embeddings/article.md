# NeRF with Per-Image Embeddings

Also called: GLO Model Variant, NeRF with Global Lighting Optimization Embeddings

## Overview
This method extends the standard NeRF MLP architecture by learning a distinct per-image embedding vector for each scene, which is injected into the color prediction network to account for view-dependent photometric variations such as lighting changes or camera exposure differences [S1]. It sets itself apart from standard NeRF by explicitly modeling global appearance shifts across input views rather than treating them as noise, and differs from [SVS](link) by neutralizing this variation during training to produce a consistent, photometrically averaged reconstruction at test time [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $\{I_i\}_{i=1}^N$ with corresponding camera poses and intrinsics. Each image $I_i$ is associated with a learnable per-image embedding vector $e_i \in \mathbb{R}^d$.
*   **Outputs:** A radiance field that predicts volume density $\sigma$ and RGB color $C$ for any 3D point $x$ and view direction $v$.

**Data Structures:**
*   **Per-Image Embeddings:** A learnable parameter tensor $E \in \mathbb{R}^{N \times d}$, where $N$ is the number of training images and $d$ is the embedding dimension. The specific value of $d$ is not given in the sources [G: typical values in related literature are 16-64, but must be treated as a hyperparameter].
*   **NeRF MLP:** The color prediction network (see [MLPRender_Fea](2414-mlprender-fea.md)) takes positional encoding of position $x$, view direction $v$, and the per-image embedding $e_i$ as input.

**Numbered Steps:**
1.  **Initialization:** Initialize the per-image embeddings $E$ randomly. Initialize the NeRF MLP weights.
2.  **Forward Pass:** For a ray sampled from image $i$:
    a. Sample points $x_j$ along the ray.
    b. Compute volume density $\sigma_j$ and view direction $v_j$ for each point.
    c. Retrieve the per-image embedding $e_i$ corresponding to the source image $i$.
    d. Concatenate the positional encoding of $x_j$, the view direction $v_j$, and the embedding $e_i$.
    e. Pass the concatenated vector through the color prediction MLP to obtain RGB color $C_j$.
    f. Perform volume rendering to compute the final pixel color $C$ using standard alpha compositing.
3.  **Loss Computation:** Compute the photometric loss between the rendered image and the ground truth image $I_i$. The sources do not specify additional regularization terms for the embeddings [G: L2 regularization is common but not cited].
4.  **Optimization:** Update both the NeRF MLP weights and the per-image embeddings $E$ using gradient descent.
5.  **Test-Time Inference:** To generate a consistent reconstruction, set the embedding vector to zero ($e_i = \mathbf{0}$) for all rays [S1]. This yields a rendering corresponding to the photometric average of all input cameras, neutralizing the per-image lighting variations learned during training [S1].

**Formulas:**
The rendered color $C$ for a ray is computed as:
$$ C = \sum_{j=1}^T T_j \alpha_j C_j $$
where $T_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \delta_k)$ is the accumulated transmittance, $\alpha_j = 1 - \exp(-\sigma_j \delta_j)$ is the opacity, $\sigma_j$ is the volume density, $\delta_j$ is the distance between samples, and $C_j$ is the predicted color. The color $C_j$ is predicted by the MLP:
$$ C_j = \text{MLP}(\text{PE}(x_j), v_j, e_i) $$
where $\text{PE}(\cdot)$ is the positional encoding function [S1].

**Parameters:**
*   **Embedding Dimension ($d$):** Not given in the sources.
*   **Number of Images ($N$):** Determined by the dataset.

**Edge Cases and Pitfalls:**
*   **Test-Time Evaluation:** Evaluating the model with the learned embeddings $e_i$ active will result in images that match the specific lighting of the source image $i$, which may appear inconsistent across views if the goal is a canonical representation. Setting $e_i = \mathbf{0}$ is required for the "photometric average" behavior [S1].
*   **Photometric Variation:** Standard metrics like PSNR, SSIM, and LPIPS are difficult to interpret on datasets with significant photometric variation because they penalize global brightness/color shifts. The sources note that the "w/GLO" model performs poorly on these raw metrics compared to baselines that attempt to "explain away" variation, but produces visually consistent results [S1].

## Subtopics

### [Affine Generative Latent Optimization](3497-affine-generative-latent-optimization.md)
Affine Generative Latent Optimization (Affine GLO) is a technique for modulating the internal representations of a neural radiance field to capture per-image appearance variations, particularly for video sequences. It solves the problem of static NeRF models failing to represent dynamic lighting or texture changes across frames by applying an affine transformation to the bottleneck vector before view-dependent color prediction. It sets itself apart from standard [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) by learning a parametric mapping (an MLP) from a latent code to the affine parameters, rather than optimizing the embedding vectors directly, which allows for faster training and smoother interpolation.

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Adds $O(N \cdot d)$ memory for the per-image embeddings, where $N$ is the number of training images and $d$ is the embedding dimension. This is negligible compared to the MLP weights for typical $N$ and $d$ [G].
*   **Time:** Adds minimal overhead per training step for embedding lookup and concatenation. The optimization involves updating $N$ additional vectors, which scales linearly with $N$ but is generally fast [G].

**Measured Performance:**
On the Tanks and Temples dataset with color-corrected metrics (quadratic polynomial fit to RGB values, ignoring saturated pixels), the model achieves the following PSNR scores [S1]:
*   Bicycle: 24.37
*   Flowers: 21.73
*   Outdoor garden: 26.98
*   Stump: 26.40
*   Treehill: 22.87
*   Room: 31.63
*   Counter: 29.55
*   Indoor kitchen: 32.23
*   Bonsai: 33.46

The variant "w/GLO" (using the per-image embeddings) achieves slightly lower PSNR on these specific metrics due to the photometric neutralization [S1]:
*   Bicycle: 23.95
*   Flowers: 21.60
*   Outdoor garden: 25.09
*   Stump: 25.98
*   Treehill: 21.99
*   Room: 28.24
*   Counter: 28.40
*   Indoor kitchen: 30.81
*   Bonsai: 30.27

**Strengths and Limitations:**
*   **Strengths:** Neutralizes photometric variation during training, yielding consistent global brightness and color across views [S1]. Produces pleasing reconstructions that correspond to the photometric average of all input cameras [S1].
*   **Limitations:** Raw PSNR/SSIM/LPIPS metrics may be lower than baselines that do not neutralize variation, making direct comparison on standard metrics misleading on photometrically varying datasets [S1].

**When to Use:**
Use when the input dataset exhibits significant per-image photometric variation (e.g., lighting changes, exposure differences) and the goal is to produce a consistent, canonical view synthesis rather than reproducing the exact lighting of each source image [S1].

## Background
This method was introduced in the context of the Mip-NeRF 360 paper, which addressed unbounded anti-aliased neural radiance fields [S1]. It was developed to handle scenes with significant photometric variation, a common issue in real-world datasets where camera settings or lighting conditions change between shots [S1]. The approach is distinct from Stable View Synthesis (SVS), which modifies scene brightness to explain camera variation, by instead learning explicit per-image embeddings to model these variations [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
