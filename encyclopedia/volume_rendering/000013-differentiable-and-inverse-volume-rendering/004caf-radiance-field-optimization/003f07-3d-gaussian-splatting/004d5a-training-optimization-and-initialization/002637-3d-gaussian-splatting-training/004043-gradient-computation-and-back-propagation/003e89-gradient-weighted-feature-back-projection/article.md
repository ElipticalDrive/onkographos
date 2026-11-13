# Gradient-Weighted Feature Back-Projection

Also called: Back-Projection Equation, Expected Feature Back-Projection, FBP, Feature Back-Projection

## Overview
Gradient-Weighted Feature Back-Projection is a training-free method for aggregating 2D feature embeddings from training views into 3D Gaussian primitives in 3D Gaussian Splatting (3DGS) [S1]. It solves the problem of feature field rendering by back-projecting features based on each Gaussian's gradient-based influence (opacity and transmittance) on the rendered pixel, avoiding the need for distillation or explicit feature field training [S1]. It is set apart from training-based alternatives like Feature 3DGS by being 10x faster and from masked-gradient methods by providing dense, normalized feature vectors suitable for 3D segmentation and affordance transfer [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of pre-trained 3D Gaussians (position, scale, rotation, opacity, color), a set of $V$ training views with corresponding ground-truth 2D feature maps $\mathbf{F}_{2D}$ (e.g., from LSeg or DINOv2), and camera parameters for each view.
*   **Outputs:** A feature vector $\mathbf{f}_k \in \mathbb{R}^D$ for each Gaussian $k$, where $D$ is the feature dimension.

**Data Structures:**
*   **Gaussians:** Stored as a list or array of primitives. Each primitive $k$ has attributes: position $\mathbf{p}_k$, scale $\mathbf{s}_k$, rotation $\mathbf{R}_k$, opacity $\sigma_k$, color $\mathbf{c}_k$, and the computed feature $\mathbf{f}_k$.
*   **Feature Maps:** For each view $n$, a 2D tensor $\mathbf{F}_{2D}^{(n)}$ of size $H \times W \times D$.

**Numbered Steps:**
1.  **Render and Collect:** For each training view $n = 1 \dots V$:
    a. Render the scene using the current Gaussian parameters to obtain the 2D feature map $\mathbf{F}_{2D}^{(n)}$ and the depth/alpha information. This is typically done using a differentiable rasterizer like `gsplat` [S1].
    b. For each pixel $(x, y)$ in the view, identify the sorted list of Gaussians contributing to the pixel.
    c. For each contributing Gaussian $k$ at pixel $(x, y)$, compute its influence weight $w_{k,x,y}^{(n)} = \alpha_k(x, y) T_k(x, y)$, where $\alpha_k$ is the opacity and $T_k$ is the transmittance.
    d. Accumulate the weighted features and weights for each Gaussian $k$:
       $$ \text{num}_k^{(n)} = \sum_{(x,y)} \mathbf{F}_{2D}^{(n)}(x, y) \cdot \alpha_k(x, y) T_k(x, y) $$
       $$ \text{den}_k^{(n)} = \sum_{(x,y)} \alpha_k(x, y) T_k(x, y) $$
2.  **Aggregate Across Views:** Sum the accumulators across all views $n$:
    $$ \text{Num}_k = \sum_{n=1}^{V} \text{num}_k^{(n)} $$
    $$ \text{Den}_k = \sum_{n=1}^{V} \text{den}_k^{(n)} $$
3.  **Prune:** Identify Gaussians where $\text{Num}_k = \mathbf{0}$ (zero vector). These Gaussians have no influence on any screen pixel and should be pruned to avoid division by zero or irrelevant data [S1].
4.  **Normalize:** Compute the final feature vector for each remaining Gaussian $k$:
    $$ \mathbf{f}_k = \frac{\text{Num}_k}{\text{Den}_k} $$
    This is the **Expected Feature Back-Projection** [S1].

**Formulas:**
The rendered color $C(x, y)$ is a weighted sum:
$$ C(x, y) = \sum_{n \leq N} c_n \alpha_n(x, y) T_n(x, y) $$
where $N$ is the total number of Gaussians, $c_n$ is the color, $\alpha_n(x, y)$ is the opacity, and $T_n(x, y) = \prod_{m < n} (1 - \alpha_m(x, y))$ is the transmittance [S1].

The gradient of the rendered color with respect to the $k$-th Gaussian's color $c_k$ is:
$$ \frac{\partial C(x, y)}{\partial c_k} = \alpha_k(x, y) T_k(x, y) $$
This gradient serves as the weight for back-projection [S1].

The expected feature $\mathbf{f}_k$ for the $k$-th Gaussian is:
$$ \mathbf{f}_k = \frac{\sum_{(x,y,n)} \mathbf{F}_{2D}(x,y,n) \alpha_k(x,y,n) T_k(x,y,n)}{\sum_{(x,y,n)} \alpha_k(x,y,n) T_k(x,y,n)} $$
where $\mathbf{F}_{2D}(x, y, n)$ is the feature at $(x, y)$ in the $n$-th viewpoint [S1].

**Alternative: Accumulated Back-Projection:**
Removing the denominator yields:
$$ \mathbf{f}_k = \sum_{(x,y,n)} \mathbf{F}_{2D}(x, y, n) \alpha_k(x, y, n) T_k(x, y, n) $$
If Euclidean normalization is applied to $\mathbf{f}_k$ after calculation, this variant becomes equivalent to the expected feature back-projection [S1].

**Parameters:**
*   **Feature Extractor:** LSeg or DINOv2 are used for generating $\mathbf{F}_{2D}$ [S1].
*   **Rasterizer:** `gsplat` is used for rendering [S1].
*   **Threshold ($\theta$):** Used in segmentation (see Subtopics) to determine if a Gaussian matches a query. Not given in the sources for the back-projection itself.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gradient_weighted_feature_back_projection(gaussians, views, feature_extractor, rasterizer):
    """
    gaussians: List of Gaussian objects with attributes:
               pos, scale, rot, opacity, color, feature (initialized to 0)
    views: List of view dictionaries with keys:
           'image', 'camera_params', 'gt_features' (H, W, D tensor)
    feature_extractor: Function to generate gt_features from image if not provided
    rasterizer: Function to render scene and return alpha, transmittance, and hit_gaussians
                Returns: alpha_map (H, W, K), transmittance_map (H, W, K), hit_indices (H, W, K)
                where K is max depth
    """
    # Initialize accumulators
    # num_accum[k] is a vector of size D
    # den_accum[k] is a scalar
    num_accum = np.zeros((len(gaussians), feature_extractor.output_dim))
    den_accum = np.zeros(len(gaussians))
    
    # Step 1: Render and Collect for each view
    for view in views:
        # Render scene to get alpha and transmittance for all Gaussians at each pixel
        # Note: rasterizer must return contributions per Gaussian per pixel
        alpha_map, transmittance_map, hit_indices = rasterizer.render(gaussians, view['camera_params'])
        
        # Get or compute ground truth features for this view
        if 'gt_features' in view:
            F_2D = view['gt_features']
        else:
            F_2D = feature_extractor(view['image'])
            
        H, W, D = F_2D.shape
        
        # Iterate over pixels
        for y in range(H):
            for x in range(W):
                # Get sorted list of Gaussians contributing to this pixel
                # hit_indices[y, x] contains indices of Gaussians sorted by depth
                # alpha_map[y, x, i] is opacity of i-th Gaussian at (x,y)
                # transmittance_map[y, x, i] is transmittance before i-th Gaussian
                
                # We only need to iterate over Gaussians that actually hit this pixel
                # Assuming hit_indices contains valid indices and -1 for empty slots
                for i in range(K): # K is max depth
                    g_idx = hit_indices[y, x, i]
                    if g_idx == -1:
                        break
                    
                    alpha = alpha_map[y, x, i]
                    transmittance = transmittance_map[y, x, i]
                    
                    # Weight is alpha * transmittance
                    weight = alpha * transmittance
                    
                    # Feature at this pixel
                    f_pixel = F_2D[y, x]
                    
                    # Accumulate
                    num_accum[g_idx] += f_pixel * weight
                    den_accum[g_idx] += weight
                    
    # Step 2: Prune and Normalize
    final_gaussians = []
    for k, g in enumerate(gaussians):
        if np.allclose(num_accum[k], 0):
            # Prune Gaussians with zero influence
            continue
        
        # Compute expected feature
        if den_accum[k] > 0:
            g.feature = num_accum[k] / den_accum[k]
        else:
            # Fallback if denominator is zero but numerator is not (should not happen if logic is correct)
            g.feature = num_accum[k]
            
        final_gaussians.append(g)
        
    return final_gaussians
```

**Edge Cases and Pitfalls:**
*   **Zero Numerator:** If the numerator in the back-projection equation is zero (no Gaussian influence on screen), the Gaussian should be pruned before inference to avoid division by zero or irrelevant data [S1].
*   **Precision:** The method relies on accurate alpha and transmittance computation. Half-precision arithmetic may be used for optimization but can introduce precision errors in gradient computation [S1].

## Subtopics

### 3D Segmentation via Back-Projection
A direct application of the back-projected features. It obtains feature vectors $\{\mathbf{f}_k\} \in \mathbb{R}^D$ for each Gaussian after back-projection, extracts a query embedding $\mathbf{q} \in \mathbb{R}^D$ from a language query or another modality, and computes similarity $sim(\mathbf{f}_k, \mathbf{q})$ (typically cosine similarity) between each Gaussian feature and the query [S1]. Segmented Gaussians $G$ are defined by thresholding similarity: $G = \{g_k \mid \text{sim}(\mathbf{f}_k, \mathbf{q}) > \theta\}$, where $\theta$ is a user-defined threshold [S1]. The 2D segmentation mask $M(x, y)$ is defined as $1$ if $\text{sim}(\mathbf{f}_k, \mathbf{q}) > \theta$, else $0$ [S1].

### Identity Encoding
Encoding each object with a unique vector and back-projecting these vectors into the 3D scene, generalizing across the entire scene given sufficient annotated views [S1]. Two variants exist:
1.  **Orthogonal Encoding:** Assigns procedurally generated orthogonal vectors to represent each object uniquely. This maximizes separation and simplifies segmentation but is limited by the embedding dimension $D$ [S1].
2.  **Contrastive Encoding:** Uses non-orthogonal vectors made distinct through contrastive learning, which maximizes distance between different objects and minimizes distance for same objects [S1].
The loss function for the Identity Encoder-Decoder is $\mathcal{L} = \mathcal{L}_{\text{classification}} + \mathcal{L}_{\text{orthogonality}}$, where $\mathcal{L}_{\text{classification}} = \text{CrossEntropy}(\hat{y}, y)$ and $\mathcal{L}_{\text{orthogonality}} = \|\mathbf{E}\mathbf{E}^T - \mathbf{I}\|_F$. Here $\hat{y}$ is predicted class probability, $y$ is true label, $\mathbf{E}$ is embedding matrix of size $N \times D$ ($N$ classes, $D$ dimension), $\mathbf{I}$ is identity matrix, and $\| \cdot \|_F$ is Frobenius norm [S1].

### Affordance Transfer
Transfers DINOv2 features to target scenes via feature back-projection. It identifies the closest matching source annotation for each Gaussian in the target scene by performing k-nearest neighbors (kNN) classification on the feature space [S1].

## Operating characteristics
*   **Time Complexity:** The method requires a single pass through all training views for back-projection. The inference time for object segmentation (including text encoding and similarity calculation) is 30 ms [S1]. Similarity calculation takes less than 1 ms [S1].
*   **Memory Complexity:** Requires storing feature maps for all training views and accumulators for all Gaussians. The method avoids training a feature field, thus reducing memory overhead compared to training-based methods [S1].
*   **Measured Performance:** On an NVIDIA A6000 GPU, the feature back-projection process (including ground truth feature map generation) takes an average of 2-3 minutes [S1]. It achieves a 900x speedup compared to segmentation using masked gradients [7] for the full pipeline including mask generation [S1]. Compared to Feature 3DGS, this method is 10x faster (2.5 mins vs 20-30 mins for 7000 iterations) because it does not train a feature field [S1].
*   **Strengths:** Training-free, fast, scalable, high-quality 3D segmentation results [S1].
*   **Limitations:** Requires pre-trained Gaussians and ground-truth feature maps for training views. Performance depends on the quality of the 2D feature extractor [S1].
*   **When to use:** 3D segmentation, affordance transfer, and identity encoding in 3D Gaussian Splatting [S1].
*   **When not to use:** When ground-truth feature maps are unavailable or when real-time training of the feature field is required.

## Background
The method was introduced by Bharadwaj Amrutur and Shalabh Bhatnagar from the Indian Institute of Science [S1]. It leverages the insight that the gradient of the rendered color with respect to a Gaussian's color is equivalent to its weight in the rendering equation, allowing for efficient inference-time gradient backpropagation to compute feature back-projections [S1]. It is used for 3D segmentation, affordance transfer, and identity encoding [S1].

## Sources
[S1] Gradient-Weighted Feature Back-Projection: A Fast Alternative to Feature Distillation in 3D Gaussian Splatting, https://doi.org/10.1145/3757377.3763926, fetched 2026-10-27
