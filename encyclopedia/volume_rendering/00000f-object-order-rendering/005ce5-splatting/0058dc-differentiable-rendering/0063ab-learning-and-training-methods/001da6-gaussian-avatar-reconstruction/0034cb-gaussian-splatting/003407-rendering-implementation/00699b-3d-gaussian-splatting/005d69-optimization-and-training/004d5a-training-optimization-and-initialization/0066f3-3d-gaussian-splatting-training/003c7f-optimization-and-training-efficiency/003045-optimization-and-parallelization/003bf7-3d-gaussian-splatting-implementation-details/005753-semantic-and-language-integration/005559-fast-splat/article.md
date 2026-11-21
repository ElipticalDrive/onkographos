# FAST-Splat

Also called: Fast, Ambiguity-Free Semantic Transfer in Gaussian Splatting

## Overview
FAST-Splat is a semantic Gaussian Splatting method that directly augments each 3D Gaussian with specific semantic codes and uses a hash-table for disambiguation, avoiding neural fields for semantic distillation to enable fast training, rendering, and ambiguity-free object localization [S1, unofficial]. It sets itself apart from siblings like [LangSplat Training Procedure](12438-langsplat-training-procedure.md) and [Feature 3D Gaussian Splatting](15902-feature-3d-gaussian-splatting.md) by employing a single-phase, neural-free training procedure that jointly optimizes geometric, visual, and semantic attributes, thereby preserving the speed benefits of Gaussian Splatting while resolving semantic ambiguity in open-vocabulary queries [S1, unofficial].

## How it works
FAST-Splat builds on [Semantic and Language Integration](15930-semantic-and-language-integration.md). It extends the standard 3D Gaussian parameter set $\Theta_i = \{x_i, q_i, s_i, \alpha_i, c_i\}$ with a semantic attribute $\beta_i$ [15902-feature-3d-gaussian-splatting.md, S1, unofficial]. During rendering, this attribute is accumulated alongside color using the standard front-to-back alpha blending equation [15902-feature-3d-gaussian-splatting.md]:
$$ F_s = \sum_{i \in \mathcal{N}} f_i \alpha_i T_i $$
where $F_s$ is the rendered semantic feature, $\alpha_i$ is opacity, and $T_i$ is transmittance [15902-feature-3d-gaussian-splatting.md]. Training involves a joint loss function combining photometric reconstruction (L1/SSIM) and a semantic distillation or classification loss [S1, 15902-feature-3d-gaussian-splatting.md, unofficial].

### Inputs and Outputs
**Inputs:**
*   A dataset of RGB images $\mathcal{D}$.
*   (Optional) Closed-set object detector (e.g., YOLO, DETR) for initial object lists.
*   Open-set object detector (e.g., GroundingDINO) and segmentation model (e.g., SAM-2) for dense semantic maps.
*   Pre-trained text encoder (e.g., CLIP, BERT) for embedding generation.

**Outputs:**
*   Optimized Gaussian Splatting model with geometric, visual, and semantic attributes.
*   Hash-table of identified objects and their text embeddings.
*   Rendered semantic feature maps and probability distributions over the hash-table entries.

### Data Structures
*   **Gaussian Attributes:** Each Gaussian $i$ stores:
    *   Mean $x_i \in \mathbb{R}^3$.
    *   Rotation quaternion $q_i \in \mathbb{R}^4$.
    *   Scale $s_i \in \mathbb{R}^3$.
    *   Opacity $\alpha_i \in \mathbb{R}$.
    *   Spherical Harmonics coefficients for color $c_i$.
    *   Semantic code $\beta_i \in \mathbb{R}^3$ [S1, unofficial].
*   **Affine Transformation:** A learnable matrix $W$ used to map $\beta_i$ to the semantic space [S1, unofficial].
*   **Hash-Table (Dictionary):** A mapping from object class indices to:
    *   Object class labels.
    *   Pre-computed text embeddings $\theta_d$ [S1, unofficial].

### Numbered Steps

1.  **Initialization:** Initialize the Gaussian Splatting model (means, covariances, opacities, SH parameters) from a sparse point cloud obtained via structure-from-motion [S1, unofficial].
2.  **Semantic Map Generation:**
    *   Feed images into an object detector (e.g., YOLO, DETR, Efficient DETR) to generate object lists and bounding boxes [S1, unofficial].
    *   Generate dense semantic maps using SAM-2, passing each object class and its bounding box [S1, unofficial].
    *   Maintain a dictionary (hash-table) of identified objects [S1, unofficial].
    *   Augment object classes with an undetected class label and assign lower weights to pixels with this label to serve as a regularization term for stability [S1, unofficial].
3.  **Text Embedding Pre-computation:** Pre-compute text embeddings $\theta_d$ for dictionary entries using a pre-trained text encoder (e.g., CLIP, BERT) without fine-tuning [S1, unofficial].
4.  **Joint Optimization (Single-Phase):**
    *   Simultaneously optimize all attributes (geometric, visual, and semantic $\beta$) and the affine transformation $W$ in a single phase using gradient descent with the Adam optimizer [S1, unofficial].
    *   For each image $I \in \mathcal{D}$:
        *   Render the semantic feature map using differentiable tile-based rasterization for the given camera pose [S1, unofficial].
        *   Transform each pixel's semantic feature to a probability distribution over the hash-table entries by applying an affine transformation $W$ to $\beta$ and then softmax [S1, unofficial].
        *   Compute the loss $L_{sgs} = L_{gs} + L_{ce}$ (see Formulas).
        *   Update parameters using backpropagation [S1, unofficial].
5.  **Runtime Query Processing (Open-Vocabulary):**
    *   Compute text embedding $\theta_q$ for the user's natural-language prompt using the pre-trained text encoder [S1, unofficial].
    *   Compute relevancy score as pairwise softmax between cosine-similarity values of dictionary embeddings $\theta_d$ and query embedding $\theta_q$ (positive) plus canonical negative embeddings [S1, unofficial].
    *   Designate entries exceeding a relevancy score threshold as similar to the query [S1, unofficial].
    *   Map these entries to pixels in the rendered feature map using the softmax probability distribution [S1, unofficial].
    *   Assign a label to each pixel by taking the maximum probability value, or use alternative techniques like conformal prediction [S1, unofficial].
    *   Provide a clarifying semantic label to each localized object to disambiguate the semantic identity [S1, unofficial].

### Formulas
*   **Gaussian Splatting Loss:**
    $$ L_{gs} = (1 - \lambda) \sum_{I \in \mathcal{D}} ||I - \hat{I}|| + \lambda L_{D-SSIM} $$
    where $\lambda \in [0, 1]$ balances the L1-loss and D-SSIM loss, $I$ is the ground-truth image, and $\hat{I}$ is the predicted image [S1, unofficial].
*   **Total Loss:**
    $$ L_{sgs} = L_{gs} + L_{ce} $$
    [S1, unofficial]
*   **Cross-Entropy Loss:**
    $$ L_{ce} = - \sum_{I \in \mathcal{D}} \frac{1}{|T|} \sum_{p \in T} \gamma_p \log \frac{\exp(F_{I,p,c})}{\sum_{j=1}^{N} \exp(F_{I,p,j})} $$
    where $T$ is the set of all indices of pixels in the rendered feature map $F_I$ associated with image $I$, $c$ is the true class of the pixel, $\gamma_p$ is the weight associated with pixel $p$, and $N$ is the size of the dictionary [S1, unofficial].
*   **Semantic Probability:**
    For each pixel, the probability over dictionary entries is computed by applying softmax to $W \beta$ [S1, unofficial].

### Parameters
*   $\lambda \in [0, 1]$: Weighting parameter between L1-loss and D-SSIM loss [S1, unofficial].
*   $\beta \in \mathbb{R}^3$: Semantic parameter for each ellipsoid [S1, unofficial].
*   $W$: Affine transformation matrix [S1, unofficial].
*   Relevancy score threshold: Threshold for determining similar objects in open-vocabulary queries [S1, unofficial].

### Edge Cases and Pitfalls
*   **Ambiguity:** Ambiguity in object localization arises when users query with semantically similar but distinct objects (e.g., 'tea' vs 'coffee machine'); existing methods localize the object but fail to provide clarifying identity information [S1, unofficial]. FAST-Splat resolves this by providing the precise semantic label [S1, unofficial].
*   **Bag-of-Words Limitation:** Vision-language foundation methods like CLIP use a bag-of-words approach for semantic similarity, which contributes to ambiguity; FAST-Splat mitigates this via the hash-table and relevancy scoring [S1, unofficial].
*   **Undetected Pixels:** Pixels not segmented during data pre-processing are assigned an undetected class label with lower weights to ensure stability [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FASTSplat:
    def __init__(self, num_classes, hidden_dim=3):
        """
        Initialize FAST-Splat model.
        :param num_classes: N, size of the semantic dictionary/hash-table.
        :param hidden_dim: Dimension of semantic code beta (default 3).
        """
        self.num_classes = num_classes
        self.hidden_dim = hidden_dim
        
        # Learnable affine transformation W mapping beta to N-dimensional space
        self.W = nn.Linear(hidden_dim, num_classes)
        
        # Pre-trained text encoder (e.g., CLIP) - assumed to be loaded externally
        self.text_encoder = None 
        
        # Dictionary of text embeddings for each class
        self.class_embeddings = None

    def initialize_gaussians(self, point_cloud):
        """
        Initialize Gaussian attributes from a sparse point cloud.
        :param point_cloud: List of 3D points.
        :return: Gaussian parameters (means, covariances, opacities, SH, betas).
        """
        # [G] Standard 3DGS initialization logic would go here.
        # We assume a function that returns a GaussianModel object.
        gaussians = GaussianModel(point_cloud)
        # Initialize semantic codes beta randomly or from a prior
        gaussians.betas = torch.randn(gaussians.num_gaussians, self.hidden_dim)
        return gaussians

    def generate_semantic_maps(self, images, detector, segmenter):
        """
        Generate dense semantic maps and build the hash-table.
        :param images: List of input images.
        :param detector: Open-set object detector (e.g., GroundingDINO).
        :param segmenter: Segmentation model (e.g., SAM-2).
        :return: List of semantic maps and the hash-table (dictionary of classes).
        """
        semantic_maps = []
        class_dict = {}
        
        for img in images:
            # 1. Detect objects
            objects = detector(img) # Returns list of (class_name, bbox)
            
            # 2. Generate dense semantic maps using SAM-2
            semantic_map = torch.zeros_like(img, dtype=torch.long)
            for class_name, bbox in objects:
                if class_name not in class_dict:
                    class_dict[class_name] = len(class_dict)
                class_idx = class_dict[class_name]
                # [G] SAM-2 segmentation logic to fill semantic_map with class_idx
                semantic_map = segmenter(img, bbox, class_name, semantic_map)
            
            # 3. Augment with undetected class
            undetected_idx = len(class_dict)
            class_dict['undetected'] = undetected_idx
            # [G] Mask pixels not covered by any detection as undetected
            semantic_map[semantic_map == 0] = undetected_idx
            
            semantic_maps.append(semantic_map)
            
        return semantic_maps, class_dict

    def precompute_text_embeddings(self, class_dict):
        """
        Pre-compute text embeddings for dictionary entries using CLIP/BERT.
        :param class_dict: Dictionary mapping class names to indices.
        """
        self.class_embeddings = []
        for class_name in class_dict.keys():
            # [G] Use pre-trained text encoder to get embedding
            emb = self.text_encoder.encode(class_name)
            self.class_embeddings.append(emb)
        self.class_embeddings = torch.stack(self.class_embeddings)

    def render_semantic_feature(self, gaussians, camera_pose):
        """
        Render semantic feature map using differentiable tile-based rasterization.
        :param gaussians: GaussianModel with betas and W.
        :param camera_pose: Camera parameters.
        :return: Rendered semantic feature map F_I.
        """
        # [G] Standard 3DGS rasterization logic.
        # For each pixel, accumulate semantic features based on alpha and transmittance.
        # F_s = sum(beta_i * alpha_i * T_i)
        # Then apply affine transformation W and softmax.
        
        # Get raw semantic features from Gaussians
        raw_features = gaussians.betas # Shape: (num_gaussians, hidden_dim)
        
        # Apply affine transformation W
        transformed_features = self.W(raw_features) # Shape: (num_gaussians, num_classes)
        
        # [G] Rasterize transformed_features using alpha blending
        # This produces a tensor of shape (H, W, num_classes)
        rendered_features = rasterize(transformed_features, gaussians, camera_pose)
        
        return rendered_features

    def compute_loss(self, rendered_features, semantic_maps, class_dict):
        """
        Compute the total loss L_sgs = L_gs + L_ce.
        :param rendered_features: Rendered semantic feature maps (logits before softmax).
        :param semantic_maps: Ground truth semantic maps.
        :param class_dict: Dictionary mapping class names to indices.
        :return: Total loss.
        """
        # 1. Photometric Loss L_gs (Standard 3DGS loss)
        # [G] Compute L1 and D-SSIM between rendered RGB and ground truth RGB
        l_gs = compute_photometric_loss(rendered_rgb, gt_rgb)
        
        # 2. Semantic Cross-Entropy Loss L_ce
        # Apply softmax to rendered_features to get probabilities
        probs = F.softmax(rendered_features, dim=-1)
        
        ce_loss = 0.0
        for feat_map, sem_map in zip(rendered_features, semantic_maps):
            # [G] Iterate over pixels, handle undetected class with lower weight
            for p in pixels:
                true_class = sem_map[p]
                # [G] Get weight gamma_p (lower for undetected)
                gamma_p = get_pixel_weight(true_class, class_dict)
                # Cross-entropy for this pixel
                ce_loss += gamma_p * F.cross_entropy(feat_map[p], true_class)
        
        # Normalize by number of pixels
        ce_loss /= num_pixels
        
        return l_gs + ce_loss

    def query(self, query_text, rendered_features, class_dict):
        """
        Perform open-vocabulary query and disambiguation.
        :param query_text: User's natural language prompt.
        :param rendered_features: Rendered semantic feature maps.
        :param class_dict: Dictionary mapping class names to indices.
        :return: Disambiguated labels for localized pixels.
        """
        # 1. Compute query embedding
        query_emb = self.text_encoder.encode(query_text)
        
        # 2. Compute relevancy scores
        # Cosine similarity between query_emb and class_embeddings
        cos_sim = F.cosine_similarity(query_emb.unsqueeze(0), self.class_embeddings, dim=1)
        
        # [G] Compute pairwise softmax with canonical negative embeddings
        # relevancy_score = softmax(cos_sim)
        relevancy_scores = F.softmax(cos_sim, dim=0)
        
        # 3. Thresholding
        threshold = 0.5 # Example threshold
        similar_classes = [class_name for class_name, score in zip(class_dict.keys(), relevancy_scores) if score > threshold]
        
        # 4. Map to pixels
        # [G] For each pixel, take the max probability among similar classes
        # Or use conformal prediction
        labels = []
        for feat_map in rendered_features:
            # [G] Filter probabilities for similar classes
            # Assign label based on max probability
            label = assign_label(feat_map, similar_classes)
            labels.append(label)
            
        return labels
```

## Subtopics

### LangSplat Training Procedure
A sibling method that likely uses a different training strategy for semantic GSplat. [Link](12438-langsplat-training-procedure.md).

### Feature 3D Gaussian Splatting
A sibling method that uses neural fields for semantic distillation. [Link](15902-feature-3d-gaussian-splatting.md).

## Operating characteristics
*   **Time Complexity:** FAST-Splat is 6x to 8x faster to train than prior work [S1, unofficial].
*   **Memory Complexity:** FAST-Splat requires approximately 6x smaller GPU memory than prior work [S1, unofficial].
*   **Rendering Speed:** FAST-Splat achieves 18x to 51x faster rendering speeds compared to prior work [S1, unofficial].
*   **Performance Metrics:**
    *   On the illustrated case: SSIM: 0.971, PSNR: 34.24, GPU: 1.02GB [S1, unofficial].
    *   On the 3D-OVS dataset 'Bed' scene: mIoU: 0.933, Localization Accuracy: 0.998, Training Time: 13.96 minutes [S1, unofficial].
    *   On the 3D-OVS dataset 'Covered-Desk' scene: mIoU: 0.913, Localization Accuracy: 0.995, Training Time: 13.96 minutes [S1, unofficial].
*   **Strengths:** Fast training and rendering, low memory usage, ambiguity-free semantic disambiguation.
*   **Limitations:** Relies on the quality of the open-set detector and segmentation model for initial semantic map generation.
*   **When to use:** When fast, memory-efficient semantic GSplat with unambiguous object localization is required.
*   **When not to use:** When neural-field-based semantic distillation is preferred for specific reasons not covered by FAST-Splat's design.

## Background
FAST-Splat was introduced by Ola Shorinwa, Jiankai Sun, and Mac Schwager from Stanford University in the paper "FAST-Splat: Fast, Ambiguity-Free Semantics Transfer in Gaussian Splatting" (arXiv:2411.13753v2, 12 Mar 2025) [S1, unofficial]. It addresses the limitations of existing semantic Gaussian Splatting methods, which are slow, memory-intensive, and ambiguous in object localization. FAST-Splat builds on the explicit, sparse representation of 3D Gaussian Splatting to achieve these improvements without relying on neural fields [S1, unofficial].

## Sources
[S1] FAST-Splat: Fast, Ambiguity-Free Semantics Transfer in Gaussian Splatting, https://arxiv.org/abs/2411.13753v2, fetched 2026-10-26 (unofficial)
