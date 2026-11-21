# SAM Encoder Feature Extraction

Also called: SAM Encoder Cropping, SAM Encoder Feature Extraction and Cropping, SAM Feature Processing, SAM ViT Feature Extraction

## Overview
This method extracts high-level semantic features from a pre-trained Segment Anything Model (SAM) encoder to serve as ground-truth or supervisory signals for neural feature volumes in volume rendering [S1]. It solves the problem of aspect-ratio mismatch and padding artifacts introduced by SAM’s fixed-input resolution requirement [S1]. By cropping the encoder output to the original image’s aspect ratio, it isolates the valid semantic information from the padded regions, ensuring that the rendered features align spatially and semantically with the input geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An original image $I$ with resolution $H \times W$.
*   **Output:** A cropped feature map $F_{crop}$ with spatial resolution matching the aspect ratio of $I$ and feature dimension $D_{SAM}$.

**Data Structures:**
*   **SAM Encoder Output:** A tensor of shape $(64, 64, D_{SAM})$ before cropping. The dimension $D_{SAM}$ is 256 for the standard ViT-H/16 encoder [S1].
*   **Cropped Feature Map:** A tensor of shape $(H', W', D_{SAM})$, where $H'$ and $W'$ are derived from the aspect ratio.

**Steps:**
1.  **Preprocessing:** Resize the input image $I$ to $1024 \times 1024$ by padding the shorter side with zeros (or black pixels) [S1]. Let the padded image be $I_{pad}$.
2.  **Encoding:** Pass $I_{pad}$ through the SAM encoder (MAE pre-trained ViT-H/16) [S1]. The encoder downsamples the resolution by a factor of 16, producing a feature map $F_{full}$ of size $64 \times 64$ with feature dimension $D_{SAM} = 256$ [S1].
3.  **Cropping:** Determine the valid spatial region corresponding to the original image aspect ratio.
    *   If $W > H$ (landscape), the width is preserved at 64, and the height is scaled: $H' = 64 \times \frac{H}{W}$. The crop is taken from the top-left corner (or centered, depending on padding strategy, but typically the padding is on the shorter side, so for $W>H$, padding is top/bottom, meaning the valid region is in the middle? No, SAM pads the shorter side. If $W>H$, height is padded. The valid region is in the middle vertically. However, the source says "crop out one side" or "crop out the padding region". Let's look at the formula: "new feature resolution becomes $64W/H \times 64$". This implies the width is scaled to $64W/H$ and height is 64? Wait. If $W>H$, the original image is wider. The padded image is square. The valid part is the original aspect ratio. If we keep height 64, the width must be $64 \times (W/H)$. Since $W>H$, $W/H > 1$, so width > 64. This contradicts the $64 \times 64$ output.
    *   Let's re-read carefully: "suppose the original image has the resolution of $H \times W$ where $W > H$, we crop the $64 \times 64$ feature map from SAM encoder so that the new feature resolution becomes $64W/H \times 64$".
    *   This implies the *output* resolution is $64W/H \times 64$. This means the width is expanded and height is fixed? No, the encoder output is $64 \times 64$. You cannot crop a $64 \times 64$ map to get a width $> 64$.
    *   Correction: The text likely means the *aspect ratio* is preserved. If $W > H$, the valid region in the $64 \times 64$ map is a rectangle of height 64 and width $64 \times (H/W)$? No, if $W>H$, the image is wide. The padding is on the top/bottom (height). The valid region is in the center vertically. The width is full (64). The height is $64 \times (H/W)$.
    *   Let's check the other case. If $H > W$, padding is on left/right. Valid width is $64 \times (W/H)$. Height is 64.
    *   The source text says: "new feature resolution becomes $64W/H \times 64$". This formula yields a width $> 64$ if $W>H$. This is physically impossible from a $64 \times 64$ crop.
    *   Alternative interpretation: The source might have a typo in the formula order or variable definition. Standard practice: If $W > H$, the valid region is $64 \times (64 \cdot H/W)$. If $H > W$, the valid region is $(64 \cdot W/H) \times 64$.
    *   Let's look at the inference step: "pad the rendered feature from $64W/H \times 64$ back to $64 \times 64$". This confirms the rendered feature has dimensions $64W/H \times 64$. This implies the *width* is the variable dimension. This only makes sense if the original image was $H \times W$ with $H > W$ (portrait), where the width is the shorter side. If $W > H$ (landscape), the width is the longer side.
    *   Let's assume the source notation $W$ and $H$ might be swapped or the formula applies to the *shorter* side being scaled.
    *   Actually, let's look at the specific text: "suppose ... $W > H$ ... new feature resolution becomes $64W/H \times 64$". This is definitely a typo in the source or my understanding of "crop". You cannot crop *out* padding to *increase* size. You crop *from* the $64 \times 64$ map.
    *   If $W > H$, the padding is vertical. The valid region is $64 \times (64 \cdot H/W)$.
    *   If $H > W$, the padding is horizontal. The valid region is $(64 \cdot W/H) \times 64$.
    *   The source says "crop out one side".
    *   Let's stick to the geometric truth: The valid region is the intersection of the $64 \times 64$ grid with the aspect ratio $H:W$.
    *   Let $S = 64$.
    *   If $W \ge H$: Valid Width = $S$. Valid Height = $S \cdot \frac{H}{W}$. Crop center vertically.
    *   If $H > W$: Valid Width = $S \cdot \frac{W}{H}$. Valid Height = $S$. Crop center horizontally.
    *   The source formula $64W/H \times 64$ suggests the width is $64W/H$. This is $> 64$ if $W>H$. This is impossible.
    *   However, if the source meant $64 \times (64 \cdot H/W)$, that fits.
    *   Let's look at the inference: "pad ... back to $64 \times 64$".
    *   I will implement the geometric crop based on aspect ratio preservation, noting the source's specific formula might contain a typo regarding the dimension $>64$.
    *   *Self-Correction*: The source says "crop out the padding region". If $W>H$, padding is top/bottom. We crop the top and bottom. The resulting width is 64. The height is $< 64$.
    *   If the source insists on $64W/H \times 64$, it might be defining $W$ as height and $H$ as width? No, standard is $H \times W$.
    *   I will write the algorithm to preserve aspect ratio from the $64 \times 64$ map.

    **Algorithm:**
    1.  Calculate aspect ratio $r = H/W$.
    2.  If $r \le 1$ ($H \le W$):
        *   Target Height $H' = 64$.
        *   Target Width $W' = \lfloor 64 \cdot r \rfloor$ or $\lceil 64 \cdot r \rceil$.
        *   Crop the $64 \times 64$ map to width $W'$ and height 64. Center horizontally.
    3.  If $r > 1$ ($H > W$):
        *   Target Width $W' = 64$.
        *   Target Height $H' = \lfloor 64 / r \rfloor$ or $\lceil 64 / r \rceil$.
        *   Crop the $64 \times 64$ map to width 64 and height $H'$. Center vertically.
    4.  The result is $F_{crop}$ of size $H' \times W' \times 256$.

4.  **Inference Padding:** When rendering features for SAM decoder input, pad the rendered feature map (which has size $H' \times W'$) back to $64 \times 64$ by adding padding on the shorter dimension side to match the original preprocessing [S1].

**Parameters:**
*   **SAM Encoder:** MAE pre-trained ViT-H/16 [S1].
*   **Architecture:** 14x14 windowed attention and four equally-spaced global attention blocks [S1].
*   **Feature Dimension:** 256 [S1].
*   **Input Resolution:** 1024x1024 [S1].
*   **Downscale Factor:** 16 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_sam_features(image: Tensor, sam_encoder: Module) -> Tensor:
    """
    Extracts semantic features from SAM encoder, cropping padding.
    image: Tensor of shape (H, W, 3)
    Returns: Tensor of shape (H_crop, W_crop, 256)
    """
    H, W = image.shape[0], image.shape[1]
    
    # Step 1: Resize to 1024x1024 with padding
    # Pad the shorter side to make it square
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
        
    # Resize to 1024x1024 if not already
    image_resized = F.interpolate(image_padded.unsqueeze(0), size=(1024, 1024), mode='bilinear', align_corners=False).squeeze(0)
    
    # Step 2: Encode
    # SAM encoder outputs 64x64x256
    features_full = sam_encoder(image_resized) # Shape: (64, 64, 256)
    
    # Step 3: Crop to aspect ratio
    # The valid region corresponds to the original H x W aspect ratio within the 1024x1024 square
    # In the 64x64 feature map, the valid region is scaled by 1/16
    
    # Calculate valid dimensions in 64x64 space
    # If original was H x W, and we padded to max(H,W) x max(H,W)
    # The valid region in the square is H_valid x W_valid
    # H_valid = 64 * (H / max(H,W))
    # W_valid = 64 * (W / max(H,W))
    
    max_dim = max(H, W)
    h_crop = int(round(64 * H / max_dim))
    w_crop = int(round(64 * W / max_dim))
    
    # Determine crop coordinates (centered)
    # The padding was added to the shorter side.
    # If H > W (portrait), we padded left/right. The valid region is in the center horizontally.
    # In the 64x64 map, the valid width is w_crop. Center it.
    if H > W:
        start_w = (64 - w_crop) // 2
        end_w = start_w + w_crop
        start_h = 0
        end_h = 64
        features_crop = features_full[start_h:end_h, start_w:end_w, :]
    else:
        # W >= H (landscape), we padded top/bottom. Valid region is in center vertically.
        start_h = (64 - h_crop) // 2
        end_h = start_h + h_crop
        start_w = 0
        end_w = 64
        features_crop = features_crop[start_h:end_h, start_w:end_w, :]
        
    return features_crop
```

## Subtopics

### Monocular Depth Feature Extraction
Uses a pre-trained monocular depth backbone (Depth Anything V2, a ViT) to extract features [S2]. The ViT has a patch size of 14 and outputs a feature map with spatial resolution 1/14 of the original image [S2]. These features are bilinearly interpolated to match the spatial resolution of the cost volume ($H/s \times W/s$) [S2]. The feature tensor is denoted $F_{mono_i} \in \mathbb{R}^{H/s \times W/s \times C_{mono}}$ [S2]. It is chosen when depth-aware semantic features are needed for tasks like depth estimation or Gaussian splatting initialization, leveraging the robustness of monocular depth models [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by the SAM encoder inference. The encoder is a ViT-H/16, which is computationally expensive. The cropping and padding operations are negligible $O(1)$ relative to the encoder.
*   **Memory Complexity:** The intermediate feature map is $64 \times 64 \times 256$ floats, which is small ($\approx 330$ KB). The input image is resized to $1024 \times 1024$.
*   **Performance:** The source provides PSNR, SSIM, and LPIPS metrics for different feature dimensions (8 to 256) [S1]. At dimension 256, PSNR is 36.9139, SSIM is 0.9709, LPIPS is 0.0230, and FPS is 8.3 [S1]. Lower dimensions (e.g., 64) offer higher FPS (32.8) with slightly lower PSNR (36.9208) [S1].
*   **Strengths:** Provides high-quality semantic supervision from a powerful pre-trained model (SAM) [S1].
*   **Limitations:** Requires significant memory and compute for the ViT-H encoder [S1]. The fixed 1024x1024 input size may lose detail for very high-resolution scenes if not handled carefully. The cropping step is critical; incorrect cropping leads to misaligned semantic features [S1].

## Background
The method originates from "Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields" [S1]. It leverages the Segment Anything Model (SAM) [20], which uses an MAE pre-trained ViT-H/16 [15, 8] with 14x14 windowed attention [S1]. The approach addresses the need for dense semantic feature fields in 3D Gaussian Splatting by using SAM's encoder as a teacher [S1].

## Sources
[S1] Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields, https://doi.org/10.1109/cvpr52733.2024.02048, fetched 2026-10-26
[S2] DepthSplat: Connecting Gaussian Splatting and Depth, https://doi.org/10.1109/cvpr52734.2025.01534, fetched 2026-10-26
