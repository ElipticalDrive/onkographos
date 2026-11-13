# DFF (Decomposing NeRF for Editing Via Feature Field Distillation)
Also called: Feature Field Distillation, LSeg-DFF, DINO-DFF

## Overview
DFF enables 3D semantic segmentation and editable novel-view synthesis by distilling semantic features from a pre-trained 2D teacher network (such as LSeg or DINO) into a neural radiance field [S1]. Unlike standard NeRFs where hidden features are entangled with low-level factors like color or spatial adjacency, DFF creates a dedicated feature field that allows for semantic-aware decomposition, enabling operations such as object deletion, extraction, and appearance editing while maintaining multi-view consistency [S1]. It sets itself apart from sibling methods like [NeRF Editing Procedure](12491-nerf-editing-procedure.md) by explicitly learning a semantic feature space via distillation rather than relying solely on geometric masks or post-hoc optimization, and it avoids the view-inconsistency issues of image-based inpainting baselines like LaMa [S1].

## How it works
DFF operates by training a radiance field to predict both color and a high-dimensional semantic feature vector for each point in space. The semantic features are supervised by target feature maps generated from a 2D teacher network, allowing the 3D field to inherit 2D semantic understanding.

### Inputs and Outputs
*   **Inputs:**
    *   Training images $I$ with known camera poses.
    *   A 2D teacher network (LSeg or DINO) [S1].
    *   For LSeg: Text labels encoded via the teacher's text encoder [S1].
    *   For DINO: Image patch queries [S1].
*   **Outputs:**
    *   A radiance field model predicting density $\sigma$, color $\mathbf{c}$, and semantic feature $\mathbf{f}$.
    *   Semantic segmentation maps derived from $\mathbf{f}$ via similarity to query vectors.
    *   Edited novel views by modifying $\mathbf{c}$ or $\sigma$ based on semantic masks derived from $\mathbf{f}$ [S1].

### Data Structures and Layouts
The model consists of two main components: the Radiance Field MLP and the Feature Field (optional/independent).

1.  **Radiance Field MLP (Branch-based DFF):**
    *   **Architecture:** 8 ReLU layers with 256 dimensions.
    *   **Outputs:**
        *   Density $\sigma$: Linear layer (1 output).
        *   Color $\mathbf{c}$: 3 ReLU layers, then linear layer (3 outputs).
        *   Feature $\mathbf{f}$: 3 ReLU layers, then linear layer (dimension $D_f$, typically matching teacher output) [S1].
    *   **Positional Encoding (PE):** Length 10 for input coordinates (and skip connection), Length 4 for viewing direction [S1].

2.  **Independent Feature Field MLP (Optional):**
    *   Used to avoid retraining the radiance field or to introduce DFF into arbitrary 3D representations [S1].
    *   **Architecture:** 4 layers.
    *   **Skip Connection:** Included at the third layer if PE is used [S1].
    *   **Smoothness Variant:** An independent MLP can be trained *without* positional encoding to act as a smoothness regularizer, reducing high-frequency artifacts in volume decomposition [S1].

### Numbered Steps

1.  **Teacher Feature Extraction:**
    *   Encode each training image $I$ using the teacher network (LSeg or DINO) to generate target feature maps $f_{img}(I, r)$ [S1].
    *   **Resize:** Resize these feature maps to the original image resolution ($H \times W$) because teacher networks often output reduced-size features [S1].
    *   **Query Probability Calculation:**
        *   *For LSeg:* Encode text labels using the LSeg text encoder. Calculate the probability $p$ of a 3D point belonging to a label using the cosine similarity between the rendered feature $\mathbf{f}$ and the encoded label [S1]. (Equation 22 in source) [S1].
        *   *For DINO:* Use image patch queries. Calculate the patch feature by averaging features of all pixels in the patch. Compute probability $p$ using thresholded cosine similarity: $p=1$ if similarity $>$ threshold, else $p=0$ (hard decomposition) [S1].

2.  **Radiance Field Initialization (Pre-training):**
    *   Train a standard NeRF (without feature branches) for 200,000 iterations using only photometric loss $L_p$ [S1]. This establishes the geometry and color baseline [S1].

3.  **Feature Field Distillation (Finetuning):**
    *   Add the feature branch to the radiance field (or initialize the independent feature MLP) [S1].
    *   Remove Gaussian noise from density [S1].
    *   Set learning rate to $1 \times 10^{-4}$ [S1].
    *   Train for 5,000 iterations using the combined loss $L_p + \lambda L_f$ [S1].
    *   **Sampling Strategy:** Use **coarse sampling** (64 points) for training the feature field. Coarse sampling produces smoother segmentations and avoids high-frequency artifacts common in fine sampling [S1]. For final edited rendering, fine sampling (128 points) can be used, but coarse features are often sufficient for smooth decomposition [S1].

4.  **Loss Calculation:**
    *   The total loss is:
        $$ L = L_p + \lambda L_f $$
    *   $L_p$: Photometric loss (e.g., $L_1$ or MSE between rendered and ground-truth RGB) [S1].
    *   $L_f$: Feature loss. This is the distance between the rendered feature $\mathbf{f}$ and the resized teacher feature map $f_{img}$ at the corresponding pixel location [S1].
    *   $\lambda$: Weight for feature loss. Typical value is $\lambda=1$ (source implies standard weighting, but notes that $\lambda \times 10$ hurts view synthesis) [S1].

5.  **Editing/Inpainting:**
    *   Render the feature field for a novel view.
    *   Compute semantic probabilities for each pixel based on the query (text or patch) [S1].
    *   Generate a mask $M$ based on the probability threshold [S1].
    *   Modify the color $\mathbf{c}$ or density $\sigma$ in the masked regions (e.g., set to background, apply color transform, or inpaint) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DFFRadianceField(nn.Module):
    def __init__(self, feature_dim=256, use_independent_feature=False):
        super().__init__()
        # Radiance Field MLP: 8 hidden layers of 256 ReLU
        self.shared_layers = nn.ModuleList([
            nn.Linear(256, 256) for _ in range(8)
        ])
        
        # Outputs
        self.density_layer = nn.Linear(256, 1)
        self.color_layers = nn.ModuleList([
            nn.Linear(256, 256) for _ in range(3)
        ])
        self.color_out = nn.Linear(256, 3)
        
        if not use_independent_feature:
            # Branch-based: 3 hidden layers for feature
            self.feature_layers = nn.ModuleList([
                nn.Linear(256, 256) for _ in range(3)
            ])
            self.feature_out = nn.Linear(256, feature_dim)
        else:
            self.feature_mlp = None # Initialized separately if independent

    def forward(self, xyz, dirs, pe_xyz, pe_dirs):
        # Concatenate PE
        inputs = torch.cat([pe_xyz, pe_dirs], dim=-1)
        h = inputs
        
        # Shared layers
        for layer in self.shared_layers:
            h = F.relu(layer(h))
            
        density = self.density_layer(h)
        
        # Color branch
        c = h
        for layer in self.color_layers:
            c = F.relu(layer(c))
        color = self.color_out(c)
        
        # Feature branch (Branch-based)
        if self.feature_mlp is None:
            f = h
            for layer in self.feature_layers:
                f = F.relu(layer(f))
            feature = self.feature_out(f)
        else:
            feature = self.feature_mlp(pe_xyz) # Independent MLP takes only xyz PE
            
        return density, color, feature

def get_pe(pos, L=10):
    """Standard NeRF positional encoding"""
    # Implementation of standard sin/cos encoding
    # Returns concatenated [sin(2^0 pi pos), cos(2^0 pi pos), ..., sin(2^(L-1) pi pos), cos(...)]
    pass

def train_dff(scene_data, teacher_network, teacher_type='LSeg'):
    """
    Main training loop for DFF.
    """
    # 1. Pre-train standard NeRF for 200k iterations (omitted for brevity, assume model exists)
    model = DFFRadianceField()
    
    # 2. Add feature branch or initialize independent MLP
    # ... (model modification) ...
    
    optimizer = torch.optim.Adam(model.parameters(), lr=1e-4)
    
    # 3. Finetune for 5k iterations
    for step in range(5000):
        # Sample rays
        rays = sample_rays(scene_data, batch_size=1024) # 1024 for Replica, 2048 for others
        
        # Forward pass with Coarse Sampling (64 points)
        # Note: Coarse sampling is recommended for feature field training
        density, color, feature = model.forward(
            rays.origins, 
            rays.directions, 
            get_pe(rays.origins, L=10), 
            get_pe(rays.directions, L=4)
        )
        
        # Volume Rendering for Color (using coarse density/color)
        rendered_color = volume_render_color(rays, density, color)
        
        # Volume Rendering for Feature (using coarse density/feature)
        rendered_feature = volume_render_feature(rays, density, feature)
        
        # Get Teacher Targets
        # Resize teacher features to image size
        teacher_features = teacher_network.encode(rays.images)
        teacher_features = F.interpolate(teacher_features, size=rays.image_size, mode='bilinear')
        
        # Calculate Losses
        # L_p: Photometric loss
        loss_p = F.l1_loss(rendered_color, rays.images)
        
        # L_f: Feature loss (MSE between rendered feature and teacher feature)
        loss_f = F.mse_loss(rendered_feature, teacher_features)
        
        # Combined Loss
        lambda_f = 1.0 # Typical weight; higher values (e.g., 10) hurt view synthesis
        loss = loss_p + lambda_f * loss_f
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

def query_semantics(model, rays, query_text=None, query_patch=None, teacher_type='LSeg'):
    """
    Generate semantic mask for editing.
    """
    # Render features
    density, _, feature = model.forward(...)
    rendered_feature = volume_render_feature(rays, density, feature)
    
    if teacher_type == 'LSeg':
        # Encode text query
        query_vec = teacher_network.encode_text(query_text)
        # Cosine similarity
        sim = F.cosine_similarity(rendered_feature, query_vec)
        prob = sim # Or sigmoid transform depending on teacher
    elif teacher_type == 'DINO':
        # Query patch feature (average of patch pixels)
        query_vec = teacher_network.encode_patch(query_patch)
        sim = F.cosine_similarity(rendered_feature, query_vec)
        # Hard threshold
        prob = (sim > threshold).float()
        
    return prob
```

### Parameters and Typical Values
*   **Teacher Networks:** LSeg [S1] or DINO [S1].
*   **Training Iterations:** 200,000 for pre-training (radiance only); 5,000 for finetuning (with feature branch) [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:**
    *   Pre-training: Linearly decaying from $5 \times 10^{-4}$ to $8 \times 10^{-5}$ [S1].
    *   Finetuning: Fixed at $1 \times 10^{-4}$ [S1].
*   **Sampling:**
    *   Coarse: 64 points (stratified) [S1].
    *   Fine: 128 points (importance) [S1].
    *   *Note:* Source notes "192 in total" for fine sampling in one context, but explicitly lists "128" in the implementation settings paragraph. We use 128 as the specific setting value [S1].
*   **Positional Encoding:** Length 10 for coordinates, Length 4 for viewing direction [S1].
*   **Image Resolution:** 320×240 (Replica dataset); 1008×756 (other datasets) [S1].
*   **Batch Size:** 1024 rays (Replica); 2048 rays (others) [S1].
*   **Feature Loss Weight ($\lambda$):** Default 1. Increasing to $\lambda \times 10$ degrades view synthesis without improving segmentation [S1].

### Edge Cases and Pitfalls
*   **Feature Loss Convergence:** Feature loss converges significantly faster than photometric loss. Therefore, short finetuning (5K iterations) is sufficient; longer training may overfit to features at the expense of photometric accuracy [S1].
*   **Excessive Feature Weight:** Setting $\lambda$ too high (e.g., 10x default) hurts novel view synthesis quality without further improving segmentation performance [S1].
*   **Naive NeRF Features:** Using hidden layers of a standard NeRF (e.g., the 8th layer) for decomposition is not robust. These features are entangled with low-level factors (color, geometry, position), causing "leakage" where selections bleed into unrelated regions with similar colors or positions (e.g., floor selection leaking to walls) [S1].
*   **Deletion Artifacts:** Deleting foreground objects may result in floating artifacts and blurred volumes in the far distance behind the deleted object [S1].
*   **Sampling Impact:** Fine sampling introduces high-frequency artifacts in semantic segmentation. Coarse sampling is preferred for smooth decomposition [S1].
*   **Independent MLP:** If using an independent feature MLP, the radiance field MLP must still be prepared to reconstruct geometry well, as the feature field relies on geometrically plausible volume rendering [S1].

## Operating characteristics
*   **Time Complexity:** Training requires two phases: 200K iterations for standard NeRF and 5K iterations for distillation. The distillation phase is short due to fast convergence of feature loss [S1].
*   **Memory Complexity:** Adds a feature output dimension to the radiance field MLP (or an additional MLP). The branch-based approach has small computational and parameter overhead compared to original NeRF [S1].
*   **Performance:**
    *   **Segmentation:** Achieves mean Intersection-over-Union (mIoU) and accuracy comparable to or better than supervised sparse 3D convolution models like MinkowskiNet42, despite using no 3D semantic labels during training [S1].
    *   **View Synthesis:** Does not degrade novel view synthesis quality compared to NeRF without feature fields, provided $\lambda$ is not excessive [S1].
*   **Strengths:**
    *   Enables zero-shot semantic segmentation via text (LSeg) or patch (DINO) queries [S1].
    *   Produces multi-view consistent results for editing/deletion, unlike image inpainting baselines (e.g., LaMa) [S1].
    *   Independent feature MLP allows integration into existing 3D representations (dynamic NeRFs, meshes) without retraining the radiance field [S1].
*   **Limitations:**
    *   Struggles with "volume decomposition" smoothness if fine sampling or high-frequency PE is used without regularization [S1].
    *   Deletion leaves artifacts in occluded background regions [S1].
    *   Coarse training may lack high-frequency detail in the feature field [S1].

## Background
DFF was introduced in the paper "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It addresses the challenge that standard NeRF feature spaces are not semantically meaningful, being entangled with low-level visual cues. By distilling features from powerful 2D teacher networks (LSeg and DINO), DFF bridges the gap between 2D semantic understanding and 3D radiance fields, enabling editable 3D scenes. It builds on the standard NeRF training loop [NeRF Training Loop](12494-nerf-editing-and-inpainting-procedures.md) but adds a distillation objective.

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
