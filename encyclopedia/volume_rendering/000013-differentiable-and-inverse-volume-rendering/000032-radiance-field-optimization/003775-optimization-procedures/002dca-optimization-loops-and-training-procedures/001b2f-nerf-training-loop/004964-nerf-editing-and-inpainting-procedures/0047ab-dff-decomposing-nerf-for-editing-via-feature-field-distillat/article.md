# DFF (Decomposing NeRF for Editing Via Feature Field Distillation)

Also called: Feature Field Distillation

## Overview
DFF enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a pre-trained 2D teacher network (such as LSeg or DINO) into a neural radiance field [S1]. It solves the problem that standard NeRF feature spaces are entangled with low-level factors like color or spatial adjacency rather than semantic similarity, making naive decomposition unreliable [S1]. Unlike sibling methods that rely on explicit masks or inpainting baselines which suffer from multi-view inconsistency, DFF achieves multi-view consistent decomposition and editing through feature distillation [S1].

## How it works
**Inputs:** Training images $I$ with known camera poses; a 2D teacher network (LSeg or DINO); optional text labels (for LSeg) or image patch queries (for DINO).
**Outputs:** A radiance field $F_\Theta$ producing color $c$ and density $\sigma$, and a feature field producing semantic features $f$.

**Data Structures:**
1.  **Radiance Field MLP:** 8 ReLU layers with 256 dimensions, followed by a linear layer for density, 3 layers for color, and 3 layers for feature [S1].
2.  **Independent Feature Field MLP (Optional):** 4 layers. Includes a skip connection at the third layer if positional encoding is used [S1].
3.  **Positional Encoding:** Length 10 for input coordinates and skip connections; length 4 for viewing direction [S1].
4.  **Teacher Features:** Target feature maps $f_{img}(I, r)$ resized to original image resolution [S1].

**Numbered Steps:**

1.  **Teacher Encoding:** Encode each training image $I$ using the image encoder of the teacher network to generate target feature maps $f_{img}(I, r)$ [S1].
2.  **Resizing:** Resize the target feature maps to the original image size, as teacher networks produce reduced-size maps [S1].
3.  **NeRF Initialization (Phase 1):** Train a standard NeRF (without a feature branch) for 200,000 iterations using only the photometric loss $L_p$ [S1].
4.  **Feature Distillation (Phase 2):** Finetune the NeRF by adding a feature branch (or independent MLP) and minimize the combined loss $L_p + \lambda L_f$ for 5,000 iterations [S1].
    *   During Phase 2, remove Gaussian noise applied to density and set the learning rate to $1 \times 10^{-4}$ [S1].
5.  **Sampling Strategy:** Use coarse sampling (64 points) for training the feature field to ensure smooth segmentations; use fine sampling (128 points) for edited rendering [S1].
6.  **Query Processing:**
    *   **LSeg-DFF:** Encode text labels using the LSeg text encoder. Calculate point probability $p$ using Equation 22 (softmax over label probabilities) [S1].
    *   **DINO-DFF:** Calculate patch features by averaging features of all pixels in the query patch. Compute probability $p$ using thresholded cosine similarity: $p=1$ if similarity exceeds threshold, else $p=0$ (hard decomposition) [S1].
7.  **Rendering:** Render novel views using the standard volume rendering integral [S1, 12372].

**Formulas:**
*   **Total Loss:**
    $$ L = L_p + \lambda L_f $$
    where $L_p$ is the photometric loss (e.g., $L_1$ or MSE between rendered and ground-truth RGB) and $L_f$ is the feature distillation loss [S1].
*   **Feature Loss ($L_f$):**
    $$ L_f = \| f_{rendered} - f_{img}(I, r) \| $$
    where $f_{rendered}$ is the feature output by the radiance/feature field and $f_{img}$ is the resized teacher feature map [S1].
*   **LSeg Probability (Eq. 22):**
    $$ p_k = \frac{\exp(\text{sim}(f_{rendered}, t_k))}{\sum_j \exp(\text{sim}(f_{rendered}, t_j))} $$
    where $t_k$ is the text embedding for label $k$ [S1].
*   **DINO Probability:**
    $$ p = \begin{cases} 1 & \text{if } \text{cosine}(f_{rendered}, f_{patch}) > \tau \\ 0 & \text{otherwise} \end{cases} $$
    where $\tau$ is a threshold [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_dff(training_images, teacher_network, query_type='text', query_data=None):
    # 1. Initialize Radiance Field
    nerf = NeRFMLP(layers=8, hidden_dim=256, 
                   density_layers=1, color_layers=3, feature_layers=3)
    
    # 2. Phase 1: Train standard NeRF
    for iteration in range(200000):
        rays = sample_rays(training_images, batch_size=2048)
        color_pred, density_pred = nerf.forward(rays)
        loss_p = photometric_loss(color_pred, rays.gt_color)
        loss = loss_p
        optimizer.step(loss)
        apply_gaussian_noise_to_density(density_pred) # [G] Standard NeRF practice per [S1]
        
    # 3. Prepare Teacher Features
    teacher_features = []
    for img in training_images:
        feat = teacher_network.encode(img)
        feat_resized = resize(feat, img.size) # Resize to original image size
        teacher_features.append(feat_resized)
        
    # 4. Phase 2: Distill Features
    # Add feature branch to existing MLP or train independent MLP
    nerf.add_feature_branch() 
    
    optimizer = Adam(learning_rate=1e-4) # Reset optimizer with new LR
    remove_gaussian_noise() # [G] Explicitly disable noise as per [S1]
    
    for iteration in range(5000):
        rays = sample_rays(training_images, batch_size=2048, sampling='coarse') # Use coarse sampling
        color_pred, density_pred, feature_pred = nerf.forward(rays)
        
        # Get target features for this ray's pixel
        target_feat = teacher_features[rays.image_index][rays.u, rays.v]
        
        loss_p = photometric_loss(color_pred, rays.gt_color)
        loss_f = l1_loss(feature_pred, target_feat)
        loss = loss_p + lambda_param * loss_f
        
        optimizer.step(loss)
        
    return nerf

def render_edit(nerf, rays, teacher_network, query_type, query_data):
    color, density, feature = nerf.forward(rays, sampling='fine')
    
    if query_type == 'text':
        # LSeg: Compute probability of each label
        text_emb = teacher_network.encode_text(query_data)
        prob = softmax_similarity(feature, text_emb)
    elif query_type == 'patch':
        # DINO: Thresholded cosine similarity
        patch_feat = average_features(query_data.patch_pixels)
        sim = cosine_similarity(feature, patch_feat)
        prob = (sim > threshold).float()
        
    return color, density, prob
```

**Parameters:**
*   **Teacher Networks:** LSeg [44] or DINO [12] [S1].
*   **Iterations:** 200,000 for Phase 1; 5,000 for Phase 2 [S1].
*   **Optimizer:** Adam, linearly decaying from $5 \times 10^{-4}$ to $8 \times 10^{-5}$ in Phase 1; fixed $1 \times 10^{-4}$ in Phase 2 [S1].
*   **Sampling:** Coarse: 64 points; Fine: 128 points (total 192 if hierarchical, but feature training uses coarse) [S1].
*   **PE Length:** 10 (coordinate), 4 (view) [S1].
*   **Image Size:** 320×240 (Replica); 1008×756 (others) [S1].
*   **Batch Size:** 1024 (Replica); 2048 (others) [S1].
*   **Feature Loss Weight ($\lambda$):** Default value not explicitly stated, but increasing by $\times 10$ is noted as harmful [S1].

**Edge Cases and Pitfalls:**
*   **Feature Loss Convergence:** Feature loss converges significantly faster than photometric loss; thus, short finetuning (5K iterations) is sufficient [S1].
*   **High Feature Weight:** Excessively increasing $\lambda$ (e.g., $\times 10$) hurts view synthesis without improving segmentation [S1].
*   **Naive NeRF Features:** Using hidden layers of the original NeRF (e.g., 8th layer) for decomposition fails; regions leak to parts with similar color/geometry/position [S1].
*   **Deletion Artifacts:** Deleting foreground objects may leave floating artifacts and blurred volumes in the far distance behind the object [S1].
*   **Smoothness:** Fine sampling introduces high-frequency artifacts in segmentation; coarse sampling or removing positional encoding (PE) in the feature MLP improves smoothness [S1].

## Subtopics

### Branch-based DFF
The feature branch shares layers with the radiance field. It does not degrade view synthesis performance compared to NeRF without feature fields and has small computational overhead [S1].

### Independent Feature Field DFF
Uses a separate, lightweight MLP for features. Achieves competitive segmentation results and allows introducing DFF into arbitrary 3D scene representations (dynamic NeRFs, meshes) without retraining the radiance field [S1].

### LSeg-DFF
Uses text queries via LSeg. Enables zero-shot semantic segmentation based on text labels [S1].

### DINO-DFF
Uses image patch queries via DINO. Uses thresholded cosine similarity for hard decomposition. Leverages self-supervised features for region selection [S1].

## Operating characteristics
*   **Time Complexity:** Phase 1 requires 200,000 iterations; Phase 2 requires 5,000 iterations [S1].
*   **Memory Complexity:** Adds feature output dimensions to the MLP; branch-based adds minimal overhead, independent MLP adds separate parameters [S1].
*   **Performance:** Achieves 3D semantic segmentation performance (mIoU) better than supervised sparse 3D convolution models like MinkowskiNet42 on Replica [S1].
*   **Strengths:** Multi-view consistent editing; successful distillation from 2D teachers; smooth decomposition with coarse sampling/no-PE [S1].
*   **Limitations:** Floating artifacts behind deleted objects; naive feature usage fails; high feature loss weight degrades synthesis [S1].
*   **When to use:** When 3D semantic segmentation or multi-view consistent editing is required without explicit 3D labels.
*   **When not to use:** When high-frequency geometric detail in the feature field is required (use fine sampling with caution) or when computational budget for 200K pre-training is unavailable.

## Background
DFF was introduced in the paper "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It addresses the challenge that editable NeRFs face in surface decomposition, volume decomposition, and estimating unobserved parts [S1]. The method builds on the observation that original NeRF features are entangled with low-level factors, necessitating distillation from external 2D semantic teachers like LSeg and DINO [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
