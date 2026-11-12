# 3D Image Resize and Crop

Also called: 3D Image Resize and Crop for Super-Resolution

## Overview
This method prepares volumetric data for super-resolution or rendering by standardizing spatial dimensions and normalizing intensity ranges. It solves the problem of non-uniform input resolutions and aspect ratios by cropping the Z-dimension to a central region and padding the X-Y plane to a square, followed by bilinear upsampling [S1, unofficial]. It sets itself apart from neural super-resolution methods by relying on deterministic geometric interpolation and explicit boundary handling rather than learned priors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume array (typically NumPy or similar) with shape $(D, H, W)$, where $D$ is the depth (Z-axis) and $H, W$ are height and width (X-Y axes).
- Output: A normalized tensor of shape $(C, H', W', 1)$ for high-resolution data, and optionally a low-resolution counterpart.

**Data Structures:**
- The input volume is treated as a dense grid.
- The output is a PyTorch tensor with dimensions permuted to Channel-Height-Width-Depth format for compatibility with standard image processing libraries [S1, unofficial].

**Steps:**
1.  **Z-Dimension Cropping:**
    - Calculate the center index of the Z-dimension: $z_{center} = \lfloor D / 2 \rfloor$.
    - Determine the crop size: $z_{crop} = \lfloor \text{image\_size}[0] / 2 \rfloor$.
    - Extract the central region: $V_{crop} = V[z_{center} - z_{crop} : z_{center} + z_{crop}, :, :]$.
    - *Note:* If the source volume is smaller than the target crop, this step may require clamping or padding, though the source assumes sufficient depth [S1, unofficial].

2.  **X-Y Dimension Padding:**
    - Let $H$ and $W$ be the height and width of $V_{crop}$.
    - If $H \neq W$:
        - Identify the larger dimension: $L = \max(H, W)$.
        - Calculate padding for the smaller dimension: $p = L - \min(H, W)$.
        - Pad the smaller dimension with zeros to center the content. For example, if $H < W$, pad height with $\lfloor p/2 \rfloor$ on top and $\lceil p/2 \rceil$ on bottom.
        - Resulting shape is $(z_{crop}, L, L)$.

3.  **Tensor Conversion and Resizing:**
    - Convert the padded volume to a PyTorch tensor with shape $(B, C, H, W)$. Typically, $B=1$ (batch), $C=1$ (single channel).
    - Resize using bilinear interpolation to the target size $(\text{image\_size}[1], \text{image\_size}[2])$.
    - Formula for bilinear interpolation at target coordinate $(u, v)$ involves weighted averaging of the four nearest neighbors in the source grid [G].

4.  **Normalization:**
    - Find the global maximum intensity of the resized volume: $M = \max(V_{resized})$.
    - Normalize the volume: $V_{norm} = V_{resized} / M$.
    - *Note:* If the input volume max is $\le 1$, it is scaled by 255 before processing [S1, unofficial].

5.  **Permutation:**
    - Permute dimensions from $(B, C, H, W)$ to $(C, H, W, 1)$ for final output [S1, unofficial].

6.  **Low-Resolution Generation (Optional):**
    - Perform the same Z-crop and X-Y padding on the original volume to create a base low-resolution version.
    - Convert to tensor and resize to the target low-resolution size.
    - Normalize by dividing by its respective maximum.
    - Permute to $(C, H, W, 1)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def resize_and_crop(volume: np.ndarray, image_size: tuple, return_lr: bool = False) -> tuple:
    """
    Resize and crop a 3D volume for super-resolution preprocessing.
    
    Args:
        volume: Input 3D array of shape (D, H, W).
        image_size: Tuple (target_depth_crop_factor, target_H, target_W). 
                    Note: image_size[0] is used for Z-crop calculation, 
                    image_size[1] and image_size[2] for final resize.
        return_lr: If True, also return the low-resolution version.
        
    Returns:
        High-resolution normalized tensor [C, H, W, 1].
        Optionally, low-resolution normalized tensor [C, H_lr, W_lr, 1].
    """
    # 1. Z-Dimension Cropping
    D, H, W = volume.shape
    z_center = D // 2
    z_crop = image_size[0] // 2
    
    # Ensure indices are within bounds
    z_start = max(0, z_center - z_crop)
    z_end = min(D, z_center + z_crop)
    
    volume_crop = volume[z_start:z_end, :, :]
    
    # 2. X-Y Dimension Padding
    h, w = volume_crop.shape[1], volume_crop.shape[2]
    if h != w:
        L = max(h, w)
        pad_h = L - h
        pad_w = L - w
        
        # Pad height
        pad_top = pad_h // 2
        pad_bottom = pad_h - pad_top
        volume_crop = np.pad(volume_crop, ((0, 0), (pad_top, pad_bottom), (0, 0)), mode='constant', constant_values=0)
        
        # Pad width
        pad_left = pad_w // 2
        pad_right = pad_w - pad_left
        volume_crop = np.pad(volume_crop, ((0, 0), (0, 0), (pad_left, pad_right)), mode='constant', constant_values=0)
        
    # 3. Tensor Conversion
    # Add batch and channel dimensions: (1, 1, H, W)
    tensor_vol = torch.from_numpy(volume_crop).unsqueeze(0).unsqueeze(0).float()
    
    # 4. Resize using Bilinear Interpolation
    # Target size is image_size[1] x image_size[2]
    target_h, target_w = image_size[1], image_size[2]
    resized_vol = torch.nn.functional.interpolate(
        tensor_vol, 
        size=(target_h, target_w), 
        mode='bilinear', 
        align_corners=False
    )
    
    # 5. Normalization
    # If original max <= 1, scale by 255 (applied to input before crop in source, 
    # but here we apply normalization after resize based on global max)
    # Source logic: "if volume.max() <= 1: volume = volume * 255" happens before crop.
    # We assume input is already scaled or we normalize by max.
    max_val = resized_vol.max()
    if max_val > 0:
        normalized_vol = resized_vol / max_val
    else:
        normalized_vol = resized_vol
        
    # 6. Permutation to [C, H, W, 1]
    # Current shape: [1, 1, H, W]
    # Permute to [1, H, W, 1] -> squeeze batch if needed or keep batch dim
    # Source says permute to [C, H, W, 1]. Assuming C=1.
    output_hr = normalized_vol.permute(0, 2, 3, 1) # [1, H, W, 1]
    
    if return_lr:
        # Generate Low-Res version
        # Perform same crop/pad on original volume
        volume_crop_lr = volume[z_start:z_end, :, :]
        h_lr, w_lr = volume_crop_lr.shape[1], volume_crop_lr.shape[2]
        if h_lr != w_lr:
            L_lr = max(h_lr, w_lr)
            pad_h_lr = L_lr - h_lr
            pad_w_lr = L_lr - w_lr
            pad_top_lr = pad_h_lr // 2
            pad_bottom_lr = pad_h_lr - pad_top_lr
            volume_crop_lr = np.pad(volume_crop_lr, ((0, 0), (pad_top_lr, pad_bottom_lr), (0, 0)), mode='constant', constant_values=0)
            pad_left_lr = pad_w_lr // 2
            pad_right_lr = pad_w_lr - pad_left_lr
            volume_crop_lr = np.pad(volume_crop_lr, ((0, 0), (0, 0), (pad_left_lr, pad_right_lr)), mode='constant', constant_values=0)
            
        tensor_vol_lr = torch.from_numpy(volume_crop_lr).unsqueeze(0).unsqueeze(0).float()
        # Resize to target low-res size (same target in source example, but conceptually lower res)
        # Source says "resize to target size to create low-resolution version". 
        # In context of SR, LR is usually the input to the network, HR is the target.
        # Here, both are resized to image_size[1]ximage_size[2] for consistency in the example,
        # but typically LR would be smaller. We follow source literally.
        resized_vol_lr = torch.nn.functional.interpolate(
            tensor_vol_lr, 
            size=(target_h, target_w), 
            mode='bilinear', 
            align_corners=False
        )
        max_val_lr = resized_vol_lr.max()
        if max_val_lr > 0:
            normalized_vol_lr = resized_vol_lr / max_val_lr
        else:
            normalized_vol_lr = resized_vol_lr
        output_lr = normalized_vol_lr.permute(0, 2, 3, 1)
        
        return output_hr, output_lr
        
    return output_hr
```

## Subtopics

### Slice-wise Nearest-Neighbor Resampling
Resizes 2D slices of a volume dataset using nearest-neighbor interpolation to correct non-uniform spacing or scaling issues in specific anatomical regions [S2, unofficial]. It is chosen when preserving sharp edges in specific slices is critical and bilinear interpolation would introduce blurring artifacts [S2, unofficial]. The method iterates through specific slice ranges (e.g., 0-209, 209-227) and applies different scaling factors (1.9, 1.3, 1.1) to each [S2, unofficial]. The coordinate mapping is defined as $nearestX = \lfloor x \cdot width / resizeW \rfloor$ and $nearestY = \lfloor y \cdot height / resizeH \rfloor$ [S2, unofficial]. Target dimensions are calculated as $CT\_x\_axis / factor$ and $CT\_y\_axis / factor$ [S2, unofficial].

### Resized Slice Centering
Centers a resized 2D matrix within a larger container matrix, padding unused regions with the Hounsfield unit value for air (-1024) [S2, unofficial]. It sets itself apart by explicitly handling anatomical context (air background) rather than zero-padding [S2, unofficial]. The output matrix is initialized to -1024. The centering logic calculates $centreBegins = (CT\_x\_axis - content.length) / 2$ and copies content into the center, assuming $content.length < containerWidth$ [S2, unofficial]. A pitfall is that if $centreBegins$ is negative or out of bounds, indices will be invalid, and the code does not explicitly clamp indices [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels in the cropped region, $O(D \cdot H \cdot W)$, due to cropping and padding. Bilinear interpolation is also linear in the output size [G].
- **Memory Complexity:** Requires storage for the original volume, the cropped/padded volume, and the output tensor. Zero-padding increases memory usage slightly compared to cropping alone [S1, unofficial].
- **Strengths:** Simple, deterministic, and fast. Effective for standardizing input dimensions for neural networks or rendering pipelines [S1, unofficial].
- **Limitations:** Bilinear interpolation blurs sharp boundaries. Zero-padding can introduce artificial edges if not handled carefully in subsequent processing [S1, unofficial]. Not suitable for preserving high-frequency details without additional filtering [S1, unofficial].
- **When to use:** When preparing data for super-resolution models that require square inputs or fixed aspect ratios [S1, unofficial].
- **When not to use:** When preserving exact anatomical boundaries without interpolation artifacts is critical, or when the volume has significant anisotropy that requires non-uniform scaling [S2, unofficial].

## Background
The method originates from preprocessing steps in deep learning-based volume rendering and super-resolution pipelines, specifically in the context of Diffusion Gaussian Radiance (DGR) and similar architectures [S1, unofficial]. It addresses the need for consistent input dimensions in neural networks that process volumetric data as 2D slices or 3D tensors [S1, unofficial]. The slice-wise nearest-neighbor variant is used in medical imaging contexts to handle non-uniform CT scan spacing [S2, unofficial].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
