# LPIPS Computation
Also called: Learned Perceptual Image Patch Similarity, LPIPS Loss, Learning Perceptual Image Patch Similarity (LPIPS)

## Overview
LPIPS is a full-reference perceptual similarity metric that quantifies the visual difference between two images by comparing deep feature representations extracted from pretrained convolutional neural networks [S1][S2, unofficial][S3]. Unlike pixel-wise metrics such as PSNR or SSIM, which often correlate poorly with human perception, LPIPS captures high-level semantic and structural nuances, making it a superior indicator of perceptual quality for tasks like image generation and differentiable rendering [S4][S6][S7, unofficial]. It sets itself apart from spatially-local statistical metrics like Wasserstein Distortion by relying on weighted Euclidean distances of feature activations across multiple network layers [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Two images, $x$ (reference) and $y$ (assessed/reconstructed), typically normalized to the range $[0, 1]$ or $[-1, 1]$.
- **Output:** A scalar perceptual distance score $LPIPS(x, y)$. Lower values indicate higher perceptual similarity.

**Data Structures:**
The method relies on the internal feature maps of a pretrained backbone network (e.g., VGG16, AlexNet, or SqueezeNet) [S1][S3][S5, unofficial]. For each layer $l$, the network outputs a feature tensor $F^l$ of shape $(C_l, H_l, W_l)$, where $C_l$ is the number of channels, and $H_l, W_l$ are the height and width of the feature map [S1]. The metric uses pre-computed learned weights $w_l$ for each layer, which are fixed constants derived from a regression against human perceptual judgments [S1][S2, unofficial].

**Parameters:**
- **Backbone Network:** Commonly VGG16 [S5, unofficial][S6], AlexNet [S2, unofficial][S3], or SqueezeNet [S1]. VGG16 is the standard reference implementation [S5, unofficial].
- **Weighted Coefficients ($w_l$):** Learned per-channel weights that normalize the feature responses. These are not computed during inference but loaded from the pretrained model weights [S1][S2, unofficial].
- **Normalization:** Inputs are typically scaled from $[0, 1]$ to $[-1, 1]$ to match the training distribution of the backbone network [S5, unofficial].

**Formula:**
The LPIPS score is calculated as the weighted sum of squared Euclidean distances between feature activations at multiple layers [S1][S2, unofficial]:

$$
LPIPS(x, y) = \sum_{l} w_l \| f_l(x) - f_l(y) \|_2^2
$$

Where:
- $l$ iterates over selected layers of the backbone network.
- $f_l(x)$ and $f_l(y)$ are the feature maps for images $x$ and $y$ at layer $l$.
- $\| \cdot \|_2^2$ denotes the squared $L_2$ norm (sum of squared differences) across all spatial positions and channels in the feature map.
- $w_l$ is the learned weight coefficient for layer $l$.

An equivalent formulation explicitly showing the spatial summation is [S1]:

$$
LPIPS(x, y) = \sum_{l} \frac{1}{H_l W_l} \sum_{h,w} ||w_l \odot (x^l_{hw} - y^l_{hw})||_2^2
$$

Where $x^l_{hw}$ and $y^l_{hw}$ are the feature vectors at pixel $(h, w)$ in layer $l$, and $\odot$ represents element-wise multiplication by the weight vector $w_l$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torchvision.models as models

def compute_lpips(reference_img: torch.Tensor, reconstructed_img: torch.Tensor, 
                  backbone_name: str = "vgg") -> float:
    """
    Computes the LPIPS score between two images.
    
    Args:
        reference_img: Tensor of shape (C, H, W) or (B, C, H, W) with values in [0, 1].
        reconstructed_img: Tensor of shape (C, H, W) or (B, C, H, W) with values in [0, 1].
        backbone_name: Name of the pretrained network ('vgg', 'alexnet', 'squeezenet').
        
    Returns:
        Scalar float representing the perceptual distance.
    """
    # 1. Load the pretrained backbone model and its LPIPS weights
    # Note: In a real implementation, this involves loading the specific 
    # LPIPS-ported weights (e.g., from the lpips library) which contain 
    # the learned channel-wise weights w_l.
    # Here we assume a helper `get_lpips_model` that returns a module 
    # computing the weighted feature distance.
    model = get_lpips_model(backbone_name)
    model.eval()
    
    # 2. Preprocessing: Scale from [0, 1] to [-1, 1]
    # This matches the normalization expected by the pretrained backbone (e.g., VGG)
    ref_scaled = reference_img * 2.0 - 1.0
    recon_scaled = reconstructed_img * 2.0 - 1.0
    
    # 3. Add batch dimension if not present
    if ref_scaled.dim() == 3:
        ref_scaled = ref_scaled.unsqueeze(0)
        recon_scaled = recon_scaled.unsqueeze(0)
        
    # 4. Feature Extraction and Distance Calculation
    # The model internally extracts features from multiple layers,
    # computes the difference, applies learned weights w_l, and sums them.
    with torch.no_grad():
        # Forward pass through the LPIPS-specific wrapper
        # This wrapper handles the multi-layer feature aggregation defined in [S1][S2, unofficial]
        distance = model(ref_scaled, recon_scaled)
        
    # 5. Return the mean distance (handles batch dimension if >1)
    return distance.mean().item()

def get_lpips_model(arch: str) -> nn.Module:
    """
    [G] Helper to load the specific LPIPS implementation.
    Standard torchvision models do not contain the learned weights w_l.
    This function returns a module that computes:
    sum_l ( w_l * || feat_l(x) - feat_l(y) ||^2 )
    """
    if arch == "vgg":
        # Load VGG16 features
        vgg = models.vgg16(pretrained=True).features
        # [G] The actual LPIPS implementation wraps this to extract specific layers
        # and applies the learned weights. For a plain-text implementation,
        # one would manually extract features from layers [3, 8, 15, 22, 29]
        # and apply the pre-computed weights.
        return LPIPSWrapper(vgg, weights_path="vgg_lpips_weights.pth")
    elif arch == "alexnet":
        alex = models.alexnet(pretrained=True).features
        return LPIPSWrapper(alex, weights_path="alex_lpips_weights.pth")
    else:
        raise ValueError("Unsupported architecture")

class LPIPSWrapper(nn.Module):
    def __init__(self, backbone: nn.Module, weights_path: str):
        super().__init__()
        self.backbone = backbone
        # [G] Load pre-computed learned weights w_l for each layer
        # These weights are specific to the LPIPS paper [S1] and are not 
        # part of the standard backbone training.
        self.weights = torch.load(weights_path)
        # Define which layers to use (e.g., conv1_2, conv2_2, conv3_3, conv4_3, conv5_3 for VGG)
        self.layer_indices = [3, 8, 15, 22, 29] 
        
    def forward(self, x: torch.Tensor, y: torch.Tensor) -> torch.Tensor:
        # Extract features from specified layers
        feats_x = []
        feats_y = []
        
        current_input = x
        current_input_y = y
        
        for i, idx in enumerate(self.layer_indices):
            # Forward pass through backbone up to layer idx
            # Note: In practice, one runs the full forward pass and caches features
            # or uses a hook. Here we simulate the extraction.
            # [G] Simplified: assume we have a function get_features that returns 
            # a list of tensors for the specified layers.
            pass 
            
        # [G] The core LPIPS calculation:
        # For each layer l:
        #   diff = feat_x[l] - feat_y[l]
        #   weighted_diff = diff * weights[l]  # Element-wise multiplication
        #   squared_norm = sum(weighted_diff^2) over spatial dims and channels
        #   layer_score = squared_norm / (H_l * W_l) # Normalization by spatial size
        # Total LPIPS = sum(layer_scores)
        
        # Since we cannot call external libraries, we assume `extract_features`
        # returns a list of tensors [F_0, F_1, ...] for the selected layers.
        feats_x = extract_features(x, self.backbone, self.layer_indices)
        feats_y = extract_features(y, self.backbone, self.layer_indices)
        
        total_loss = 0.0
        for i, (fx, fy) in enumerate(zip(feats_x, feats_y)):
            # fx, fy shape: (B, C, H, W)
            diff = fx - fy
            # Apply learned weights w_l (shape: (1, C, 1, 1) or similar broadcastable)
            w = self.weights[i]
            weighted_diff = diff * w
            # Compute squared L2 norm, normalized by spatial dimensions
            # Sum over C, H, W
            squared_norm = torch.sum(weighted_diff ** 2, dim=(1, 2, 3))
            # Normalize by number of elements in the feature map (H*W*C? No, usually just H*W or total elements)
            # The formula in [S1] divides by H_l * W_l, but the weight w_l 
            # typically absorbs the channel normalization. 
            # Standard LPIPS implementation normalizes by the number of elements in the feature map.
            num_elements = fx.shape[1] * fx.shape[2] * fx.shape[3]
            layer_loss = torch.mean(squared_norm / num_elements)
            total_loss += layer_loss
            
        return total_loss

def extract_features(img: torch.Tensor, backbone: nn.Module, layer_indices: list) -> list:
    """
    [G] Helper to extract features from specific layers of a backbone.
    """
    features = []
    x = img
    current_idx = 0
    for module in backbone.children():
        x = module(x)
        if current_idx in layer_indices:
            features.append(x)
        current_idx += 1
        if current_idx > layer_indices[-1]:
            break
    return features
```

**Edge Cases and Pitfalls:**
- **Pretrained Weights:** The first run requires downloading pretrained weights (e.g., VGG16 and LPIPS-specific weights) to the cache directory (e.g., `TORCH_HOME`) [S5, unofficial]. This requires internet access or pre-cached weights.
- **Input Normalization:** Failing to scale inputs to $[-1, 1]$ (if the backbone expects it) can lead to incorrect feature activations and invalid scores [S5, unofficial].
- **Spatial Alignment:** LPIPS assumes pixel-perfect alignment between $x$ and $y$. Misaligned images will yield high distances even if perceptually similar [S1].
- **Batch Processing:** The metric is typically computed per image and then averaged. Ensure batch dimensions are handled correctly [S5, unofficial].

## Subtopics

### [Wasserstein Distortion](15233-wasserstein-distortion.md)
Wasserstein Distortion (WD) is a perceptual image-to-image metric that compares spatially local estimates of statistics (mean and standard deviation) in deep feature space, allowing textures with large pointwise differences but similar local statistics to be considered perceptually similar. Unlike pointwise metrics such as LPIPS Computation which penalize phase shifts and exact pixel alignment, WD models human peripheral vision by aggregating features over local pooling regions, making it suitable for optimizing perceptual texture quality in differentiable rendering.

### Reference-based LPIPS
Reference-based LPIPS is a metric to evaluate reference-based perceptual similarity for stylized radiance fields, measuring how well novel views preserve the style of a given reference image [S8]. The procedure involves rendering a set of views from the stylized NeRF, using these rendered views as style references to generate a set of stylized NeRFs, and then computing the PSNR between the new rendering results and the original NeRF to measure robustness [S8]. For frame-wise consistency, the closest 10 frames are selected to represent consistency with the given style reference [S8]. Scene-wise LPIPS scores show significant variations across scenes, attributed to substantial differences in camera poses between the reference view and test views [S8]. While methods like Texler achieve slightly better reference-related LPIPS scores, they fail when the test camera pose diverges significantly from the reference camera, producing artifacts in occluded regions and flickering/ghosting in continuous sequences [S8].

### DISTS Evaluation Metric
DISTS is a metric that assesses image quality by calculating distances through feature mapping learning based on differentiable image saliency transforms [S6]. It involves calculating the significance score of each pixel using a pre-trained deep learning model (e.g., VGG network), sorting pixels according to their saliency scores to highlight the most important image parts, constructing a new image representation using the sorted pixel order, and computing an assessment score by comparing the difference in the new representation between the original and processed images [S6]. DISTS provides more accurate and efficient evaluation results compared to traditional methods by leveraging deep learning and neural network modeling [S6].

## Operating characteristics
- **Time Complexity:** The computation is dominated by the forward pass through the pretrained backbone network. For VGG16, this is $O(C \cdot H \cdot W)$ per layer, summed over $L$ layers. It is significantly slower than PSNR or SSIM but faster than training a new network.
- **Memory Complexity:** Requires storing the feature maps for all selected layers for both images. For VGG16, this is manageable but increases with image resolution.
- **Measured Performance:** LPIPS correlates strongly with human perceptual judgments, outperforming PSNR and SSIM in tasks like image generation and stylization [S3][S6][S7, unofficial]. It is widely used in GANs and image generation tasks [S6].
- **Strengths:** Captures semantic and structural information better than pixel-wise metrics; robust to minor misalignments if features are high-level; aligns well with human perception [S4][S6].
- **Limitations:** Computationally expensive compared to PSNR; requires pretrained weights; sensitive to exact pixel alignment; does not account for spatial statistics like WD [S1][S2, unofficial].
- **When to Use:** When perceptual quality is more important than pixel accuracy; in differentiable rendering optimization where human perception is the target; in evaluating stylized or generative outputs [S4][S6][S7, unofficial].
- **When Not to Use:** When real-time performance is critical and PSNR/SSIM are sufficient; when pixel-perfect reconstruction is required regardless of perception; when computational resources are extremely limited.

## Background
LPIPS was introduced by Zhang et al. in 2018 [S1][S3]. It was developed to address the limitations of traditional metrics like PSNR and SSIM, which often fail to correlate with human perception. The metric uses learned convolutional features from pretrained networks (initially SqueezeNet, VGG, and AlexNet) to score perceptual similarity [S1]. It has since become a standard metric in computer vision, particularly in generative modeling and differentiable rendering, where perceptual quality is paramount [S4][S6].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
[S3] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
[S4] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S5] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S6] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S7] Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, https://doi.org/10.48550/arXiv.2607.17411, fetched 2026-10-29 (unofficial)
[S8] Ref-NPR: Reference-Based Non-Photorealistic Radiance Fields for Controllable Scene Stylization, https://doi.org/10.1109/cvpr52729.2023.00413, fetched 2026-10-26
