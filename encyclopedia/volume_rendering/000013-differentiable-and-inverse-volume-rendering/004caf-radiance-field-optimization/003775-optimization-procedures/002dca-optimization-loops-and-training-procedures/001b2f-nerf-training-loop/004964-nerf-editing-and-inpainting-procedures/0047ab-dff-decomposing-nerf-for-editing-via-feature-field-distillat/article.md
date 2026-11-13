# DFF (Decomposing NeRF for Editing Via Feature Field Distillation)
Also called: Feature Field Distillation, LSeg-DFF, DINO-DFF

## Overview
DFF enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a pre-trained 2D teacher network (LSeg or DINO) into a neural radiance field, allowing the decomposition of scenes into semantic regions [S1]. Unlike standard NeRFs where hidden features are entangled with low-level factors like color or spatial adjacency, DFF creates a dedicated feature space that supports semantic queries, enabling multi-view consistent editing, deletion, and extraction of objects [S1]. It sets itself apart from sibling methods by relying on 2D-to-3D knowledge distillation rather than explicit 3D supervision or diffusion-based generative priors [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed training images $I$, camera poses, and a pre-trained 2D teacher network (LSeg or DINO) [S1].
*   **Outputs:** A radiance field $F_\Theta$ mapping $(\mathbf{x}, \mathbf{v}) \to (\sigma, \mathbf{c})$ and a feature field $F_\Phi$ mapping $\mathbf{x} \to \mathbf{f}$, where $\mathbf{f}$ represents semantic features [S1].

**Data Structures:**
1.  **Radiance Field MLP ($F_\Theta$):**
    *   Architecture: 8 ReLU layers with 256 dimensions, followed by a linear layer for density $\sigma$, 3 layers for color $\mathbf{c}$, and 3 layers for feature $\mathbf{f}$ (in branch-based configuration) [S1].
    *   Positional Encoding (PE): Length 10 for input coordinate $\mathbf{x}$ and its skip connection; Length 4 for viewing direction $\mathbf{v}$ [S1].
2.  **Independent Feature Field MLP ($F_\Phi$) (Optional):**
    *   Architecture: 4 layers. Includes a skip connection at the third layer if PE is used [S1].
    *   Configuration: Can be trained without PE for smoothness regularization [S1].
3.  **Teacher Network Features:**
    *   LSeg: Text-encoded labels mapped to feature space [S1].
    *   DINO: Image patch features averaged from pixels in a query patch [S1].

**Numbered Steps:**

1.  **Teacher Feature Extraction:** Encode each training image $I$ using the teacher network's image encoder to generate target feature maps $f_{img}(I, r)$ [S1].
2.  **Feature Map Resizing:** Resize the target feature maps to the original image resolution (e.g., 320×240 or 1008×756) because the teacher networks output reduced-size features [S1].
3.  **NeRF Initialization:** Train a standard NeRF (radiance field only) for 200K iterations using photometric loss $\mathcal{L}_p$ [S1].
4.  **Feature Field Distillation (Finetuning):**
    *   Add a feature branch to the NeRF or train an independent feature field MLP [S1].
    *   Minimize the combined loss $\mathcal{L} = \mathcal{L}_p + \lambda \mathcal{L}_f$ for 5K iterations [S1].
    *   $\mathcal{L}_p$: Photometric loss (e.g., L1 or MSE between rendered and ground-truth RGB) [S1].
    *   $\mathcal{L}_f$: Feature distillation loss between rendered feature field values and teacher feature maps [S1].
    *   Remove Gaussian noise added to density during this phase [S1].
    *   Set learning rate to $1e^{-4}$ [S1].
5.  **Sampling Strategy:** Use coarse sampling (64 points) for training and rendering the feature field to ensure smooth segmentations [S1]. Fine sampling (128 points) can be used for final radiance rendering but introduces high-frequency artifacts in features [S1].
6.  **Query-Based Decomposition:**
    *   **LSeg-DFF:** Encode a text label using the LSeg text encoder. Compute the probability $p$ of each 3D point belonging to the label using cosine similarity in the feature space [S1].
    *   **DINO-DFF:** Define a query patch in an image. Calculate the patch feature by averaging features of all pixels in the patch. Compute probability $p$ using thresholded cosine similarity: $p=1$ if similarity exceeds threshold, $p=0$ otherwise (hard decomposition) [S1].
7.  **Editing:** Modify the radiance field based on the probability $p$. For example, set $\sigma=0$ or transform $\mathbf{c}$ for points where $p$ indicates the target object [S1].

**Formulas:**
*   **Feature Loss ($\mathcal{L}_f$):** Minimizes the distance between the rendered feature $\mathbf{f}$ and the teacher feature $f_{img}$ [S1].
*   **LSeg Probability (Eq. 22 in [S1]):** Probability of a point belonging to a text label is derived from the cosine similarity between the point's feature and the text-encoded label [S1].
*   **DINO Probability:** Hard thresholding of cosine similarity between point feature and patch feature [S1].

**Parameters:**
*   **Teacher Networks:** LSeg [44] or DINO [12] [S1].
*   **Iterations:** 200K for initial NeRF; 5K for finetuning [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** Linearly decaying from $5e^{-4}$ to $8e^{-5}$ for initial training; fixed at $1e^{-4}$ for finetuning [S1].
*   **Sampling:** Coarse: 64 points; Fine: 128 points (total 192 for standard NeRF) [S1].
*   **PE Length:** 10 (coordinate), 4 (direction) [S1].
*   **Image Size:** 320×240 (Replica); 1008×756 (others) [S1].
*   **Batch Size:** 1024 (Replica); 2048 (others) [S1].
*   **Feature Loss Weight ($\lambda$):** Default value not explicitly stated, but increasing by $\times 10$ degrades view synthesis [S1].

**Edge Cases and Pitfalls:**
*   **Feature Loss Convergence:** Feature loss converges significantly faster than photometric loss; thus, short finetuning (5K iterations) is sufficient [S1].
*   **High Feature Loss Weight:** Excessively increasing $\lambda$ (e.g., $\times 10$) hurts view synthesis quality without improving segmentation [S1].
*   **Naive NeRF Features:** Using hidden layers of the original NeRF (e.g., 8th layer) for decomposition is not robust; selections leak to regions with similar colors/geometry [S1].
*   **Deletion Artifacts:** Deleting foreground objects may result in floating artifacts and blurred volumes in the far distance behind the deleted object [S1].
*   **Smoothness vs. Detail:** Coarse sampling or removing PE yields smoother decompositions but lacks high-frequency detail; fine sampling introduces artifacts [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DFF:
    def __init__(self, teacher_network_type='LSeg', use_independent_feature_mlp=False):
        """
        Initialize DFF.
        :param teacher_network_type: 'LSeg' or 'DINO'
        :param use_independent_feature_mlp: If True, train separate MLP for features.
        """
        self.teacher_type = teacher_network_type
        self.use_independent = use_independent_feature_mlp
        
        # Radiance Field MLP: 8 ReLU(256) -> Linear(256, 1) [sigma]
        # -> Linear(256, 3) [color] -> Linear(256, 3) [feature]
        self.radiance_mlp = self._build_radiance_mlp()
        
        if self.use_independent:
            # Independent Feature MLP: 4 layers, skip connection at 3rd if PE used
            self.feature_mlp = self._build_feature_mlp()
        else:
            self.feature_mlp = None # Feature is output of radiance_mlp branch
            
        self.optimizer = None
        self.teacher = self._load_teacher()

    def _build_radiance_mlp(self):
        layers = []
        for _ in range(8):
            layers.append(nn.Linear(256, 256))
            layers.append(nn.ReLU())
        # Density
        layers.append(nn.Linear(256, 1))
        # Color
        layers.append(nn.Linear(256, 3))
        # Feature (if branch-based)
        layers.append(nn.Linear(256, 3))
        return nn.Sequential(*layers)

    def _build_feature_mlp(self):
        # 4 layers, skip connection at 3rd layer if PE is used
        # Assuming PE is used for independent MLP as per notes
        layers = []
        # Layer 1
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        # Layer 2
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        # Layer 3 with skip connection
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        # Layer 4
        layers.append(nn.Linear(256, 3)) # Feature output
        return nn.Sequential(*layers)

    def _load_teacher(self):
        if self.teacher_type == 'LSeg':
            return LSegTeacher()
        elif self.teacher_type == 'DINO':
            return DINOTeacher()
        else:
            raise ValueError("Unsupported teacher network")

    def positional_encoding(self, x, pe_length=10):
        # Standard NeRF positional encoding
        # x shape: (N, 3)
        # Returns shape: (N, 2 * pe_length * 3)
        freq_bands = 2.0 ** torch.linspace(0, pe_length - 1, pe_length).to(x.device)
        x = x.unsqueeze(-1) * freq_bands.unsqueeze(0)
        x = x.reshape(x.shape[0], -1)
        return torch.sin(x)

    def forward_radiance(self, x, v):
        """
        Forward pass for radiance field.
        :param x: (N, 3) coordinates
        :param v: (N, 3) viewing directions
        :return: sigma, color, feature
        """
        # Encode inputs
        x_enc = self.positional_encoding(x, pe_length=10)
        v_enc = self.positional_encoding(v, pe_length=4)
        
        # Concatenate x_enc and skip connection (x_enc)
        # Note: Standard NeRF concatenates x_enc with original x_enc for skip
        input_vec = torch.cat([x_enc, x_enc], dim=-1) # (N, 20) -> (N, 20) if PE len 10? 
        # Actually PE len 10 means 20 dims. Skip connection usually adds original coords or PE.
        # Based on "PE of length 10 for input coordinate and its skip connection", 
        # it implies the input to the first layer is PE(x) concatenated with PE(x) or similar.
        # Let's assume standard NeRF input: PE(x) concatenated with PE(x) is not standard.
        # Standard: PE(x) is input. Skip connection adds PE(x) to later layers.
        # The note says "PE of length 10 for input coordinate and its skip connection".
        # This likely means the input vector to the MLP is PE(x) (20 dims) and skip connections add PE(x).
        
        # Simplified forward for implementation clarity based on "8 ReLU layers... followed by linear"
        h = x_enc # Initial input
        
        for i, layer in enumerate(self.radiance_mlp[:8]): # First 8 layers (ReLU)
            if i > 0 and i % 2 == 1: # Every other layer is ReLU in the list? 
                # The list has 8 Linear+ReLU pairs.
                pass
            h = layer(h)
            # Skip connection logic would go here in a full implementation
            # For brevity, assuming the MLP structure handles it or h is updated sequentially
        
        # The note says "8 ReLU layers with 256 dimensions, followed by a linear layer for density..."
        # This implies the architecture is:
        # Linear(256,256)->ReLU ... 8 times.
        # Then Linear(256,1) for sigma.
        # Then Linear(256,3) for color.
        # Then Linear(256,3) for feature.
        
        # Let's reconstruct the forward pass more accurately
        h = x_enc
        for i in range(8):
            h = self.radiance_mlp[i*2](h) # Linear
            h = self.radiance_mlp[i*2+1](h) # ReLU
            # Skip connection: h = h + x_enc (if i is even/odd as per standard NeRF)
            # Standard NeRF: skip at layers 4, 8, 12, 16.
            # Here we have 8 layers.
            if i in [3, 7]: # Approximate skip connection points
                 h = h + x_enc 
        
        # Density
        sigma = self.radiance_mlp[16](h)
        
        # Color
        color = self.radiance_mlp[17](h)
        
        # Feature
        feature = self.radiance_mlp[18](h)
        
        return sigma, color, feature

    def compute_feature_loss(self, rendered_feature, teacher_feature_map):
        """
        Compute feature distillation loss.
        :param rendered_feature: (N, 3)
        :param teacher_feature_map: (N, 3) resized to image size
        """
        return F.mse_loss(rendered_feature, teacher_feature_map)

    def train_step(self, rays, teacher_features):
        """
        One training step.
        :param rays: (B, 6) [x, y, z, vx, vy, vz]
        :param teacher_features: (B, 3) target features
        """
        x = rays[:, :3]
        v = rays[:, 3:]
        
        sigma, color, feature = self.forward_radiance(x, v)
        
        # Photometric loss (assuming ground truth color is in rays or separate)
        # gt_color = rays[:, 6:9] # Placeholder
        # loss_p = F.mse_loss(color, gt_color)
        
        # Feature loss
        loss_f = self.compute_feature_loss(feature, teacher_features)
        
        # Combined loss
        lambda_f = 1.0 # Default, tune as needed
        loss = loss_f # + lambda_f * loss_p
        
        self.optimizer.zero_grad()
        loss.backward()
        self.optimizer.step()
        
        return loss.item()

    def query_decomposition(self, points_3d, query):
        """
        Decompose scene based on query.
        :param points_3d: (N, 3)
        :param query: Text string (LSeg) or Patch Image (DINO)
        :return: probabilities p
        """
        features = self.get_features(points_3d)
        
        if self.teacher_type == 'LSeg':
            teacher_label_feat = self.teacher.encode_text(query)
            # Cosine similarity
            sim = F.cosine_similarity(features, teacher_label_feat.unsqueeze(0), dim=-1)
            # Eq 22 logic: probability derived from similarity
            p = torch.sigmoid(sim) # Simplified
        elif self.teacher_type == 'DINO':
            teacher_patch_feat = self.teacher.encode_patch(query)
            sim = F.cosine_similarity(features, teacher_patch_feat.unsqueeze(0), dim=-1)
            # Hard decomposition
            threshold = 0.5 # Example threshold
            p = (sim > threshold).float()
            
        return p

    def get_features(self, x):
        if self.use_independent:
            x_enc = self.positional_encoding(x, pe_length=10)
            return self.feature_mlp(x_enc)
        else:
            # Get from radiance field
            # Requires dummy v
            v = torch.zeros_like(x)
            _, _, f = self.forward_radiance(x, v)
            return f
```

## Subtopics

### Branch-based DFF
The feature branch shares layers with the radiance field MLP. It does not degrade view synthesis performance compared to NeRF without feature fields and has small computational overhead [S1].

### Independent Feature Field DFF
Uses a separate, lightweight MLP for the feature field. Achieves competitive segmentation results and allows introducing DFF into arbitrary 3D scene representations (e.g., dynamic NeRFs, meshes) without retraining the radiance field [S1].

### LSeg-DFF
Uses LSeg as the teacher network. Enables zero-shot semantic segmentation using text queries. The text encoder encodes labels, and point probabilities are calculated via cosine similarity [S1].

### DINO-DFF
Uses DINO as the teacher network. Enables query-based decomposition using image patch queries. Uses thresholded cosine similarity for hard decomposition ($p=1$ or $0$) [S1].

## Operating characteristics
*   **Time Complexity:** Training requires 200K iterations for the base NeRF and 5K for finetuning. The feature loss converges significantly faster than photometric loss [S1].
*   **Memory Complexity:** Branch-based DFF adds minimal overhead. Independent feature field adds a small MLP (4 layers) [S1].
*   **Performance:** DFF achieves 3D semantic segmentation performance (mIoU) better than supervised sparse 3D convolution models like MinkowskiNet42 on the Replica dataset [S1].
*   **Strengths:** Multi-view consistent editing and deletion; successful distillation from 2D teachers; smooth volume decomposition with coarse sampling or no-PE [S1].
*   **Limitations:** Deleting foreground objects can leave floating artifacts and blurred volumes in the far distance; naive use of NeRF hidden features fails for decomposition; excessive feature loss weight hurts view synthesis [S1].
*   **When to use:** When 3D semantic segmentation and editable novel-view synthesis are required without 3D semantic labels; when multi-view consistency is critical [S1].
*   **When not to use:** When high-frequency detail in semantic boundaries is required (use fine sampling with caution); when the feature loss weight cannot be tuned carefully [S1].

## Background
DFF was introduced in the paper "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It addresses the challenges of editable NeRFs, including surface decomposition, volume decomposition, lighting decomposition, and estimation of unobserved parts [S1]. The method leverages pre-trained 2D teacher networks (LSeg and DINO) to distill semantic information into the 3D radiance field, enabling zero-shot semantic segmentation and editing [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
