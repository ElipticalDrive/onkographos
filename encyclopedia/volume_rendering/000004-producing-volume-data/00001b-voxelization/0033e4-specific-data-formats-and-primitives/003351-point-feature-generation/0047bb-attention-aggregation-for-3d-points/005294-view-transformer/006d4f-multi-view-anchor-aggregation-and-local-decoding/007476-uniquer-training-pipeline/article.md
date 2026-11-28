# UniQueR Training Pipeline

Also called: UniQueR

## Overview
The UniQueR training pipeline is a feedforward method for generating sparse 3D Gaussian representations from multi-view images, designed to solve the problem of incomplete geometry reconstruction in occluded or unseen regions. Unlike pixel-aligned methods that leave empty holes in unsupported areas, UniQueR uses learnable spatial anchors and held-out-view supervision to allocate structure in global 3D space beyond observed surfaces [S1]. It builds on [Multi-view Anchor Aggregation and Local Decoding](21319-multi-view-anchor-aggregation-and-local-decoding.md) by refining point-based features into a set of 3D Gaussians via a query-based transformer architecture.

## How it works

### Inputs and Outputs
**Inputs:**
- A set of multi-view images $\{I_i\}$ for a scene. The number of input views is randomly sampled between 2 and 64 during training [S1].
- Input images are resized to a dynamic resolution (224 or 448) with random aspect ratios (1.0, 1.33, 1.52, 1.77, 2.0) [S1].

**Outputs:**
- A set of 3D Gaussians $G$ representing the scene geometry and appearance.
- Predicted RGB images $\hat{I}$ and depth maps $\hat{D}$ for rendering [S1].

### Data Structures
- **Queries ($Q$):** A set of learnable vectors. The total number of queries is fixed at $Q = 4096$ [S1].
- **Gaussians ($G$):** Each query spawns $K = 64$ Gaussians, resulting in a total of $\le 262\text{K}$ Gaussians per scene [S1].
- **Features ($T_i$):** DINO features extracted from input images [S1].
- **Point Predictions:** For each view, the decoder predicts point positions $P_i$, point features $X_i$, and colors $C_i$ [S1].

### Steps
1. **Feature Extraction:** Compute DINO features for each input image:
   $$T_i \leftarrow \text{DINO}(I_i)$$
   [S1]

2. **Attention Aggregation:** Process the DINO features through an AA-Transformer to produce updated features:
   $$\{T_i\} \leftarrow \text{AA-Transformer}(\{T_i\})$$
   [S1]

3. **Point Prediction:** Decode the aggregated features to predict point positions $P_i$, point features $X_i$, and colors $C_i$:
   $$\{P_i, X_i, C_i\} = \text{Decoder}(\{T_i\})$$
   [S1]

4. **Query Initialization:** Initialize the spatial queries $Q$ based on the predicted point positions $P_i$. Half of the queries are initialized from the predicted pointmaps. The other half are learnable spatial anchors initialized uniformly in $[-1.2, 1.2]^3$, scaled according to the predicted point range during training [S1].
   $$Q \leftarrow \text{InitQueries}(P_i)$$
   [S1]

5. **Query Refinement:** Iterate $L$ times (number of transformer layers, not specified in sources) to refine queries:
   - Cross-attention between queries and image features:
     $$Q \leftarrow \text{Cross-Attn}(Q, \{T_i\})$$
   - Self-attention among queries:
     $$Q \leftarrow \text{Self-Attn}(Q)$$
   [S1]

6. **Gaussian Spawning:** Spawn 3D Gaussians $G$ from the refined queries:
   $$G \leftarrow \text{SpawnGaussians}(Q)$$
   [S1]

7. **Rendering:** Render the Gaussians to produce predicted RGB images $\hat{I}$ and depth maps $\hat{D}$ using a differentiable Gaussian rasterizer [S1].
   $$\hat{I}, \hat{D} \leftarrow \text{Render}(G)$$
   [S1]

8. **Loss Computation and Update:** Compute losses and update model parameters [S1].

### Formulas
**Losses:**
The total loss is composed of:
1. **RGB Reconstruction Loss:** Composed of $\ell_1$ loss and LPIPS (Learned Perceptual Image Patch Similarity) [S1].
   $$\mathcal{L}_{\text{RGB}} = \mathcal{L}_1(\hat{I}, I) + \lambda_{\text{LPIPS}} \mathcal{L}_{\text{LPIPS}}(\hat{I}, I)$$
   [S1]

2. **Scale-Invariant Depth Loss:** Applied to the rendered depth $\hat{D}$ against ground-truth depth $D$ (if available) or used for regularization [S1].
   $$\mathcal{L}_{\text{depth}} = \mathcal{L}_{\text{SI}}(\hat{D}, D)$$
   [S1]

3. **Camera Loss:** Inherited from the backbone [S1].

$$\mathcal{L}_{\text{total}} = \mathcal{L}_{\text{RGB}} + \mathcal{L}_{\text{depth}} + \mathcal{L}_{\text{camera}}$$
[S1]

### Parameters
- **Number of queries $Q$:** 4096 [S1].
- **Gaussians per query $K$:** 64 [S1].
- **Total Gaussians:** $\le 262\text{K}$ [S1].
- **Optimizer:** AdamW [S1].
- **Initial learning rate:** $1 \times 10^{-4}$ [S1].
- **Training schedule:** 100 epochs at resolution 224, followed by 20 epochs at resolution 448 [S1].
- **Input views per scene:** Randomly sampled 2–64 views [S1].
- **Dynamic aspect ratios:** 1.0, 1.33, 1.52, 1.77, and 2.0 [S1].
- **Training hardware:** 32x A100 GPUs [S1].
- **Timing hardware:** Single A100 80GB GPU [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniquer_training_pipeline(input_images, num_queries=4096, gaussians_per_query=64, 
                              num_epochs_224=100, num_epochs_448=20, lr=1e-4):
    """
    UniQueR Training Pipeline.
    
    Args:
        input_images: List of multi-view images {I_i} for a scene.
        num_queries: Number of spatial queries Q.
        gaussians_per_query: Number of Gaussians K spawned per query.
        num_epochs_224: Number of training epochs at resolution 224.
        num_epochs_448: Number of fine-tuning epochs at resolution 448.
        lr: Initial learning rate for AdamW optimizer.
        
    Returns:
        Trained model parameters.
    """
    # Initialize model and optimizer
    model = UniQueRModel(num_queries=num_queries, gaussians_per_query=gaussians_per_query)
    optimizer = AdamW(model.parameters(), lr=lr)
    
    # Training schedule
    resolutions = [(224, num_epochs_224), (448, num_epochs_448)]
    aspect_ratios = [1.0, 1.33, 1.52, 1.77, 2.0]
    
    for resolution, epochs in resolutions:
        for epoch in range(epochs):
            # Randomly sample 2-64 input views per scene
            num_views = random.randint(2, 64)
            selected_images = random.sample(input_images, num_views)
            
            # Randomly select aspect ratio
            aspect_ratio = random.choice(aspect_ratios)
            
            # Resize images to resolution with selected aspect ratio
            resized_images = [resize_with_aspect_ratio(img, resolution, aspect_ratio) for img in selected_images]
            
            # Step 1: Compute DINO features
            dino_features = [dino_extractor(img) for img in resized_images]
            
            # Step 2: Process DINO features through AA-Transformer
            aggregated_features = aa_transformer(dino_features)
            
            # Step 3: Decode features to predict point positions, features, and colors
            point_positions, point_features, point_colors = decoder(aggregated_features)
            
            # Step 4: Initialize spatial queries
            # Half from predicted pointmaps, half from uniform [-1.2, 1.2]^3 scaled by point range
            queries = init_queries(point_positions)
            
            # Step 5: Refine queries via Cross-Attention and Self-Attention
            # Number of layers L is not specified in sources
            for _ in range(L):
                queries = cross_attention(queries, aggregated_features)
                queries = self_attention(queries)
            
            # Step 6: Spawn 3D Gaussians from refined queries
            gaussians = spawn_gaussians(queries, gaussians_per_query)
            
            # Step 7: Render Gaussians to produce predicted RGB and depth
            predicted_rgb, predicted_depth = render_gaussians(gaussians)
            
            # Step 8: Compute losses
            # RGB loss: L1 + LPIPS
            rgb_loss = l1_loss(predicted_rgb, resized_images) + lpips_loss(predicted_rgb, resized_images)
            
            # Depth loss: Scale-invariant depth loss
            # Note: Ground truth depth may not be available for all views; 
            # used for regularization and reducing floaters/holes [S1]
            depth_loss = scale_invariant_depth_loss(predicted_depth)
            
            # Camera loss: Inherited from backbone
            camera_loss = camera_loss_from_backbone(aggregated_features)
            
            total_loss = rgb_loss + depth_loss + camera_loss
            
            # Update parameters
            optimizer.zero_grad()
            total_loss.backward()
            optimizer.step()
            
    return model
```

### Edge Cases and Pitfalls
- **Empty holes in unseen regions:** Pixel-aligned predictions tend to leave empty holes in unsupported regions. UniQueR handles this by placing Gaussians in occluded space via held-out-view supervision [S1].
- **Supervision views:** The model is supervised on a superset of input views, including held-out views, to penalize empty holes and encourage modeling geometry beyond observed surfaces [S1].

## Operating characteristics

### Time and Memory Complexity
- **Memory:** The method uses $\le 262\text{K}$ Gaussians per scene, with 4096 queries and 64 Gaussians per query [S1].
- **Time:** Training is performed on 32x A100 GPUs [S1]. Timing hardware is a single A100 80GB GPU [S1]. Specific timing metrics are not provided in the sources.

### Measured Performance
- **Hardware:** 32x A100 GPUs for training, single A100 80GB GPU for timing [S1].
- **Resolution:** 224 and 448 [S1].
- **Scene:** Not specified in the sources.

### Scaling
- The method supports dynamic aspect ratios (1.0, 1.33, 1.52, 1.77, 2.0) and random sampling of 2–64 input views per scene [S1].

### Strengths and Limitations
- **Strengths:** 
  - Allocates structure in occluded regions via sparse 3D queries and held-out-view supervision [S1].
  - Reduces floaters and holes in difficult views through scale-invariant depth loss [S1].
- **Limitations:** 
  - Pixel-aligned methods may leave empty holes in unsupported regions, which UniQueR addresses but may require more computational resources due to the query-based approach [S1].

### When to Use
- Use UniQueR for sparse-view novel-view synthesis where complete geometry reconstruction in occluded regions is critical [S1].
- Suitable for scenarios requiring feedforward 3D reconstruction with high geometric accuracy [S1].

### When Not to Use
- Not specified in the sources.

## Background

### Origin
- UniQueR was introduced by C. Peng et al. in the paper "UniQueR: Unified Query-Based Feedforward 3D Reconstruction" [S1].

### History and Development
- The method builds on prior work in query-based 3D reconstruction and Gaussian splatting [S1].
- It follows MapAnything for dynamic resolution during training [S1].

### Usage
- **Evaluation Protocol for Sparse-View Novel-View Synthesis:** Input and testing views are strictly disjoint. PSNR, SSIM, and LPIPS are measured only on held-out views. 5 curated subsets per scene are evaluated and averaged [S1].
- **Evaluation Protocol for Dense-View Setting (32/64 Views):** Evaluates both direct feedforward predictions and per-scene optimization initialized from the method. The optimizer receives predicted camera poses and Gaussian/point initialization [S1].
- **Geometry Metric:** Absolute relative error (Abs Rel) computed against ground-truth depth, rendered from predicted Gaussians under evaluation cameras. Reported numbers use 32 input views at resolution $448 \times 448$ [S1].
- **Post-Optimization Initialization:** For VGGT, points are used directly. For AnySplat and UniQueR, only Gaussian centers and colors are retained for initialization. The downstream optimization stage is identical across methods [S1].

## Sources

[S1] UniQueR: Unified Query-Based Feedforward 3D Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_20, fetched 2026-10-30
