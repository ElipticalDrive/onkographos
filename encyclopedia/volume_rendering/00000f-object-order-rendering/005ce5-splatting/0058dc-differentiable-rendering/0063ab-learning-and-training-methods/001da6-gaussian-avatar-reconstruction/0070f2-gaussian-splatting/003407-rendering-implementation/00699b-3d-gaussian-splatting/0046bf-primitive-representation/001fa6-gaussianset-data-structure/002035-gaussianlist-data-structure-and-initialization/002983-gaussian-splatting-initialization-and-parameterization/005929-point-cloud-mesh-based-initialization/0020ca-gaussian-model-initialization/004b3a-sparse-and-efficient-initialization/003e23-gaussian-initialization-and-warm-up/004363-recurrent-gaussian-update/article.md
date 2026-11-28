# Recurrent Gaussian Update

Also called: Feedback Loop Update, Learned rendering-feedback refinement, ReSplat (Recurrent Gaussian Splatting), Recurrent Gaussian Splating, Recurrent Refinement

## Overview
Recurrent Gaussian Update is a gradient-free, feed-forward refinement method that iteratively corrects initial 3D Gaussian predictions by predicting incremental updates to Gaussian parameters and hidden states based on rendering errors from input views [S1]. It solves the problem of inaccurate initial reconstructions in single-step feed-forward models by establishing a feedback loop that progressively refines the Gaussian set without requiring backpropagation or gradient computation [S1]. Unlike standard differentiable optimization which relies on iterative gradient descent, this method uses a learned recurrent network to map rendering errors directly to parameter updates, enabling convergence in a fixed number of iterations (typically 4) rather than thousands [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial set of 3D Gaussians $\mathcal{G}^0 = \{(\mathbf{g}_j^0, z_j^0)\}_{j=1}^M$ predicted by a feed-forward initialization network [S1]; a set of $N$ input views $\{I_i\}_{i=1}^N$ (ground truth images) [S1].
*   **Outputs:** Refined set of 3D Gaussians $\mathcal{G}^T = \{(\mathbf{g}_j^T, z_j^T)\}_{j=1}^M$ after $T$ iterations [S1].

**Data Structures:**
*   **Gaussian Parameters ($\mathbf{g}_j$):** Vector of dimension $C_1$ containing Gaussian attributes [S1].
*   **Hidden State ($z_j$):** Vector of dimension $C_0$ storing recurrent memory for Gaussian $j$ [S1].
*   **Rendering Error ($\hat{\mathbf{E}}^t$):** Tensor of shape $N \times \frac{H}{4} \times \frac{W}{4} \times C_2$, where $C_2=256$ [S1].
*   **Aggregated Error ($\mathbf{E}^t$):** Tensor of shape $M \times C_2$ after global attention aggregation [S1].

**Numbered Steps:**

1.  **Rendering:** Render the input views $\{\hat{I}_i^t\}_{i=1}^N$ using the current Gaussian parameters $\mathcal{G}^t$ at iteration $t$ [S1].
2.  **Pixel-Space Error Computation:** Compute pixel-wise difference $\{\hat{I}_i^t - I_i\}_{i=1}^N$ [S1]. Perform 4× spatial downsampling on the error using pixel unshuffle to align the spatial resolution with the number of 3D Gaussians [S1].
3.  **Feature-Space Error Computation:** Extract the first three stage features (at 1/2, 1/4, and 1/8 resolutions) from an ImageNet pre-trained ResNet-18 for both rendered views $\{\hat{I}_i^t\}$ and ground-truth views $\{I_i\}$ [S1]. Bilinearly resize the three scale features to the 1/4 resolution [S1]. Concatenate the resized features [S1]. Compute the difference: $\{\hat{F}_i^t - F_i\}_{i=1}^N$, where $\hat{F}_i^t, F_i \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times C_2}$ and $C_2 = 256$ [S1].
4.  **Error Combination:** Project the pixel-space error to the feature space using a linear layer followed by Layer Normalization [S1]. Add the projected pixel-space error to the feature-space error [S1]. Formula:
    $$ \hat{\mathbf{E}}^t = f \left( \{\hat{I}_i^t\}_{i=1}^N, \{I_i\}_{i=1}^N \right) = \{\hat{F}_i^t - F_i\}_{i=1}^N + \text{proj}(\{\hat{I}_i^t - I_i\}_{i=1}^N) $$
    where $\text{proj}$ denotes the linear projection and Layer Normalization [S1]. The resulting tensor $\hat{\mathbf{E}}^t$ contains $N \times \frac{H}{4} \times \frac{W}{4}$ error vectors $\hat{\mathbf{e}}_j^t \in \mathbb{R}^{C_2}$ [S1].
5.  **Error Propagation (Global Attention):** Apply global attention across all $N \times \frac{H}{4} \times \frac{W}{4}$ rendering errors in $\hat{\mathbf{E}}^t$ to aggregate global context [S1]. This allows each Gaussian to receive information from all rendering errors, overcoming the limitation of local spatial concatenation [S1]. Output: $\mathbf{E}^t = \text{global\_attention}(\hat{\mathbf{E}}^t) = \{\mathbf{e}_j^t\}_{j=1}^{M}$, where $\mathbf{e}_j^t$ is the aggregated error for Gaussian $j$ [S1].
6.  **Update Module Input:** Concatenate the Gaussian parameters $\mathbf{g}_j^t$, hidden state $z_j^t$, and aggregated error $\mathbf{e}_j^t$ for each Gaussian $j$ [S1]. Input set: $\{(\mathbf{g}_j^t, z_j^t, \mathbf{e}_j^t)\}_{j=1}^M$ [S1].
7.  **Update Prediction:** Pass the input set through four kNN attention blocks to model local structural details in 3D space [S1]. Decode the incremental updates $\Delta \mathbf{g}_j^t \in \mathbb{R}^{C_1}$ and $\Delta z_j^t \in \mathbb{R}^{C_0}$ using a lightweight four-layer MLP head [S1]. Formula:
    $$ \{(\mathbf{g}_j^t, z_j^t, \mathbf{e}_j^t)\}_{j=1}^M \rightarrow \{(\Delta \mathbf{g}_j^t, \Delta z_j^t)\}_{j=1}^M $$
    [S1].
8.  **Parameter Update:** Update the Gaussian parameters and hidden state:
    $$ \mathbf{g}_j^{t+1} = \mathbf{g}_j^t + \Delta \mathbf{g}_j^t, \quad z_j^{t+1} = z_j^t + \Delta z_j^t $$
    [S1].
9.  **Iteration:** Repeat steps 1–8 for $T$ iterations [S1]. During training, $T$ is randomly sampled between 1 and 4 [S1]. During inference, $T$ is flexible, allowing a speed-accuracy trade-off; convergence is typically observed after 4 iterations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recurrent_gaussian_update(initial_gaussians, input_views, cameras, num_iterations):
    """
    Refines initial 3D Gaussians using recurrent error feedback.
    
    Args:
        initial_gaussians: List of tuples (g_j, z_j) for j=1..M. 
                           g_j is parameter vector (dim C1), z_j is hidden state (dim C0).
        input_views: List of ground truth images I_i for i=1..N.
        cameras: List of camera poses corresponding to input_views.
        num_iterations: Number of refinement steps T.
        
    Returns:
        Refined list of tuples (g_j, z_j).
    """
    current_gaussians = list(initial_gaussians)
    M = len(current_gaussians)
    
    for t in range(num_iterations):
        # 1. Render current Gaussians to get synthetic views
        rendered_views = render_gaussians(current_gaussians, cameras)
        
        # 2. Compute Pixel-Space Error
        pixel_errors = [rendered_views[i] - input_views[i] for i in range(len(input_views))]
        downsampled_pixel_errors = [pixel_unshuffle(err, factor=4) for err in pixel_errors]
        
        # 3. Compute Feature-Space Error
        rendered_features = [extract_resnet_features(rendered_views[i]) for i in range(len(input_views))]
        gt_features = [extract_resnet_features(input_views[i]) for i in range(len(input_views))]
        resized_rendered_features = [concatenate_resize(feat, target_res='1/4') for feat in rendered_features]
        resized_gt_features = [concatenate_resize(feat, target_res='1/4') for feat in gt_features]
        feature_errors = [resized_rendered_features[i] - resized_gt_features[i] for i in range(len(input_views))]
        
        # 4. Combine Errors
        projected_pixel_errors = [layer_norm(linear_proj(err)) for err in downsampled_pixel_errors]
        combined_errors = [feature_errors[i] + projected_pixel_errors[i] for i in range(len(input_views))]
        flat_errors = [err.flatten() for err in combined_errors]
        
        # 5. Global Attention Propagation
        aggregated_errors = global_attention(flat_errors, num_gaussians=M)
        
        # 6. Prepare Update Input
        update_inputs = [(current_gaussians[j][0], current_gaussians[j][1], aggregated_errors[j]) for j in range(M)]
            
        # 7. Predict Updates using kNN Attention Blocks and MLP Head
        deltas = update_module_knn_mlp(update_inputs)
        
        # 8. Update Parameters
        new_gaussians = []
        for j in range(M):
            g_j, z_j = current_gaussians[j]
            delta_g, delta_z = deltas[j]
            new_gaussians.append((g_j + delta_g, z_j + delta_z))
            
        current_gaussians = new_gaussians
        
    return current_gaussians

# Standard operations / Neural network components (implementation details omitted as per standard libraries)
def render_gaussians(gaussians, cameras): ...
def pixel_unshuffle(image, factor): ...
def extract_resnet_features(image): ...
def concatenate_resize(features_list, target_res): ...
def linear_proj(tensor): ...
def layer_norm(tensor): ...
def global_attention(errors, num_gaussians): ...
def update_module_knn_mlp(inputs): ...
```

**Parameters:**
*   $C_2$: Channel dimension of the feature-space error. Value: 256 [S1].
*   $T$: Total number of iterations. Training: Randomly sampled between 1 and 4. Inference: Flexible, allows speed-accuracy trade-off. Convergence: Observed after 4 iterations [S1].
*   $k$: Number of neighbors for kNN attention in the recurrent model. Value: 8 [S1]. (Note: $k=16$ is used in the *initialization* model, but 8 in the recurrent model) [S1].
*   Spatial subsampling factor: 4× in 3D space for 2-view, 256×256 resolution setups [S1].
*   Gaussians decoded per subsampled 3D point: 4 [S1].

**Edge Cases and Pitfalls:**
*   **kNN Bottleneck:** The kNN operation during reconstruction is a primary bottleneck for inference speed and requires implementation-level optimization for faster deployment [S1].
*   **Global Attention Cost:** Performing global attention on $N \times \frac{H}{4} \times \frac{W}{4}$ features can be expensive for high-resolution images; efficient implementation is required [S1].
*   **Convergence:** The model converges after 4 iterations; fewer iterations may result in suboptimal quality, while more iterations yield diminishing returns [S1].

## Subtopics

### [Recurrent Mean-Shift Update](16677-recurrent-mean-shift-update.md)
The Recurrent Mean-Shift Update is a regularization step applied to the descriptor parameters of 3D Gaussians to enforce holistic separability across the entire scene. It addresses the problem where local segmentation losses only distinguish descriptors within individual image views, failing to ensure that descriptors for the same 3D plane instance remain consistent across different camera viewpoints. By applying a mean-shift operation based on the von Mises-Fisher distribution, it pulls similar descriptors together and pushes dissimilar ones apart globally, improving plane instance segmentation. It differs from standard differentiable updates by operating on the statistical distribution of descriptors rather than individual pixel-wise gradients.

## Operating characteristics
*   **Time Complexity:** Significantly faster than gradient-based optimization. ReSplat is 100× faster than 3DGS optimization on 8-view DL3DV [S1]. Reconstruction time is 1.2–1.7 seconds for 1–2 iterations on 16-view DL3DV [S1].
*   **Memory Complexity:** Compresses the number of Gaussians by 16× compared to per-pixel feed-forward models (e.g., MVSplat, DepthSplat), resulting in 4× faster rendering speed [S1]. Uses 246K Gaussians for 8-view DL3DV and 518K for 16-view DL3DV [S1].
*   **Measured Performance:**
    *   **8-view (512×960) DL3DV:** 246K Gaussians, 100× faster than 3DGS optimization, 4× faster rendering than per-pixel models [S1]. Outperforms 3DGS by 4.2dB PSNR and DepthSplat by 3.5dB PSNR [S1].
    *   **16-view (540×960) DL3DV:** 518K Gaussians, reconstruction time 1.2–1.7 sec for 1–2 iterations [S1]. Outperforms Long-LRM (518K Gaussians) with 1 iteration (23.23 PSNR vs 22.69 PSNR) [S1].
    *   **2-view (256×256) RealEstate10K:** Outperforms LVSM encoder-decoder by 1.1 dB PSNR, 20× faster rendering than 3DGS-free methods [S1].
*   **Scaling:** Supports flexible speed-accuracy trade-off at inference by varying the number of iterations $T$ [S1].
*   **Strengths:** Gradient-free feed-forward architecture eliminates gradient computation, providing a 53× speedup in reconstruction time compared to 3DGS optimization-based refinement [S1]. Converges in 4 iterations vs 30,000 for 3DGS [S1]. More parameter-efficient than single-step models; ReSplat-Small (77M params) outperforms ReSplat-Large (559M params) and WorldMirror (1263M params) after 4 iterations [S1].
*   **Limitations:** kNN operation slows down inference speed [S1]. Global attention can be computationally expensive for high-resolution images [S1].
*   **When to use:** When fast, high-quality view synthesis is required from a small number of input views, and gradient-based optimization is too slow [S1].
*   **When not to use:** When extremely low latency is required and even the feed-forward inference time is prohibitive, or when the scene requires fine-grained optimization that a fixed number of recurrent steps cannot capture [S1].

## Background
Recurrent Gaussian Update was introduced in the paper "ReSplat: Learning Recurrent Gaussian Splatting" [S1]. It builds on the concept of feed-forward Gaussian splatting initialization [S1]. The method was developed to address the limitations of single-step feed-forward models, which lack the refinement capability of iterative optimization, while avoiding the computational cost of gradient-based methods [S1]. It is evaluated on DL3DV, RealEstate10K, and ACID datasets [S1].

## Sources
[S1] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
