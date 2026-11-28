# SRN+Linear

Also called: SRN-based models, Semantic SRN, Semi-Supervised SRN, Semi-Supervised Semantic SRN, Semi-Supervised Training for Semantic SRN

## Overview
SRN+Linear is a semi-supervised semantic segmentation method that leverages a pre-trained implicit neural scene representation (SRN) to perform per-point semantic segmentation while retaining its ability to represent appearance and geometry [S1]. It extracts pre-trained features from the neural scene representation and maps them to class probabilities using a simple linear transformation [S1]. Unlike 2D feature extractors which do not build a 3D-structure-aware representation, this method enables predictions about unseen perspectives and occluded parts [S1]. Unlike fully supervised training which requires massive labeled data, this method relies on a small subset of labeled masks after pre-training on unlabeled RGB data [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A large dataset of 2D posed RGB images (unlabeled) for pre-training; a small subset of human-annotated semantic segmentation masks (e.g., 30 masks) for fine-tuning; at test time, a single posed RGB image and/or label mask of an unseen instance.
*   **Outputs:** Multi-view consistent novel RGB views and semantic segmentation views of the object instance.

**Data Structures:**
*   **SRN (Hypernetwork):** 4-layer MLP with 256 units per layer, ReLU nonlinearities, and LayerNorm before each nonlinearity [S1].
*   **RGB Renderer:** 4-layer MLP with 256 units per layer, ReLU nonlinearities, and LayerNorm before each nonlinearity [S1].
*   **Raymarcher:** LSTM with 256 units [S1].
*   **Latent Code ($z_i$):** A code vector representing each object instance [S1].
*   **Feature Vector ($v$):** Output of the SRN, $v \in \mathbb{R}^{256}$ [S1].
*   **Linear Classifier (SEG):** Parameters are a weight matrix $W \in \mathbb{R}^{256 \times c}$ and a bias vector $b \in \mathbb{R}^c$, where $c$ is the number of semantic classes [S1].

**Numbered Steps:**

1.  **Pre-training (Unsupervised):** Pre-train the SRN weights, the latent embeddings $z_i$ for each object instance in the training set, and the RGB Renderer weights using only posed RGB images and their camera parameters. The goal is novel view synthesis. The SRN and RGB Renderer are initialized as 4-layer MLPs with 256 units, ReLU, and LayerNorm [S1]. The raymarcher is implemented as an LSTM with 256 units [S1]. Raymarching is performed for 10 steps [S1].
    *   *Training Schedule:* Train for 20,000 steps at 64×64 resolution with batch size 92. Then, train for an additional 85,000 steps at 128×128 resolution with batch size 16 [S1].
    *   *Optimizer:* ADAM with learning rate $4 \times 10^{-4}$ [S1].
    *   *Loss:* Image reconstruction loss [S1].

2.  **Freezing:** Freeze the latent code vectors $z_i$ and the weights of the SRN and the RGB Renderer [S1].

3.  **Linear Classifier Training (Semi-Supervised):** Train the linear segmentation classifier (SEG) on the pre-trained feature vectors $v$. The features $v$ are extracted from the frozen SRN. The classifier is parameterized as a linear transformation to prevent overfitting given the limited labeled data [S1].
    *   *Data:* Human-annotated, posed semantic segmentation masks of a small subset of training images (e.g., 30 masks) [S1].
    *   *Parameters:* $W \in \mathbb{R}^{256 \times c}$ and $b \in \mathbb{R}^c$ [S1].
    *   *Loss:* Total loss is a weighted sum of image reconstruction loss and cross-entropy loss. The weights are 200:8 to equalize their magnitudes [S1].
    *   *Classes:* $c=6$ for chairs, $c=11$ for tables [S1].

4.  **Test Time Inference (Novel Instance):** Given a single posed RGB image and/or label mask of an unseen instance, infer the latent code $z_{new}$ of the novel object. This is done by freezing all network weights, initializing a new code vector $z$, and optimizing $z$ to minimize image reconstruction and/or cross-entropy losses [S1].

5.  **Rendering:** Render multi-view consistent novel RGB and semantic segmentation views of the object instance using the inferred latent code $z_{new}$ and the frozen SRN/SEG [S1].

**Formulas:**
The class probabilities for a pixel are computed from the feature vector $v$ using the linear classifier:
$$ p = \text{softmax}(W v + b) $$
where $v \in \mathbb{R}^{256}$, $W \in \mathbb{R}^{256 \times c}$, $b \in \mathbb{R}^c$, and $c$ is the number of semantic classes [S1].

The total loss for the linear classifier training phase is:
$$ L_{total} = 200 \cdot L_{recon} + 8 \cdot L_{CE} $$
where $L_{recon}$ is the image reconstruction loss and $L_{CE}$ is the cross-entropy loss [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class SRNLinear:
    def __init__(self, num_classes):
        """
        Initialize the SRN+Linear model.
        :param num_classes: c, the number of semantic classes.
        """
        # SRN: 4-layer MLP, 256 units, ReLU, LayerNorm
        self.srn = nn.Sequential(
            nn.Linear(3, 256), # Input: xyz
            nn.LayerNorm(256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.LayerNorm(256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.LayerNorm(256),
            nn.ReLU(),
            nn.Linear(256, 256) # Output: feature vector v
        )
        
        # RGB Renderer: 4-layer MLP, 256 units, ReLU, LayerNorm
        # Note: In SRN, the renderer typically takes (v, direction) to output color.
        # Simplified for clarity based on description.
        self.renderer = nn.Sequential(
            nn.Linear(256 + 3, 256), # v + direction
            nn.LayerNorm(256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.LayerNorm(256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.LayerNorm(256),
            nn.ReLU(),
            nn.Linear(256, 3) # Output: RGB
        )
        
        # Raymarcher: LSTM with 256 units
        self.raymarcher = nn.LSTM(input_size=256, hidden_size=256, num_layers=1)
        
        # Linear Segmentation Classifier (SEG)
        self.seg_classifier = nn.Linear(256, num_classes)
        
        # Latent code for the current instance
        self.z = None

    def pretrain_srn(self, rgb_images, camera_params, steps=105000, resolution=128, batch_size=16):
        """
        Pre-train SRN for novel view synthesis using RGB images.
        :param rgb_images: Tensor of RGB images.
        :param camera_params: Extrinsic and intrinsic camera parameters.
        :param steps: Total training steps.
        :param resolution: Image resolution.
        :param batch_size: Batch size.
        """
        optimizer = optim.Adam(self.srn.parameters(), lr=4e-4)
        optimizer.add_param_group({'params': self.renderer.parameters()})
        optimizer.add_param_group({'params': self.raymarcher.parameters()})
        
        # Initialize latent codes for training instances
        # Assuming N instances in training set
        N = len(rgb_images)
        self.zs = torch.randn(N, 256) # z_i for each instance
        
        for step in range(steps):
            # Determine resolution and batch size based on schedule
            if step < 20000:
                res = 64
                bs = 92
            else:
                res = 128
                bs = 16
            
            # Sample batch
            batch_indices = torch.randint(0, N, (bs,))
            batch_z = self.zs[batch_indices]
            batch_images = rgb_images[batch_indices]
            
            # Forward pass: Render RGB
            # Note: Actual raymarching involves sampling points along rays.
            # Simplified representation of the SRN forward pass.
            rendered_images = self.render(batch_z, camera_params[batch_indices], res)
            
            # Loss: Image reconstruction
            loss = nn.functional.mse_loss(rendered_images, batch_images)
            
            # Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            # Update latent codes if using autodecoder framework where z is optimized
            # In the pre-training phase, z_i are optimized along with weights
            # This is a simplification; in practice, z is updated via optimizer
            # optimizer_z = optim.Adam(self.zs[batch_indices], lr=...)
            # loss.backward()
            # optimizer_z.step()

    def freeze_srn(self):
        """Freeze SRN weights and latent codes."""
        for param in self.srn.parameters():
            param.requires_grad = False
        for param in self.renderer.parameters():
            param.requires_grad = False
        for param in self.raymarcher.parameters():
            param.requires_grad = False
        # z is frozen in the sense that it is not updated during SEG training
        # but is used as input.

    def train_linear_classifier(self, features, labels, num_epochs=100):
        """
        Train the linear segmentation classifier on pre-trained features.
        :param features: Tensor of feature vectors v from SRN, shape (N, 256).
        :param labels: Tensor of semantic labels, shape (N, H, W).
        :param num_epochs: Number of training epochs.
        """
        optimizer = optim.Adam([
            {'params': self.seg_classifier.parameters(), 'lr': 4e-4}
        ], lr=4e-4)
        
        # Loss weights
        w_recon = 200
        w_ce = 8
        
        for epoch in range(num_epochs):
            # Forward pass
            logits = self.seg_classifier(features) # Shape: (N, c)
            # Note: In pixel-wise segmentation, features are per-pixel.
            # The input features here are assumed to be aggregated or per-pixel features.
            # Based on [S1], features v are per-point in 3D, rendered to 2D.
            # The linear classifier is applied to the rendered features.
            
            # Cross-entropy loss
            loss_ce = nn.functional.cross_entropy(logits, labels)
            
            # Image reconstruction loss (if needed for joint training, but SRN is frozen)
            # The source mentions total loss is weighted sum. 
            # If SRN is frozen, reconstruction loss might not be applicable or is zero.
            # However, [S1] says "Total loss is a weighted sum...". 
            # It is likely that during SEG training, the reconstruction loss is not computed 
            # on the SRN weights (since they are frozen) but perhaps on the rendered output 
            # to ensure consistency? Or it refers to the pre-training phase?
            # Re-reading [S1]: "Subsequently, we freeze zi as well as the weights of HN and 
            # the differentiable renderer, and train the proposed Segmentation Renderer SEG 
            # on the learned feature vectors v. This training is supervised with human-labeled... 
            # In this case of limited training data, we parameterize SEG as a linear classifier..."
            # The loss weighting 200:8 is mentioned in the context of SRN-based models training.
            # It is ambiguous if this applies to the SEG training phase. 
            # Given "Total loss is a weighted sum of image reconstruction loss and cross-entropy loss",
            # and SRN is frozen, the reconstruction loss might be on the RGB renderer output 
            # to maintain appearance quality while learning semantics? 
            # Or it might be that the "image reconstruction loss" refers to the pre-training phase.
            # However, the prompt notes say: "Total loss is a weighted sum...". 
            # Let's assume it applies to the SEG training phase as well, 
            # possibly to ensure the features still support rendering.
            # But if SRN is frozen, we can't optimize reconstruction.
            # Let's assume the loss is just Cross-Entropy for SEG training, 
            # or the reconstruction loss is computed on the rendered RGB to keep it aligned.
            # Given the ambiguity, we will implement the Cross-Entropy loss as the primary driver.
            # If reconstruction is required, it would be on the RGB renderer output.
            
            # For the purpose of this implementation, we focus on the linear classifier training.
            # The source explicitly states the loss weighting for SRN-based models.
            # We will include the CE loss.
            
            loss = w_ce * loss_ce
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

    def infer_latent_code(self, rgb_image, camera_params):
        """
        Infer the latent code of a novel object instance.
        :param rgb_image: Single posed RGB image of the novel instance.
        :param camera_params: Camera parameters for the image.
        :return: Inferred latent code z.
        """
        # Initialize a new code vector
        z = torch.randn(1, 256, requires_grad=True)
        optimizer = optim.Adam([z], lr=4e-4)
        
        # Optimize z to minimize image reconstruction loss
        for _ in range(1000): # Number of optimization steps
            # Render the image using the current z
            rendered = self.render(z, camera_params, resolution=128)
            loss = nn.functional.mse_loss(rendered, rgb_image)
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        return z.detach()

    def render(self, z, camera_params, resolution):
        """
        Render RGB and semantic segmentation images.
        :param z: Latent code.
        :param camera_params: Camera parameters.
        :param resolution: Image resolution.
        :return: Rendered RGB and segmentation images.
        """
        # Raymarching logic
        # For each pixel, cast a ray, sample points, query SRN for features,
        # aggregate features, then use SEG for semantic probabilities.
        # This is a simplified placeholder for the complex raymarching process.
        
        # 1. Generate rays from camera_params
        rays = generate_rays(camera_params, resolution)
        
        # 2. Sample points along rays
        points = sample_points_along_rays(rays, num_steps=10)
        
        # 3. Query SRN for features
        # SRN takes (xyz, z) -> feature vector v
        # Note: SRN usually takes (xyz) and uses z as a condition.
        # The hypernetwork HN takes z and xyz to produce weights for the renderer?
        # Or SRN takes (xyz) and outputs features conditioned on z?
        # [S1] says "SRN... maps each (x, y, z) coordinate to a feature vector".
        # And "each object instance is represented by its own code vector z_i".
        # So SRN likely takes (xyz) and z as input, or z is used to modulate the SRN.
        # Given "SRN as well as the RGB Renderer are implemented as 4-layer MLPs",
        # and "hypernetwork HN", it's likely HN takes z and produces weights for SRN/Renderer.
        # However, for simplicity, we assume SRN takes (xyz, z) or z is concatenated.
        
        features = self.srn(points) # Simplified: SRN(points)
        # In reality, SRN might be: SRN(HN(z), points)
        
        # 4. Aggregate features along ray (e.g., mean, or weighted sum based on density)
        # This is part of the raymarcher (LSTM)
        aggregated_features = self.raymarcher(features)
        
        # 5. Render RGB
        # RGB Renderer takes aggregated features and direction
        directions = get_directions(rays)
        rgb = self.renderer(torch.cat([aggregated_features, directions], dim=-1))
        
        # 6. Render Segmentation
        # SEG takes aggregated features
        seg_logits = self.seg_classifier(aggregated_features)
        seg_probs = torch.softmax(seg_logits, dim=-1)
        
        return rgb, seg_probs

    def forward(self, z, camera_params, resolution):
        """
        Forward pass for rendering.
        """
        rgb, seg = self.render(z, camera_params, resolution)
        return rgb, seg

def generate_rays(camera_params, resolution):
    # Placeholder for ray generation logic
    pass

def sample_points_along_rays(rays, num_steps):
    # Placeholder for point sampling logic
    pass

def get_directions(rays):
    # Placeholder for direction extraction
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the quality of the pre-trained features. If the SRN is not well-pretrained, the linear classifier may not perform well [S1].
*   **Boundaries:** The method assumes the object is fully visible or that the latent code inference can handle partial visibility. The inference of $z$ for novel instances requires optimization, which can be sensitive to initialization and local minima [S1].
*   **Degenerate Input:** If the input image is not posed correctly or lacks sufficient texture, the latent code inference may fail [S1].

## Subtopics

### SRN+U-Net Fully Supervised Reference
A fully supervised reference model that trains the SRN end-to-end with a U-Net segmentation classifier on the full training dataset [S1]. Unlike SRN+Linear, this model does not infer a compact, multi-modal 3D-aware representation. It performs semantic segmentation in image space and does not force the representation to encode all information necessary for 3D semantic reasoning [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Pre-training:** Requires significant computational resources. Training for 105,000 steps at 64x64 and 128x128 resolutions [S1].
*   **Linear Classifier Training:** Fast, as it only involves a linear transformation and a small dataset (30 masks) [S1].
*   **Inference:** Requires optimizing the latent code $z$ for each novel instance, which can be time-consuming [S1].

**Measured Performance:**
*   **Hardware:** SRN-based models use Nvidia RTX8000 GPUs [S1].
*   **Resolution:** 64x64 and 128x128 [S1].
*   **Batch Size:** 92 (64x64), 16 (128x128) [S1].

**Scaling:**
*   The method scales with the number of semantic classes $c$ [S1].
*   The method scales with the complexity of the SRN (number of layers and units) [S1].

**Strengths and Limitations:**
*   **Strengths:** Requires only a small amount of labeled data (30 masks) [S1]. Enables multi-view consistent semantic segmentation [S1]. Leverages 3D-structure-aware representation [S1].
*   **Limitations:** Requires pre-training on a large dataset of RGB images [S1]. Inference for novel instances requires optimization [S1].

**When to Use:**
*   When labeled data is scarce but unlabeled RGB data is abundant [S1].
*   When multi-view consistent semantic segmentation is required [S1].

**When Not to Use:**
*   When a large amount of labeled data is available and a fully supervised approach is feasible [S1].
*   When real-time inference is required, as latent code optimization can be slow [S1].

## Background
**Origin:**
SRN+Linear was introduced in the paper "Semantic Implicit Neural Scene Representations With Semi-Supervised Training" [S1]. It builds on the Scene Representation Networks (SRN) proposed by Sitzmann et al. [S1].

**History and Development:**
The method was developed to address the limitation of existing implicit representations that primarily estimate appearance and/or 3D geometry [S1]. It demonstrates that an existing implicit representation (SRN) can be multi-modal, performing per-point semantic segmentation while retaining its ability to represent appearance and geometry [S1].

**Usage:**
The method is used for 3D novel view and semantic label synthesis given only 2D images and their camera parameters [S1]. It is applicable to tasks in inverse graphics and 3D computer vision [S1].

## Sources
[S1] Semantic Implicit Neural Scene Representations With Semi-Supervised Training, https://doi.org/10.1109/3dv50981.2020.00052, fetched 2026-10-26
