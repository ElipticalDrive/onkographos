# F2-NeRF Training Loss

Also called: F2-NeRF Loss, F2-NeRF Loss Function, F2-NeRF Regularization, F2-NeRF training objective

## Overview
The F2-NeRF training loss is a composite objective function for optimizing neural radiance fields with multiresolution hash grids, combining color reconstruction with disparity and total variation regularization [S1]. It specifically penalizes excessive inverse depth estimates to reduce floating artifacts and enforces feature consistency across octree node boundaries [S1].

## How it works
The method computes the total loss as a weighted sum of three components: color reconstruction, disparity regularization, and total variation regularization [S1].

### Inputs and Outputs
**Inputs:**
*   $r$: A camera ray.
*   $c(r)$: The rendered color for ray $r$, computed via volume rendering.
*   $c_{\text{gt}}$: The ground-truth color for ray $r$.
*   $\{t_i\}$: Sampled inverse distances (disparities) along the ray.
*   $\{w_i\}$: Alpha weights computed during volume rendering for each sample $i$.
*   $\text{feat}_0^k, \text{feat}_1^k$: Feature vectors fetched from the hash table for border points.

**Outputs:**
*   $\mathcal{L}$: The scalar training loss value.

### Data Structures
The implementation relies on the multiresolution hash grid structure [S1]. Specifically:
*   **Hash Table:** Stores feature vectors. In F2-NeRF, this is configured with 16 levels, where each level contains $2^{19}$ feature vectors of dimension 2 [S1].
*   **Hash Functions:** Two distinct hash functions are used to fetch features for neighboring octree nodes to compute the total variation loss [S1].

### Parameters
*   $\epsilon$: Numerical stability constant for reconstruction loss. Value: $10^{-4}$ [S1].
*   $\lambda_{\text{Disp}}$: Weight for disparity loss.
    *   LLFF dataset: $2.5 \times 10^{-4}$ [S1].
    *   Free and NeRF-360-V2 datasets: $10^{-3}$ [S1].
*   $\lambda_{\text{TV}}$: Weight for total variation loss.
    *   LLFF, Free, and NeRF-360-V2 datasets: $10^{-1}$ [S1].
*   $n_b$: Number of border points sampled for total variation loss. Value: 8192 [S1].
*   $n_r$: Number of rays processed in a batch.

### Formulas
The total loss is defined as [S1]:
$$
\mathcal{L} = \mathcal{L}_{\text{recon}}(c(r), c_{\text{gt}}) + \lambda_{\text{Disp}} \mathcal{L}_{\text{Disp}} + \lambda_{\text{TV}} \mathcal{L}_{\text{TV}}
$$

1.  **Reconstruction Loss** [S1]:
    $$
    \mathcal{L}_{\text{recon}}(c(r), c_{\text{gt}}) = \sqrt{(c(r) - c_{\text{gt}})^2 + \epsilon}
    $$

2.  **Disparity Loss** [S1]:
    $$
    \mathcal{L}_{\text{Disp}} = \frac{1}{n_r} \sum_{k=1}^{n_r} \text{disp}_k^2
    $$
    Where the disparity for ray $k$ is:
    $$
    \text{disp}_k = \sum_{i} w_i t_i
    $$
    Here, $t_i$ is the inverse distance (disparity) of the $i$-th sample, and $w_i$ is the corresponding volume rendering weight [S1].

3.  **Total Variation Loss** [S1]:
    $$
    \mathcal{L}_{\text{TV}} = \frac{1}{n_b} \sum_{k=1}^{n_b} \|\text{feat}_0^k - \text{feat}_1^k\|_2^2
    $$
    Where $\text{feat}_0^k$ and $\text{feat}_1^k$ are feature vectors fetched from the hash table using two different hash functions conditioned on the two neighboring octree nodes adjacent to the border point $k$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_f2_nerf_loss(rendered_color, ground_truth_color, 
                         weights, inverse_distances, 
                         hash_table, hash_func_0, hash_func_1,
                         epsilon=1e-4, lambda_disp=1e-3, lambda_tv=0.1):
    """
    Computes the F2-NeRF training loss.
    
    Args:
        rendered_color: List of lists or array of shape (N_rays, channels).
        ground_truth_color: List of lists or array of shape (N_rays, channels).
        weights: List of lists or array of shape (N_rays, N_samples). Alpha weights from volume rendering.
        inverse_distances: List of lists or array of shape (N_rays, N_samples). Sampled inverse depths t_i.
        hash_table: List or array storing feature vectors.
        hash_func_0: Callable returning hash index for first neighboring node.
        hash_func_1: Callable returning hash index for second neighboring node.
        epsilon: Numerical stability constant.
        lambda_disp: Weight for disparity loss.
        lambda_tv: Weight for total variation loss.
        
    Returns:
        total_loss: Scalar float.
    """
    n_rays = len(rendered_color)
    n_samples = len(weights[0])
    channels = len(rendered_color[0])

    # 1. Reconstruction Loss: sqrt((c(r) - c_gt)^2 + epsilon)
    recon_loss = 0.0
    for i in range(n_rays):
        diff_sq = sum((rendered_color[i][c] - ground_truth_color[i][c])**2 for c in range(channels))
        recon_loss += math.sqrt(diff_sq + epsilon)
    l_recon = recon_loss / n_rays

    # 2. Disparity Loss: mean_k (sum_i w_i * t_i)^2
    l_disp = 0.0
    for k in range(n_rays):
        disp_k = sum(weights[k][i] * inverse_distances[k][i] for i in range(n_samples))
        l_disp += disp_k ** 2
    l_disp /= n_rays

    # 3. Total Variation Loss: mean_k ||feat_0^k - feat_1^k||_2^2
    n_b = 8192
    # [G] Implementation detail: sample n_b points on octree node boundaries
    border_points = sample_border_points(n_b)
    tv_sum = 0.0
    for k in range(n_b):
        # [G] Implementation detail: determine two neighboring octree node IDs for point k
        node_id_0, node_id_1 = get_neighboring_node_ids(border_points[k])
        # Fetch features using two different hash functions conditioned on nodes
        feat_0 = hash_table[hash_func_0(node_id_0, border_points[k])]
        feat_1 = hash_table[hash_func_1(node_id_1, border_points[k])]
        # Compute squared L2 distance
        diff_sq = sum((feat_0[d] - feat_1[d])**2 for d in range(len(feat_0)))
        tv_sum += diff_sq
    l_tv = tv_sum / n_b

    return l_recon + lambda_disp * l_disp + lambda_tv * l_tv
```

### Edge Cases and Pitfalls
*   **Border Point Sampling:** The method requires randomly sampling points on the borders of octree nodes [S1]. If the octree structure is not explicitly maintained or if border detection is inaccurate, the loss cannot be computed [S1].
*   **Disparity Magnitude:** The disparity loss penalizes large inverse depths [S1]. In scenes with very deep geometry, this might bias the network towards shallower depths if $\lambda_{\text{Disp}}$ is too high [S1].
*   **Feature Dimension:** The hash features are small (dimension 2 in the reported implementation) [S1]. The L2 distance is computed on these low-dimensional vectors [S1].

## Operating characteristics
*   **Time Complexity:** The reconstruction and disparity losses are linear in the number of rays and samples [S1]. The total variation loss adds a cost proportional to $n_b$ (8192) hash lookups and vector subtractions per iteration, which is constant relative to the number of rays but adds a fixed overhead per training step [S1].
*   **Memory Complexity:** No additional large memory structures are required beyond the hash table itself [S1]. The border points and fetched features are transient [S1].
*   **Strengths:** Effectively reduces floating artifacts in unbounded scenes by constraining disparity [S1]. Enforces spatial smoothness in the hash grid features, leading to better generalization in free-trajectory settings [S1].
*   **Limitations:** Requires explicit access to octree node boundaries and hash functions [S1]. The fixed number of border points ($n_b=8192$) may not scale perfectly with very large or complex scenes without adjustment [S1].
*   **When to use:** Recommended for F2-NeRF and Instant-NGP-style architectures with multiresolution hash grids, especially in unbounded scenes with free camera trajectories [S1].

## Background
The F2-NeRF training loss was introduced in the paper "F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories" [S1]. It was designed to stabilize training and improve visual quality in unbounded scenes with free camera trajectories [S1]. The disparity loss and total variation loss were added to mitigate floating artifacts and enforce feature consistency across octree boundaries [S1].

## Sources
[S1] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
