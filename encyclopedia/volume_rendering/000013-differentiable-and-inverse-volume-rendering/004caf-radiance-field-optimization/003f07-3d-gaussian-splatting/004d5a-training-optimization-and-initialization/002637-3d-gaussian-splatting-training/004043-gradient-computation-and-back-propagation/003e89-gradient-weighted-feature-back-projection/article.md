# Gradient-Weighted Feature Back-Projection

Also called: Back-Projection Equation, Expected Feature Back-Projection, FBP, Feature Back-Projection

## Overview
Gradient-Weighted Feature Back-Projection is a training-free method for rendering feature fields in 3D Gaussian Splatting by aggregating 2D feature embeddings from training views into pre-trained 3D Gaussians [S1]. It solves the problem of assigning semantic or appearance features to 3D Gaussians without the computational cost and memory overhead of training a separate feature field [S1]. Unlike training-based alternatives like Feature 3DGS, which require 20–30 minutes for optimization, this method completes the aggregation in a single pass through training views, offering a 10x speedup and high-quality 3D segmentation results [S1]. It sets itself apart by using the rendering gradient (opacity-weighted transmittance) as the back-projection weight, effectively treating the gradient as a measure of Gaussian influence on the final pixel [S1].

## How it works
The method computes the feature $\mathbf{f}_k$ for each Gaussian $k$ by back-projecting 2D features $\mathbf{F}_{2D}$ observed in multiple training views, weighted by the Gaussian's contribution to the rendered image.

**Inputs and Outputs:**
*   **Inputs:**
    *   A set of pre-trained 3D Gaussians $\{g_k\}_{k=1}^N$ with parameters (mean, covariance, opacity, color).
    *   A set of $V$ training views, each providing a rendered 2D feature map $\mathbf{F}_{2D}(x, y, v)$ for $v \in \{1, \dots, V\}$. These features are typically extracted from a pre-trained model (e.g., LSeg [S1], DINOv2 [S1]) using the same camera poses and intrinsics as the training data.
    *   The rasterizer parameters (resolution, near/far planes).
*   **Outputs:**
    *   A feature vector $\mathbf{f}_k \in \mathbb{R}^D$ for each Gaussian $k$, where $D$ is the feature dimension.

**Data Structures:**
*   **Gaussian Buffer:** Stores $N$ Gaussians. Each Gaussian $k$ has a feature field $\mathbf{f}_k$ initialized to zero or a default value.
*   **Feature Map Buffer:** For each view $v$, a 2D array of shape $(H, W, D)$ storing $\mathbf{F}_{2D}(x, y, v)$.
*   **Accumulation Buffer:** A temporary buffer for each Gaussian to accumulate the numerator and denominator of the back-projection equation.

**Numbered Steps:**

1.  **Initialization:** Initialize the accumulation buffers for all $N$ Gaussians. For the expected feature variant, initialize `num_k = 0` and `den_k = 0` for each $k$. For the accumulated variant, initialize `num_k = 0`.
2.  **Iterate over Training Views:** For each training view $v$ from $1$ to $V$:
    a. **Render 2D Features:** Use the current Gaussian parameters to render the scene. Instead of color, render the 2D feature map $\mathbf{F}_{2D}(x, y, v)$ by treating the feature vector as the "color" in the alpha-compositing equation. Simultaneously, compute the opacity $\alpha_k(x, y, v)$ and transmittance $T_k(x, y, v)$ for each Gaussian $k$ contributing to each pixel $(x, y)$.
    b. **Compute Weights:** For each Gaussian $k$ and each pixel $(x, y)$ it contributes to, calculate the weight $w_{k,x,y,v} = \alpha_k(x, y, v) T_k(x, y, v)$. This weight is equivalent to $\frac{\partial C(x, y)}{\partial c_k}$ [S1].
    c. **Accumulate:** For each Gaussian $k$, update its buffers:
       $$ \text{num}_k \leftarrow \text{num}_k + \sum_{(x,y)} \mathbf{F}_{2D}(x, y, v) \cdot w_{k,x,y,v} $$
       $$ \text{den}_k \leftarrow \text{den}_k + \sum_{(x,y)} w_{k,x,y,v} $$
       (Note: For the accumulated variant, skip the `den_k` update).
3.  **Pruning:** Identify Gaussians where the numerator $\text{num}_k$ is zero (indicating no influence on any screen pixel). Prune these Gaussians from further inference to avoid division by zero or irrelevant data [S1].
4.  **Normalization (Expected Feature Variant):** For each remaining Gaussian $k$, compute the final feature:
    $$ \mathbf{f}_k = \frac{\text{num}_k}{\text{den}_k} $$
    If Euclidean normalization is desired, normalize $\mathbf{f}_k$ to unit length [S1].
5.  **Completion:** The process is complete after a single pass through all training views [S1].

**Formulas:**

The rendered color equation serves as the basis for the gradient weight [S1]:
$$ C(x, y) = \sum_{n \leq N} c_n \alpha_n(x, y) T_n(x, y) $$
where $T_n(x, y) = \prod_{m < n} (1 - \alpha_m(x, y))$ is the transmittance [S1].

The gradient of the rendered color with respect to the $k$-th Gaussian's color $c_k$ is:
$$ \frac{\partial C(x, y)}{\partial c_k} = \alpha_k(x, y) T_k(x, y) $$
This gradient serves as the weight for back-projection [S1].

The **Expected Feature Back-Projection** equation is:
$$ \mathbf{f}_k = \frac{\sum_{(x,y,n)} \mathbf{F}_{2D}(x,y,n) \alpha_k(x,y,n) T_k(x,y,n)}{\sum_{(x,y,n)} \alpha_k(x,y,n) T_k(x,y,n)} $$
where $\mathbf{F}_{2D}(x, y, n)$ is the feature at $(x, y)$ in the $n$-th viewpoint [S1].

The **Accumulated Back-Projection** variant removes the denominator:
$$ \mathbf{f}_k = \sum_{(x,y,n)} \mathbf{F}_{2D}(x, y, n) \alpha_k(x, y, n) T_k(x, y, n) $$
If Euclidean normalization is applied to $\mathbf{f}_k$ after calculation, Equations 4 and 5 become equivalent [S1].

**Parameters:**
*   **Feature Extractor:** LSeg [S1] or DINOv2 [S1] are used for generating $\mathbf{F}_{2D}$.
*   **Rasterizer:** gsplat [S1] is used for rendering.
*   **Threshold ($\theta$):** Used in segmentation tasks to filter Gaussians based on similarity to a query [S1]. Not given in the sources for a default value.

**Edge Cases and Pitfalls:**
*   **Zero Numerator:** If the numerator in the back-projection equation is zero (no Gaussian influence on the screen), the Gaussian should be pruned before inference to avoid division by zero or irrelevant data [S1].
*   **Precision:** The method relies on accurate alpha-compositing. Standard floating-point precision is assumed, but the use of `half2` buffers for accumulation is suggested by parent principles for memory/throughput alignment [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_weighted_feature_back_projection(gaussians, training_views, feature_extractor, rasterizer):
    """
    gaussians: List of Gaussian objects with properties: mean, cov, opacity, color.
    training_views: List of dicts containing 'image', 'camera_params', 'feature_map'.
    feature_extractor: Function to extract features from an image.
    rasterizer: Function to render Gaussians and return alpha, transmittance, and depth.
    
    Returns:
    features: List of feature vectors for each Gaussian.
    """
    N = len(gaussians)
    D = feature_extractor.get_dimension() # e.g., 768 for DINOv2
    
    # Initialize accumulation buffers
    # num_k: shape (N, D)
    # den_k: shape (N,)
    num_k = torch.zeros(N, D)
    den_k = torch.zeros(N)
    
    for view in training_views:
        # 1. Render the scene to get alpha and transmittance for each Gaussian per pixel
        # rasterizer returns: alphas (H, W, N), transmittances (H, W, N)
        # Note: The rasterizer must support returning per-Gaussian contribution weights
        alphas, transmittances = rasterizer.render(gaussians, view['camera_params'])
        
        # 2. Get 2D feature map for this view
        # If feature map is not pre-computed, extract it
        if 'feature_map' not in view:
            view['feature_map'] = feature_extractor(view['image'])
        F_2D = view['feature_map'] # Shape (H, W, D)
        
        # 3. Compute weights: w = alpha * transmittance
        # alphas: (H, W, N), transmittances: (H, W, N)
        weights = alphas * transmittances # Shape (H, W, N)
        
        # 4. Accumulate features
        # We need to sum over (x, y) for each Gaussian k
        # weights: (H, W, N), F_2D: (H, W, D)
        # Element-wise multiply and sum over H, W
        # Result: (N, D)
        weighted_features = weights.unsqueeze(-1) * F_2D.unsqueeze(-2) # (H, W, N, D)
        num_k += weighted_features.sum(dim=(0, 1)) # Sum over H, W
        
        # Denominator is sum of weights over H, W for each Gaussian
        den_k += weights.sum(dim=(0, 1)) # Sum over H, W
        
    # 5. Prune Gaussians with zero numerator
    # Avoid division by zero
    zero_mask = (num_k == 0).all(dim=1) # Shape (N,)
    valid_indices = torch.where(~zero_mask)[0]
    
    # 6. Normalize
    # Expected Feature Back-Projection
    features = torch.zeros(N, D)
    # Handle division safely
    safe_den_k = den_k.clone()
    safe_den_k[safe_den_k == 0] = 1.0 # Prevent div by zero, though pruned Gaussians are ignored
    
    features[valid_indices] = num_k[valid_indices] / safe_den_k[valid_indices].unsqueeze(1)
    
    # Optional: Euclidean normalization
    # features = F.normalize(features, p=2, dim=1)
    
    return features
```

## Subtopics

### 3D Segmentation via Back-Projection
A direct application of the back-projected features for semantic segmentation. It obtains feature vectors $\{\mathbf{f}_k\} \in \mathbb{R}^D$ for each Gaussian after back-projection, extracts a query embedding $\mathbf{q} \in \mathbb{R}^D$ from a language query or another modality, and computes similarity $sim(\mathbf{f}_k, \mathbf{q})$ (typically cosine similarity) [S1]. Segmented Gaussians $G$ are defined by thresholding similarity: $G = \{g_k \mid \text{sim}(\mathbf{f}_k, \mathbf{q}) > \theta\}$, where $\theta$ is a user-defined threshold [S1]. 2D segmentation mask $M(x, y)$ is defined as $1$ if $\text{sim}(\mathbf{f}_k, \mathbf{q}) > \theta$, else $0$ [S1].

### Identity Encoding
Encoding each object with a unique vector and back-projecting these vectors into the 3D scene, generalizing across the entire scene given sufficient annotated views [S1]. Two variants exist:
*   **Orthogonal Encoding:** Assigns procedurally generated orthogonal vectors to represent each object uniquely. This maximizes separation and simplifies segmentation but is limited by the embedding dimension $D$ [S1].
*   **Contrastive Encoding:** Uses non-orthogonal vectors made distinct through contrastive learning, which maximizes distance between different objects and minimizes distance for same objects. This allows encoding more objects than possible with strictly orthogonal vectors [S1].
The loss function for the Identity Encoder-Decoder is $\mathcal{L} = \mathcal{L}_{\text{classification}} + \mathcal{L}_{\text{orthogonality}}$, where $\mathcal{L}_{\text{classification}} = \text{CrossEntropy}(\hat{y}, y)$ and $\mathcal{L}_{\text{orthogonality}} = \|\mathbf{E}\mathbf{E}^T - \mathbf{I}\|_F$. Here $\hat{y}$ is predicted class probability, $y$ is true label, $\mathbf{E}$ is embedding matrix of size $N \times D$ ($N$ classes, $D$ dimension), $\mathbf{I}$ is identity matrix, and $\| \cdot \|_F$ is Frobenius norm [S1].

### Affordance Transfer
Transferring annotated regions from source images to a target object. It transfers DINOv2 features to target scenes via feature back-projection and identifies the closest matching source annotation for each Gaussian in the target scene by performing k-nearest neighbors (kNN) classification on the feature space [S1].

## Operating characteristics
*   **Time Complexity:** The method requires a single pass through all training views. The inference time for object segmentation (including text encoding and similarity calculation) is 30 ms [S1]. Similarity calculation takes less than 1 ms [S1]. The entire feature back-projection process, including ground truth feature map generation, takes an average of 2–3 minutes [S1].
*   **Memory Complexity:** Memory usage is dominated by the storage of 2D feature maps for all training views and the accumulation buffers for $N$ Gaussians. The method avoids training a separate feature field, reducing memory overhead compared to Feature 3DGS [S1].
*   **Measured Performance:** On an NVIDIA A6000 GPU using gsplat rasterizer, the method achieves a 900x speedup compared to segmentation using masked gradients [7] for the full pipeline including mask generation [S1]. It is 10x faster than Feature 3DGS (2.5 mins vs 20–30 mins for 7000 iterations) [S1].
*   **Strengths:** Training-free, fast, scalable, high-quality 3D segmentation, no need for distillation or feature field training [S1].
*   **Limitations:** Requires pre-trained Gaussians and training views with features. Performance depends on the quality of the 2D feature extractor (LSeg, DINOv2) [S1].
*   **When to use:** 3D segmentation, affordance transfer, and identity encoding in 3D Gaussian Splatting [S1].
*   **When not to use:** When no training views are available, or when real-time feature field training is required for dynamic scenes (though this method is inference-time, it assumes static pre-trained Gaussians) [S1].

## Background
Gradient-Weighted Feature Back-Projection was introduced by Bharadwaj Amrutur and Shalabh Bhatnagar from the Indian Institute of Science [S1]. The method leverages the differentiable nature of the alpha-compositing equation in 3D Gaussian Splatting to back-project 2D features into 3D Gaussians without additional training [S1]. It addresses the limitations of training-based feature field rendering methods, which often perform poorly at 3D segmentation without post-processing [S1]. The approach is related to "vote using masked gradients" [7] but generalizes it to continuous feature embeddings [S1].

## Sources
[S1] Gradient-Weighted Feature Back-Projection: A Fast Alternative to Feature Distillation in 3D Gaussian Splatting, https://doi.org/10.1145/3757377.3763926, fetched 2026-10-27
