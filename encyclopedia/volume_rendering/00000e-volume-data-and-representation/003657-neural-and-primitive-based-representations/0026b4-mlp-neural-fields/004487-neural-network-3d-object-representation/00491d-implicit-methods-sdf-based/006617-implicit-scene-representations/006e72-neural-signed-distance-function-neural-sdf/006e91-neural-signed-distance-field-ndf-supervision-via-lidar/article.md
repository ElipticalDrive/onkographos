# Neural Signed Distance Field (NDF) Supervision via LiDAR
Also called: LiDAR-supervised Neural SDF

## Overview
This method trains a neural signed distance field (NDF) using direct 3D LiDAR point cloud data as geometric supervision, rather than relying solely on image-based photometric or depth losses. It differs from standard Neural SDF approaches [Neural Signed Distance Function (Neural SDF)](21608-neural-signed-distance-function-neural-sdf.md) by converting the continuous signed distance ground truth into a binary occupancy supervision signal via a spatially varying sigmoid, allowing the network to learn surface boundaries from sparse LiDAR returns.

## How it works
**Inputs and Outputs**
*   **Input:** A 3D LiDAR point $^L\mathbf{x}$, defined by its origin $^L\mathbf{o}$ and ray direction $^L\mathbf{d}$, such that $^L\mathbf{x} = ^L\mathbf{o} + t^L\mathbf{d}$, where $t$ is the distance parameter.
*   **Output:** The geometry neural network $f_S$ outputs a predicted signed distance $\hat{s}$ and a spatially varying scale factor $\hat{\beta}$ for any queried 3D point $\mathbf{x}$.

**Data Structures**
The geometry neural network $f_S$ is implemented using hash encoding [31] and tiny multilayer perceptrons (MLPs) to encode a scalable signed distance field. The specific hash table layout and MLP architecture details are not provided in the source text [S1].

**Algorithm Steps**
1.  **Ray Sampling:** For a given LiDAR point $^L\mathbf{x}$, uniformly sample $N_L$ points along the ray segment from the LiDAR origin $^L\mathbf{o}$ to the LiDAR point $^L\mathbf{x}$. Let the $k$-th sample point be $^L\mathbf{x}_k = ^L\mathbf{o} + t_k ^L\mathbf{d}$, where $t_k$ is the distance parameter for the sample.
2.  **Network Query:** For each sample point $^L\mathbf{x}_k$, query the geometry network to obtain the predicted signed distance $\hat{s}_k$ and the predicted scale factor $\hat{\beta}_k$:
    $$ \hat{s}_k, \hat{\beta}_k = f_S(^L\mathbf{x}_k) $$
3.  **Ground Truth Occupancy Calculation:** Compute the ray distance $s_k$ from the sample point to the LiDAR point:
    $$ s_k = t - t_k $$
    Transform this distance into a ground truth occupancy value $o_k$ using a sigmoid function with the *predicted* scale factor $\hat{\beta}_k$:
    $$ o_k = \Phi(-s_k, \hat{\beta}_k) $$
    where $\Phi(v, h) = (1 + \exp(-v/h))^{-1}$.
4.  **Predicted Occupancy Calculation:** Transform the predicted signed distance $\hat{s}_k$ into a predicted occupancy value $\hat{o}_k$ using the same sigmoid function and the predicted scale factor $\hat{\beta}_k$:
    $$ \hat{o}_k = \Phi(-\hat{s}_k, \hat{\beta}_k) $$
5.  **Loss Computation:** Compute the binary cross-entropy loss $\mathcal{L}_{sdf}$ over all $N_L$ sampled points on the ray:
    $$ \mathcal{L}_{sdf} = -\frac{1}{N_L} \sum_{k=1}^{N_L} \left[ o_k \log(\hat{o}_k) + (1 - o_k) \log(1 - \hat{o}_k) \right] $$

**Parameters**
*   $\hat{\beta}_k$: Spatially varying scale factor. It indicates the standard deviation in the Logistic function and controls the sharpness of the transition between occupied and free space [S1]. Typical values are not specified in the source text.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(v, h):
    """
    Computes the sigmoid function Phi(v, h) = (1 + exp(-v/h))^-1.
    """
    if h == 0:
        # Handle division by zero edge case; typically h > 0.
        # If h approaches 0, sigmoid becomes a step function.
        # [G] Guard against numerical instability.
        return 1.0 if v < 0 else 0.0
    return 1.0 / (1.0 + math.exp(-v / h))

def compute_lidar_sdf_loss(lidar_origin, lidar_direction, lidar_distance, 
                           geometry_network, num_samples):
    """
    Computes the LiDAR-supervised SDF loss for a single LiDAR ray.
    
    Args:
        lidar_origin: 3D vector ^L_o
        lidar_direction: 3D vector ^L_d (assumed normalized)
        lidar_distance: float t (distance from origin to LiDAR point)
        geometry_network: Function f_S(x) returning (s_hat, beta_hat)
        num_samples: int N_L
        
    Returns:
        float: The binary cross-entropy loss value.
    """
    total_loss = 0.0
    
    # Step 1: Uniformly sample points from origin to LiDAR point
    # t_k ranges from 0 to t
    for i in range(num_samples):
        # Uniform sampling
        t_k = (i + 0.5) * lidar_distance / num_samples
        
        # Calculate sample point position
        sample_point = lidar_origin + lidar_direction * t_k
        
        # Step 2: Query network
        s_hat, beta_hat = geometry_network(sample_point)
        
        # Step 3: Calculate ground truth occupancy
        # s_k is the distance from sample point to LiDAR point
        s_k = lidar_distance - t_k
        
        # Ground truth occupancy o_k uses the predicted beta_hat
        # Note: The source uses the predicted beta_hat for the ground truth 
        # transformation, which is unusual but explicitly stated.
        o_k = sigmoid(-s_k, beta_hat)
        
        # Step 4: Calculate predicted occupancy
        o_hat_k = sigmoid(-s_hat, beta_hat)
        
        # Step 5: Binary Cross-Entropy
        # Clamp values to avoid log(0)
        eps = 1e-7
        o_hat_k_clamped = max(min(o_hat_k, 1.0 - eps), eps)
        
        loss_term = o_k * math.log(o_hat_k_clamped) + \
                    (1.0 - o_k) * math.log(1.0 - o_hat_k_clamped)
        total_loss -= loss_term
        
    # Average over N_L
    loss = total_loss / num_samples
    return loss
```

**Edge Cases and Pitfalls**
*   **Scale Factor Zero:** If $\hat{\beta}_k$ is zero, the sigmoid function involves division by zero. The implementation must handle this, typically by treating it as a step function or adding a small epsilon [G].
*   **Logarithm Domain:** The predicted occupancy $\hat{o}_k$ must be clamped to $[\epsilon, 1-\epsilon]$ before taking the logarithm to prevent numerical instability [G].
*   **Sparse LiDAR:** Since supervision is derived from LiDAR points, regions not covered by LiDAR returns may lack direct geometric supervision, potentially leading to artifacts in free space or unobserved surfaces. The source mentions using a "visible-aware occupancy map" to handle unknown grids, but the specific algorithm for this is not detailed in the provided text [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of sampled points $N_L$ per LiDAR ray and the cost of the hash encoding and tiny MLP inference. Not given in the sources.
*   **Memory Complexity:** Depends on the hash encoding table size and MLP parameters. Not given in the sources.
*   **Strengths:** Directly leverages accurate 3D LiDAR measurements for geometric supervision, potentially improving accuracy in large-scale scenes compared to image-only supervision.
*   **Limitations:** Relies on LiDAR coverage; performance may degrade in areas with sparse or no LiDAR returns. The use of the predicted scale factor $\hat{\beta}_k$ for both ground truth and predicted occupancy transformation is a specific design choice that may affect gradient flow.
*   **When to use:** When high-precision 3D LiDAR data is available and the goal is to reconstruct geometry that aligns well with LiDAR measurements.
*   **When not to use:** When LiDAR data is unavailable or extremely sparse, or when computational resources are limited for hash encoding lookups.

## Background
This method originates from the work on "Neural Surface Reconstruction and Rendering for LiDAR-Visual Systems" [S1]. It builds on the concept of Neural Signed Distance Functions [Neural Signed Distance Function (Neural SDF)](21608-neural-signed-distance-function-neural-sdf.md) but introduces LiDAR-based supervision. The approach uses hash encoding [31] and tiny MLPs to create a scalable representation. It contrasts with existing volume rendering methods for SDFs [16, 17] that train via pure images.

## Sources
[S1] Neural Surface Reconstruction and Rendering for LiDAR-Visual Systems, https://arxiv.org/abs/2409.05310v1, fetched 2026-10-26
