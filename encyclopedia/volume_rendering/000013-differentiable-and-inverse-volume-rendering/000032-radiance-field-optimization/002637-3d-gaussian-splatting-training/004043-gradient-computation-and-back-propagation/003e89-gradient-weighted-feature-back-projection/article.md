# Gradient-Weighted Feature Back-Projection

Also called: Back-Projection Equation, Expected Feature Back-Projection, FBP, Feature Back-Projection

## Overview
Gradient-Weighted Feature Back-Projection is a training-free method for rendering feature fields in 3D Gaussian Splatting by aggregating 2D feature embeddings from training views into pre-trained 3D Gaussians [S1]. It solves the problem of assigning semantic or visual features to 3D Gaussians without the computational cost and memory overhead of training a separate feature field [S1]. Unlike training-based alternatives like Feature 3DGS, which require iterative optimization of feature attributes, this method computes features in a single pass using the gradient of the rendered color with respect to each Gaussian's color as a weighting factor [S1]. This approach achieves high-quality 3D segmentation and affordance transfer with significantly lower latency than training-based methods [S1].

## How it works
**Inputs:**
- A set of $N$ 3D Gaussians $\{g_k\}_{k=1}^N$ with parameters (position, covariance, opacity $\alpha$, color $c$) obtained from a standard 3DGS reconstruction.
- A set of $V$ training views, each providing a 2D feature map $\mathbf{F}_{2D}(x, y, v)$ for $v \in \{1, \dots, V\}$. The feature maps are typically generated using a pre-trained model such as LSeg [S1] or DINOv2 [S1].
- The rasterization order of Gaussians for each pixel.

**Outputs:**
- A feature vector $\mathbf{f}_k \in \mathbb{R}^D$ for each Gaussian $k$, where $D$ is the feature dimension.

**Data Structures:**
- Gaussian attributes: $\alpha_k(x, y)$ (opacity), $T_k(x, y)$ (transmittance).
- 2D Feature Map: $\mathbf{F}_{2D}(x, y, v)$, a tensor of shape $(H, W, D)$ for each view $v$.
- Accumulator: $\mathbf{f}_k$, initialized to zero vector.

**Formulas:**
The rendered color $C(x, y)$ at pixel $(x, y)$ is defined as [S1]:
$$ C(x, y) = \sum_{n \leq N} c_n \alpha_n(x, y) T_n(x, y) \quad (1) $$
where $T_n(x, y) = \prod_{m < n} (1 - \alpha_m(x, y))$ is the transmittance, representing the probability that the ray passes through all preceding Gaussians $m < n$ [S1].

The gradient of the rendered color with respect to the color $c_k$ of the $k$-th Gaussian serves as the weight for back-projection [S1]:
$$ \frac{\partial C(x, y)}{\partial c_k} = \alpha_k(x, y) T_k(x, y) \quad (2) $$

The **Expected Feature Back-Projection** computes the feature $\mathbf{f}_k$ as a weighted average [S1]:
$$ \mathbf{f}_k = \frac{\sum_{v=1}^{V} \sum_{(x,y) \in \text{pixels}_v} \mathbf{F}_{2D}(x, y, v) \alpha_k(x, y, v) T_k(x, y, v)}{\sum_{v=1}^{V} \sum_{(x,y) \in \text{pixels}_v} \alpha_k(x, y, v) T_k(x, y, v)} \quad (3) $$
where $\alpha_k(x, y, v)$ and $T_k(x, y, v)$ are the opacity and transmittance of Gaussian $k$ at pixel $(x, y)$ in view $v$.

The **Accumulated Feature Back-Projection** variant removes the denominator [S1]:
$$ \mathbf{f}_k = \sum_{v=1}^{V} \sum_{(x,y) \in \text{pixels}_v} \mathbf{F}_{2D}(x, y, v) \alpha_k(x, y, v) T_k(x, y, v) \quad (4) $$
Note: If Euclidean normalization is applied to $\mathbf{f}_k$ after calculation, Equations 3 and 4 become equivalent [S1].

**Steps:**
1. **Render Feature Maps:** For each training view $v$, render the scene using the current Gaussian parameters to obtain the 2D feature map $\mathbf{F}_{2D}(x, y, v)$ and the depth/alpha information required to compute $\alpha_k(x, y, v)$ and $T_k(x, y, v)$ for all visible Gaussians [S1].
2. **Initialize Accumulators:** For each Gaussian $k$, initialize the numerator accumulator $\mathbf{Num}_k = \mathbf{0}$ and the denominator accumulator $Den_k = 0$ [S1].
3. **Back-Project:** Iterate through each training view $v$ and each pixel $(x, y)$:
   - Identify the set of Gaussians $\mathcal{G}_{x,y,v}$ that contribute to pixel $(x, y)$ in view $v$ (i.e., those with $\alpha_k > 0$).
   - For each $k \in \mathcal{G}_{x,y,v}$:
     - Compute weight $w_k = \alpha_k(x, y, v) T_k(x, y, v)$ [S1].
     - Update accumulators:
       $$ \mathbf{Num}_k \leftarrow \mathbf{Num}_k + \mathbf{F}_{2D}(x, y, v) \cdot w_k $$
       $$ Den_k \leftarrow Den_k + w_k $$
4. **Prune Zero-Influence Gaussians:** If $Den_k = 0$ (meaning the Gaussian had no influence on any screen pixel, resulting in a zero numerator), prune Gaussian $k$ from the scene to avoid division by zero or irrelevant data during inference [S1].
5. **Finalize Features:** For each remaining Gaussian $k$:
   - If using Expected Feature Back-Projection: $\mathbf{f}_k = \mathbf{Num}_k / Den_k$ [S1].
   - If using Accumulated Feature Back-Projection: $\mathbf{f}_k = \mathbf{Num}_k$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_weighted_feature_back_projection(gaussians, training_views, feature_extractor):
    """
    gaussians: List of Gaussian objects with attributes:
               - id: unique identifier
               - alpha(x, y, view_idx): function returning opacity at pixel (x,y) in view
               - T(x, y, view_idx): function returning transmittance at pixel (x,y) in view
    training_views: List of view objects, each containing:
               - image_id: index of the view
               - get_feature_map(): returns 2D feature tensor F_2D of shape (H, W, D)
    feature_extractor: Function to generate F_2D from raw image if not pre-computed.
    
    Returns:
    - features: Dict mapping gaussian_id -> feature_vector (D,)
    - pruned_ids: List of gaussian_ids pruned due to zero influence
    """
    # Initialize accumulators
    # Using dictionaries to handle sparse Gaussians efficiently
    num_accumulators = {} # id -> vector (D,)
    den_accumulators = {} # id -> float
    
    for g in gaussians:
        num_accumulators[g.id] = np.zeros(g.feature_dim)
        den_accumulators[g.id] = 0.0
        
    pruned_ids = []
    
    # Step 1 & 3: Iterate through views and pixels
    for view in training_views:
        # Get 2D feature map for this view
        F_2D = view.get_feature_map() # Shape: (H, W, D)
        
        # Iterate over all pixels (or use rasterizer to get contributing Gaussians per pixel)
        # Optimization: Use gsplat rasterizer to get alpha and T for all Gaussians per pixel
        # Here we assume a rasterizer call returns a list of (x, y, gaussian_id, alpha, T)
        contributions = rasterize_view(view, gaussians) 
        
        for x, y, g_id, alpha, T in contributions:
            if alpha <= 0:
                continue
                
            weight = alpha * T
            feature_vec = F_2D[x, y]
            
            # Update accumulators
            num_accumulators[g_id] += feature_vec * weight
            den_accumulators[g_id] += weight
            
    # Step 4 & 5: Finalize and prune
    features = {}
    for g in gaussians:
        g_id = g.id
        num = num_accumulators[g_id]
        den = den_accumulators[g_id]
        
        if den == 0:
            pruned_ids.append(g_id)
            continue
            
        # Expected Feature Back-Projection (Equation 3)
        f_k = num / den
        features[g_id] = f_k
        
    return features, pruned_ids
```

**Parameters:**
- **Feature Extractor:** LSeg [S1] or DINOv2 [S1] are used for generating $\mathbf{F}_{2D}$.
- **Rasterizer:** gsplat [S1] is used for rendering.
- **Hardware:** Experiments performed on NVIDIA A6000 GPU [S1].

## Subtopics

### 3D Segmentation via Back-Projection
A method for segmenting objects in the 3D scene using the back-projected features. It obtains feature vectors $\{\mathbf{f}_k\}$ for each Gaussian, extracts a query embedding $\mathbf{q}$ from a language query or other modality, and computes similarity $sim(\mathbf{f}_k, \mathbf{q})$ (typically cosine similarity) [S1]. Segmented Gaussians $G$ are defined by thresholding similarity: $G = \{g_k \mid \text{sim}(\mathbf{f}_k, \mathbf{q}) > \theta\}$, where $\theta$ is a user-defined threshold [S1]. 2D segmentation mask $M(x, y)$ is defined as 1 if $\text{sim}(\mathbf{f}_k, \mathbf{q}) > \theta$, else 0 [S1].

### Identity Encoding
Encoding each object with a unique vector and back-projecting these vectors into the 3D scene, generalizing across the entire scene given sufficient annotated views [S1]. Two variants exist:
1. **Orthogonal Encoding:** Assigns procedurally generated orthogonal vectors to represent each object uniquely. This maximizes separation and simplifies segmentation but is limited by the embedding dimension $D$ [S1].
2. **Contrastive Encoding:** Uses non-orthogonal vectors made distinct through contrastive learning, maximizing distance between different objects and minimizing distance for same objects [S1].
The loss function for the Identity Encoder-Decoder is $\mathcal{L} = \mathcal{L}_{\text{classification}} + \mathcal{L}_{\text{orthogonality}}$, where $\mathcal{L}_{\text{classification}} = \text{CrossEntropy}(\hat{y}, y)$ and $\mathcal{L}_{\text{orthogonality}} = \|\mathbf{E}\mathbf{E}^T - \mathbf{I}\|_F$. Here $\hat{y}$ is predicted class probability, $y$ is true label, $\mathbf{E}$ is embedding matrix of size $N \times D$ ($N$ classes, $D$ dimension), $\mathbf{I}$ is identity matrix, and $\| \cdot \|_F$ is Frobenius norm [S1].

### Affordance Transfer
Transferring DINOv2 features to target scenes via feature back-projection. It identifies the closest matching source annotation for each Gaussian in the target scene by performing k-nearest neighbors (kNN) classification on the feature space [S1].

## Operating characteristics
- **Time Complexity:** The method requires a single pass through all training views for back-projection [S1]. The entire process, including ground truth feature map generation, takes an average of 2-3 minutes [S1].
- **Inference Speed:** Object segmentation (including text encoding and similarity calculation) takes 30 ms [S1]. Similarity calculation alone takes less than 1 ms [S1].
- **Comparison:** It achieves a 900x speedup compared to segmentation using masked gradients [7] for the full pipeline including mask generation [S1]. Compared to Feature 3DGS, this method is 10x faster (2.5 mins vs 20-30 mins for 7000 iterations) because it does not train a feature field [S1].
- **Strengths:** Training-free, fast, scalable, high-quality 3D segmentation results compared to training-based methods [S1].
- **Limitations:** Requires pre-trained 3D Gaussians and training views with feature maps. Performance depends on the quality of the 2D feature extractor [S1].
- **When to use:** 3D segmentation, affordance transfer, and identity encoding in 3D Gaussian Splatting [S1].

## Background
The method was introduced by Bharadwaj Amrutur and Shalabh Bhatnagar from the Indian Institute of Science [S1]. It addresses the limitations of training-based feature field rendering methods, which excel at 2D segmentation but perform poorly at 3D segmentation without post-processing [S1]. The approach leverages the gradient of the rendered color with respect to Gaussian colors to weight the back-projection of 2D features, avoiding the need for distillation or feature field training [S1].

## Sources
[S1] Gradient-Weighted Feature Back-Projection: A Fast Alternative to Feature Distillation in 3D Gaussian Splatting, https://doi.org/10.1145/3757377.3763926, fetched 2026-10-27
