# DINO ViT Feature Extraction

Also called: DINO Feature Encoding, DINO feature extraction, DINOv2 Feature Integration, Self-Supervised ViT Feature Extraction for Transfer Functions, ViT-S/8

## Overview
This method extracts high-level semantic features from 3D volumetric data by leveraging a frozen, self-supervised Vision Transformer (ViT), specifically DINO, to create an explicit neural feature volume [S3]. It solves the problem of ambiguous intensity-based segmentation in medical imaging (e.g., CT scans) where different tissues share similar Hounsfield units but possess distinct semantic textures [S3]. Unlike traditional intensity-thresholding or supervised learning approaches that require extensive annotation or training, this method performs a one-time feature extraction and provides semantic features for segmentation that overcome the locality of traditional intensity-based features [S3]. It sets itself apart from its siblings by providing the specific pre-processing pipeline that converts raw volumetric data into a 3D feature tensor using 2D ViT projections, enabling interactive exploration without model training [S3].

## How it works
**Inputs and Outputs**
*   **Input:** Volumetric data $V \in \mathbb{R}^{W \times H \times D}$ (intensity values).
*   **Output:** A 3D feature volume $F \in \mathbb{R}^{W' \times H' \times D' \times F}$, where $W', H', D'$ are the spatial dimensions of the feature grid and $F$ is the feature dimension.

**Data Structures**
*   **Feature Volume ($F$):** A dense tensor of type Float32.
    *   Dimensions: $W' \times H' \times D' \times F$.
    *   Typical Size: $80 \times 80 \times 80 \times 384$ [S3].
    *   Memory: $80 \times 80 \times 80 \times 384 \times 4 \text{ bytes} \approx 768 \text{ MB}$ [S3].

**Parameters**
*   **Model:** `dino_vits8` (ViT-Small with patch size 8) [S3].
*   **Patch Size ($p$):** 8 [S3].
*   **Feature Dimension ($F$):** 384 (output dimension of the ViT-S/8 attention layers) [S3].
*   **Input Resize Dimension:** $\approx 640 \times 640$ pixels to achieve a spatial feature dimension of 80 ($640/8 = 80$) [S3].
*   **Layer Selection:** Features are taken from the 11th layer of the DINO network [S2].
*   **Implementation Variant:** Extended implementation by Amir et al. using overlapping patches with stride 4 to enlarge feature map size [S2].

**Steps**
1.  **Pre-processing (Feature Extraction):**
    1.  Resize the input volume slices to the target input resolution (e.g., $640 \times 640$) [S3].
    2.  Slice the input volume $V$ along its three principal axes (X, Y, Z) [S3].
    3.  For each slice, replicate the 2D slice to form an RGB image (3 channels) [S3].
    4.  Feed the RGB slices separately through the frozen pre-trained DINO ViT network [S3].
    5.  Extract the Key ($K$) vectors from the last self-attention layer of the ViT, as they represent semantic features designed to be matched to queries [S3].
    6.  (Optional Variant) If using the extended implementation, average the extracted features with features from the horizontally flipped image [S2].
    7.  The resulting 2D feature maps have dimensions reduced by the patch size $p$ in the non-sliced dimensions:
        *   $F_X \in \mathbb{R}^{W \times H/p \times D/p \times F}$
        *   $F_Y \in \mathbb{R}^{W/p \times H \times D/p \times F}$
        *   $F_Z \in \mathbb{R}^{W/p \times H/p \times D \times F}$
        [S3]
    8.  Merge the three stacks ($F_X, F_Y, F_Z$) into a single 3D feature volume $F$:
        *   Average pool each stack to the target dimensions ($W'=W/p, H'=H/p, D'=D/p$).
        *   Average the three pooled volumes together [S3].

2.  **Similarity Computation (Interactive Phase):**
    1.  User annotates structures of interest in a slice viewer [S3].
    2.  Sample the feature vectors $f_{ann}$ from the feature volume $F$ at the annotated locations [S3].
    3.  Compute the cosine similarity between $f_{ann}$ and every voxel in $F$ to generate a similarity volume $S_L$ [S3].
        $$ S_L(x,y,z) = \frac{f_{ann} \cdot F(x,y,z)}{\|f_{ann}\| \|F(x,y,z)\|} $$

3.  **Refinement (Optional):**
    1.  Apply a 3D extension to the Fast Bilateral Solver (BLS) to increase the resolution of the similarity map and adapt it to the underlying intensity volume [S3].
    2.  This step takes approximately 0.5 seconds [S3].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def extract_dino_features(volume: torch.Tensor, model: torch.nn.Module, patch_size: int = 8) -> torch.Tensor:
    """
    Extracts 3D feature volume from 3D intensity volume using DINO ViT.
    
    Args:
        volume: Input 3D volume of shape (W, H, D).
        model: Frozen DINO ViT-S/8 model.
        patch_size: Patch size of the ViT (default 8).
        
    Returns:
        Feature volume F of shape (W', H', D', F_dim).
    """
    W, H, D = volume.shape
    F_dim = 384  # Output dimension of ViT-S/8
    target_size = 640  # Resize dimension for 80x80 feature maps
    
    # Initialize empty tensor for merged features
    # Dimensions: W' x H' x D' x F_dim
    W_prime, H_prime, D_prime = W // patch_size, H // patch_size, D // patch_size
    F_volume = torch.zeros((W_prime, H_prime, D_prime, F_dim), dtype=torch.float32)
    
    # Helper to process 2D slices
    def process_slices(axis: int, volume_slice: torch.Tensor):
        """
        Processes slices along a specific axis.
        axis: 0 for X (slicing along W), 1 for Y (slicing along H), 2 for Z (slicing along D).
        volume_slice: 2D tensor of shape (Dim1, Dim2).
        """
        # Resize to target input size
        resized = F.interpolate(
            volume_slice.unsqueeze(0).unsqueeze(0), # Add batch and channel dims
            size=(target_size, target_size),
            mode='bilinear',
            align_corners=False
        ).squeeze(0).squeeze(0)
        
        # Replicate to RGB (3 channels)
        rgb_input = resized.repeat(3, 1, 1)
        
        # Forward pass through frozen ViT
        # Extract keys from the last self-attention layer
        # Note: In practice, this requires accessing the internal layers of the ViT
        # Assuming model returns features for the last layer's keys
        with torch.no_grad():
            features = model(rgb_input.unsqueeze(0)) # Add batch dim
            # features shape: (1, num_patches, F_dim) or similar depending on implementation
            # Reshape to spatial grid
            # num_patches = (target_size / patch_size)^2 = 80*80 = 6400
            spatial_dim = target_size // patch_size # 80
            
            # Reshape features to (1, F_dim, spatial_dim, spatial_dim)
            # Then permute to (1, spatial_dim, spatial_dim, F_dim)
            feat_grid = features[0].permute(1, 0).reshape(1, F_dim, spatial_dim, spatial_dim)
            feat_grid = feat_grid.permute(0, 2, 3, 1) # (1, 80, 80, 384)
            
            # Remove batch dim
            feat_grid = feat_grid.squeeze(0) # (80, 80, 384)
            
            return feat_grid

    # Process X-axis slices (slice along W)
    # Resulting shape contribution: W (unchanged) x H/p x D/p
    # We need to map these to the global F_volume
    for i in range(W):
        slice_2d = volume[i, :, :] # Shape (H, D)
        feat_2d = process_slices(0, slice_2d) # Shape (H/p, D/p, F_dim)
        
        # Map to F_volume
        # F_volume[i, :, :, :] = feat_2d
        # However, F_volume dimensions are (W/p, H/p, D/p, F_dim)
        # Wait, the source says F_X has shape W x H/p x D/p.
        # But the merged volume is W/p x H/p x D/p.
        # This implies we must downsample/average pool the W dimension as well.
        
        # Correction based on source: "average pooled to the target dimensions"
        # So we collect all W slices, then pool.
        pass

    # Implementation Note: The source describes collecting three stacks F_X, F_Y, F_Z
    # and then average pooling them to W/p, H/p, D/p.
    
    # Stack X: Shape (W, H/p, D/p, F_dim)
    stack_X = torch.zeros((W, H_prime, D_prime, F_dim))
    for i in range(W):
        slice_2d = volume[i, :, :]
        stack_X[i, :, :, :] = process_slices(0, slice_2d)
        
    # Stack Y: Shape (W/p, H, D/p, F_dim)
    stack_Y = torch.zeros((W_prime, H, D_prime, F_dim))
    for j in range(H):
        slice_2d = volume[:, j, :]
        stack_Y[:, j, :, :] = process_slices(1, slice_2d)
        
    # Stack Z: Shape (W/p, H/p, D, F_dim)
    stack_Z = torch.zeros((W_prime, H_prime, D, F_dim))
    for k in range(D):
        slice_2d = volume[:, :, k]
        stack_Z[:, :, k, :] = process_slices(2, slice_2d)
        
    # Average Pool each stack to (W/p, H/p, D/p, F_dim)
    # Pooling over the dimension that was not reduced by patching
    # For X: Pool over W (size W) to W/p
    pool_X = F.avg_pool3d(stack_X.permute(3, 0, 1, 2), kernel_size=(patch_size, 1, 1)).permute(1, 2, 3, 0)
    # For Y: Pool over H (size H) to H/p
    pool_Y = F.avg_pool3d(stack_Y.permute(3, 0, 1, 2), kernel_size=(1, patch_size, 1)).permute(1, 2, 3, 0)
    # For Z: Pool over D (size D) to D/p
    pool_Z = F.avg_pool3d(stack_Z.permute(3, 0, 1, 2), kernel_size=(1, 1, patch_size)).permute(1, 2, 3, 0)
    
    # Average the three pooled volumes
    F_volume = (pool_X + pool_Y + pool_Z) / 3.0
    
    return F_volume

def compute_similarity(feature_volume: torch.Tensor, annotations: list) -> torch.Tensor:
    """
    Computes similarity map based on annotated feature vectors.
    
    Args:
        feature_volume: F of shape (W', H', D', F_dim).
        annotations: List of 3D coordinates (x, y, z) annotated by user.
        
    Returns:
        Similarity volume S_L of shape (W', H', D').
    """
    W_prime, H_prime, D_prime, F_dim = feature_volume.shape
    
    # Get mean feature vector of annotations
    ann_features = feature_volume[annotations[:, 0], annotations[:, 1], annotations[:, 2], :]
    mean_ann_feat = torch.mean(ann_features, dim=0)
    
    # Compute cosine similarity
    # Normalize feature volume
    norm_feat = F.normalize(feature_volume, p=2, dim=-1)
    norm_ann = F.normalize(mean_ann_feat, p=2, dim=-1)
    
    # Dot product
    similarity = torch.sum(norm_feat * norm_ann, dim=-1)
    
    return similarity
```

**Edge Cases and Pitfalls**
*   **Low Resolution:** Due to input patching in ViTs, extracted features are of low resolution, which prohibits high visual quality when rendered directly [S3]. Refinement is required [S3].
*   **2D vs 3D Mismatch:** Pre-trained ViTs are trained on 2D data, requiring a specific strategy (slice-wise extraction and merging) to extract meaningful features from 3D volumetric data [S3].
*   **Memory Constraints:** Larger models (ViT-B, ViT-L, DINOv2) require prohibitive GPU memory and have larger patch sizes, decreasing spatial resolution [S3]. The method uses ViT-S/8 to fit on consumer GPUs (e.g., RTX 2070, 8GB VRAM) [S3].
*   **Text Queries:** DINO cannot accept text queries, unlike LSeg [S2].

## Subtopics

### [Feature-based TF Design via ViT Features](6466-feature-based-tf-design-via-vit-features.md)
This method designs transfer functions (TFs) for volume rendering by leveraging semantic features extracted from a pre-trained Vision Transformer (ViT), specifically DINO, to guide segmentation and opacity mapping. It solves the problem of ambiguous intensity-based segmentation in medical imaging (e.g., CT scans) where different tissues share similar Hounsfield units. Unlike traditional intensity-thresholding or supervised learning approaches that require extensive annotation or training, this method performs one-time feature extraction and uses a post-processing step (Block Label Smoothing) to generate similarity maps for TF design. It sets itself apart from its parent [DINO ViT Feature Extraction](6425-dino-vit-feature-extraction.md) by providing the specific annotation and post-processing pipeline that converts raw feature volumes into usable transfer functions, and it outperforms Support Vector Machines (SVM) and Random Forests (RF) in segmentation quality with significantly fewer annotations.

## Operating characteristics
*   **Time Complexity:** Pre-processing takes approximately 1-3 minutes on a consumer GPU (RTX 2070, 8GB VRAM) [S3]. Interactive similarity computation is fast and can run on CPU [S3]. Refinement takes approximately 0.5 seconds [S3].
*   **Memory Complexity:** The feature volume for an $80 \times 80 \times 80$ grid with 384 dimensions requires approximately 768 MB of VRAM [S3]. Larger models or higher resolutions increase memory usage prohibitively [S3].
*   **Strengths:** No training required; interactive exploration; leverages semantic features to overcome intensity ambiguity; fast inference [S3].
*   **Limitations:** Low spatial resolution of features; requires refinement for high-quality rendering; limited to 2D pre-trained models; cannot use text queries [S3][S2].
*   **When to Use:** When semantic segmentation is needed for volume rendering without labeled training data, and interactive exploration is desired [S3].
*   **When Not to Use:** When high-resolution features are needed directly without refinement, or when text-based queries are required [S3][S2].

## Background
The method originates from the paper "Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design" (2024) [S3]. It builds on the capabilities of self-supervised Vision Transformers (ViTs) like DINO, which extract meaningful features from 2D images without supervision [S3]. The approach addresses the limitations of traditional intensity-based transfer functions and supervised learning methods that require extensive annotation and training [S3]. It is used in volume rendering for interactive exploration and segmentation-based transfer function design [S3].

## Sources
[S1] SparSplat: Fast Multi-View Reconstruction with Generalizable 2D Gaussian Splatting, https://doi.org/10.48550/arXiv.2505.02175, fetched 2026-10-26 (unofficial)
[S2] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S3] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
