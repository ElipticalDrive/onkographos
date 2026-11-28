# SAM Encoder Feature Extraction

Also called: SAM Encoder Cropping, SAM Encoder Feature Extraction and Cropping, SAM Feature Processing, SAM ViT Feature Extraction

## Overview
This method extracts high-level semantic features from a pre-trained Segment Anything Model (SAM) encoder to serve as ground-truth or supervisory signals for neural feature volumes in volume rendering [S1]. It handles the aspect-ratio mismatch and padding introduced by SAM’s fixed 1024×1024 input requirement by cropping the encoder output to the original image’s aspect ratio [S1]. By isolating the valid semantic information from the padded regions, it ensures that the rendered features align spatially with the input geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An original image $I$ with resolution $H \times W$ [S1].
*   **Output:** A cropped feature map $F_{crop}$ with spatial resolution matching the aspect ratio of $I$ and feature dimension $D_{SAM} = 256$ [S1].

**Data Structures:**
*   **SAM Encoder Output:** A tensor of shape $(64, 64, 256)$ before cropping [S1].
*   **Cropped Feature Map:** A tensor of shape $(H', W', 256)$, where $H'$ and $W'$ are derived from the aspect ratio [S1].

**Steps:**
1.  **Preprocessing:** Resize the input image $I$ to $1024 \times 1024$ by padding the shorter side with zeros [S1]. Let the padded image be $I_{pad}$.
2.  **Encoding:** Pass $I_{pad}$ through the SAM encoder (MAE pre-trained ViT-H/16) [S1]. The encoder downsamples the resolution by a factor of 16, producing a feature map $F_{full}$ of size $64 \times 64$ with feature dimension $256$ [S1].
3.  **Cropping:** Crop the padding region from the $64 \times 64$ feature map to retain only the valid semantic information corresponding to the original aspect ratio [S1]. If $W > H$, the valid region has height $H' = 64 \times \frac{H}{W}$ and width $W' = 64$; if $H > W$, the valid region has width $W' = 64 \times \frac{W}{H}$ and height $H' = 64$ [S1]. The crop is centered on the shorter dimension to account for the symmetric padding [S1].
4.  **Inference Padding:** When rendering features for SAM decoder input, pad the rendered feature map (which has size $H' \times W'$) back to $64 \times 64$ by adding padding on the shorter dimension side to match the original preprocessing [S1].

**Parameters:**
*   **SAM Encoder:** MAE pre-trained ViT-H/16 [S1].
*   **Architecture:** 14×14 windowed attention and four equally-spaced global attention blocks [S1].
*   **Feature Dimension:** 256 [S1].
*   **Input Resolution:** 1024×1024 [S1].
*   **Downscale Factor:** 16 [S1].
*   **Speed-up Module Dimension:** In practice, the rendered feature dimension is often reduced to 128 for the speed-up module [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def extract_sam_features(image: torch.Tensor, sam_encoder: torch.nn.Module) -> torch.Tensor:
    """
    Extracts semantic features from SAM encoder, cropping padding.
    image: Tensor of shape (H, W, 3)
    Returns: Tensor of shape (H_crop, W_crop, 256)
    """
    H, W = image.shape[0], image.shape[1]
    
    # Step 1: Pad shorter side to make square
    if H > W:
        pad_size = H - W
        pad_left = pad_size // 2
        pad_right = pad_size - pad_left
        image_padded = F.pad(image, (pad_left, pad_right, 0, 0), mode='constant', value=0)
    else:
        pad_size = W - H
        pad_top = pad_size // 2
        pad_bottom = pad_size - pad_top
        image_padded = F.pad(image, (0, 0, pad_top, pad_bottom), mode='constant', value=0)
        
    # Resize to 1024x1024
    image_resized = F.interpolate(image_padded.unsqueeze(0), size=(1024, 1024), mode='bilinear', align_corners=False).squeeze(0)
    
    # Step 2: Encode
    # SAM encoder outputs 64x64x256
    features_full = sam_encoder(image_resized) # Shape: (64, 64, 256)
    
    # Step 3: Crop to aspect ratio
    # Valid region in 64x64 map corresponds to original H x W aspect ratio
    max_dim = max(H, W)
    h_crop = int(round(64 * H / max_dim))
    w_crop = int(round(64 * W / max_dim))
    
    if H > W:
        # Padded left/right, valid region centered horizontally
        start_w = (64 - w_crop) // 2
        end_w = start_w + w_crop
        features_crop = features_full[:, start_w:end_w, :]
    else:
        # Padded top/bottom, valid region centered vertically
        start_h = (64 - h_crop) // 2
        end_h = start_h + h_crop
        features_crop = features_crop[start_h:end_h, :, :]
        
    return features_crop
```

## Subtopics

### Monocular Depth Feature Extraction
Uses a pre-trained monocular depth backbone (Depth Anything V2, a ViT) to extract features [S2]. The ViT has a patch size of 14 and outputs a feature map with spatial resolution 1/14 of the original image [S2]. These features are bilinearly interpolated to match the spatial resolution of the cost volume ($H/s \times W/s$) [S2]. The feature tensor is denoted $F_{mono_i} \in \mathbb{R}^{H/s \times W/s \times C_{mono}}$ [S2]. It is chosen when depth-aware semantic features are needed for tasks like depth estimation or Gaussian splatting initialization, leveraging the robustness of monocular depth models [S2].

### SAM Encoder-Decoder
An encoder-decoder architecture where the encoder extracts features from an image, and the decoder generates a 2D segmentation mask bitmap from features and prompts [S3]. The encoder computes $f_I = S_e(I)$ [S3]. The decoder computes $M_{SAM} = S_d(f_I, P)$, where $P$ is a set of prompts (points, boxes, texts, and masks) and $M_{SAM}$ is the output 2D segmentation mask bitmap [S3]. The computational overhead is dominated by the encoder, while the decoder accounts for only a minimal portion [S3].

## Operating characteristics
*   **Time Complexity:** Dominated by the SAM encoder inference [S1]. The encoder is a ViT-H/16, which is computationally expensive [S1]. The cropping and padding operations are negligible $O(1)$ relative to the encoder [S1].
*   **Memory Complexity:** The intermediate feature map is $64 \times 64 \times 256$ floats, which is small ($\approx 330$ KB) [S1]. The input image is resized to $1024 \times 1024$ [S1].
*   **Performance:** The source provides PSNR, SSIM, and LPIPS metrics for different feature dimensions (8 to 256) [S1]. At dimension 256, PSNR is 36.9139, SSIM is 0.9709, LPIPS is 0.0230, and FPS is 8.3 [S1]. Lower dimensions (e.g., 64) offer higher FPS (32.8) with slightly lower PSNR (36.9208) [S1].
*   **Strengths:** Provides high-quality semantic supervision from a powerful pre-trained model (SAM) [S1].
*   **Limitations:** Requires significant memory and compute for the ViT-H encoder [S1]. The fixed 1024×1024 input size may lose detail for very high-resolution scenes if not handled carefully [S1].

## Background
The method originates from "Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields" [S1]. It leverages the Segment Anything Model (SAM) [S3], which uses an MAE pre-trained ViT-H/16 [S1]. The approach addresses the need for dense semantic feature fields in 3D Gaussian Splatting by using SAM's encoder as a teacher [S1].

## Sources
[S1] Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields, https://doi.org/10.1109/cvpr52733.2024.02048, fetched 2026-10-26
[S2] DepthSplat: Connecting Gaussian Splatting and Depth, https://doi.org/10.1109/cvpr52734.2025.01534, fetched 2026-10-26
[S3] Segment Anything in 3D with Radiance Fields, https://doi.org/10.1007/s11263-025-02421-7, fetched 2026-10-26
