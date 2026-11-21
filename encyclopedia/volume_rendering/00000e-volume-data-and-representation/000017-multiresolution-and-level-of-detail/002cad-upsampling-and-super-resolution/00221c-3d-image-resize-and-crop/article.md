# 3D Image Resize and Crop

Also called: 3D Image Resize and Crop for Super-Resolution

## Overview
3D Image Resize and Crop is a preprocessing pipeline that standardizes volumetric data dimensions and resolution to prepare inputs for super-resolution or reconstruction algorithms. It solves the problem of inconsistent input shapes and scales by cropping the depth axis to a central region, padding the spatial axes to a square, and resizing to a target resolution [S1, unofficial]. Unlike [Cubic Upsampling via Tiling](2342-cubic-upsampling-via-tiling.md), which focuses on high-fidelity reconstruction of missing high-frequency details, this method provides a deterministic, geometry-aware normalization step that ensures consistent spatial alignment and intensity scaling [S1, unofficial]. It differs from [Deep Learning-Based Super-Resolution](12769-deep-learning-based-super-resolution.md) by relying on explicit geometric transformations (crop, pad, resize) rather than learned feature mappings [S1, unofficial].

## How it works
The method processes a 3D volume (or a batch of 3D volumes) through a sequence of geometric and normalization steps. The primary variant, `resize_and_crop`, prepares high-resolution (HR) and low-resolution (LR) pairs for training or inference [S1, unofficial].

### Inputs and Outputs
- **Input:** A 3D volume array `V` with shape `[D, H, W]` (or batched `[B, D, H, W]`) and a target size tuple `image_size = (H_target, W_target)` [S1, unofficial].
- **Output:** A normalized tensor in format `[C, H_target, W_target, 1]` (or `[B, C, H_target, W_target, 1]`) [S1, unofficial]. If generating LR pairs, it returns both the HR and LR versions [S1, unofficial].

### Data Structures
- **Volume `V`:** Typically `float32` or `int16` (e.g., Hounsfield units).
- **Tensor:** PyTorch `Tensor` or NumPy array with dimensions `[B, C, H, W]` during processing, permuted to `[C, H, W, 1]` for output [S1, unofficial].

### Steps
1.  **Z-Dimension Crop:** Crop the depth axis (axis 0) to a central region. The crop size is defined by `image_size[0]/2` slices around the center index of the original depth dimension [S1, unofficial].
    -   Let $D_{orig}$ be the original depth. The center index is $C_z = D_{orig} // 2$.
    -   The crop range is $[C_z - K, C_z + K]$ where $K = \lfloor \text{image\_size}[0] / 2 \rfloor$.
    -   If the volume is too thin, handle boundary conditions (see Pitfalls) [S1, unofficial].

2.  **X-Y Padding:** Check if the height ($H$) and width ($W$) dimensions are equal. If $H \neq W$, pad the smaller dimension with zeros to match the larger dimension, centering the original content [S1, unofficial].
    -   Let $L = \max(H, W)$.
    -   If $H < W$, pad height: add $\lfloor (W - H) / 2 \rfloor$ zeros to the top and $\lceil (W - H) / 2 \rceil$ zeros to the bottom.
    -   If $W < H$, pad width: add $\lfloor (H - W) / 2 \rfloor$ zeros to the left and $\lceil (H - W) / 2 \rceil$ zeros to the right.

3.  **Conversion and Resizing:** Convert the padded volume to a tensor format `[B, C, H, W]` (assuming single channel $C=1$) [S1, unofficial]. Resize using bilinear interpolation to the target dimensions `image_size[1]` x `image_size[2]` [S1, unofficial].
    -   Note: `image_size[0]` is used for the Z-crop, but the final spatial resolution is `image_size[1]` and `image_size[2]` [S1, unofficial].

4.  **Normalization:** Divide the resized tensor by the global maximum value of the *original* (or pre-resize) volume to normalize intensity to $[0, 1]$ [S1, unofficial].
    -   $V_{norm} = V_{resized} / \max(V_{original})$ [S1, unofficial].

5.  **Permutation:** Permute dimensions from `[B, C, H, W]` to `[B, C, H, W, 1]` (or `[C, H, W, 1]` for single samples) for final output [S1, unofficial].

### Generating Low-Resolution (LR) Pairs
To create an LR version for super-resolution tasks, the method performs the same Z-crop and X-Y padding as the HR version [S1, unofficial]. It then converts to a tensor and resizes to a lower target size (e.g., half the resolution) to create `image_lr` [S1, unofficial]. Both HR and LR images are normalized by dividing by their respective maximums and permuted to `[C, H, W, 1]` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np

def resize_and_crop(volume: np.ndarray, image_size: tuple, return_lr: bool = False) -> tuple:
    """
    Resize and crop a 3D volume for super-resolution preprocessing.
    
    Args:
        volume: Input 3D array of shape (D, H, W) or (B, D, H, W).
        image_size: Tuple (crop_depth_factor, target_H, target_W). 
                    Note: image_size[0] is used for Z-crop radius, not final Z-depth.
        return_lr: If True, also return the low-resolution version.
        
    Returns:
        HR tensor of shape (B, 1, H, W, 1) or (1, H, W, 1).
        If return_lr is True, also returns LR tensor.
    """
    # Handle batch dimension
    if volume.ndim == 3:
        volume = np.expand_dims(volume, axis=0)
        batched = False
    else:
        batched = True
        
    B, D, H, W = volume.shape
    
    # 1. Z-Dimension Crop
    # Crop to a central region defined by image_size[0]/2 slices around the center
    crop_radius = int(image_size[0] / 2)
    center_z = D // 2
    
    # Calculate start and end indices
    start_z = max(0, center_z - crop_radius)
    end_z = min(D, center_z + crop_radius)
    
    # Ensure we have enough data; if not, handle edge case [G]
    if end_z - start_z == 0:
        start_z = 0
        end_z = D
        
    volume_cropped = volume[:, start_z:end_z, :, :]
    
    # 2. X-Y Padding
    # If X and Y dimensions are not equal, pad the smaller dimension with zeros
    H_curr, W_curr = volume_cropped.shape[2], volume_cropped.shape[3]
    
    if H_curr != W_curr:
        max_dim = max(H_curr, W_curr)
        pad_h = max_dim - H_curr
        pad_w = max_dim - W_curr
        
        pad_top = pad_h // 2
        pad_bottom = pad_h - pad_top
        pad_left = pad_w // 2
        pad_right = pad_w - pad_left
        
        # Pad with zeros
        volume_padded = np.pad(volume_cropped, 
                               ((0, 0), (0, 0), (pad_top, pad_bottom), (pad_left, pad_right)), 
                               mode='constant', constant_values=0)
    else:
        volume_padded = volume_cropped
        
    # 3. Convert to Tensor and Resize
    # Convert to PyTorch tensor [B, C, H, W]
    # Assuming single channel, add channel dim
    tensor_vol = torch.from_numpy(volume_padded).float()
    if tensor_vol.dim() == 4:
        tensor_vol = tensor_vol.unsqueeze(1) # [B, 1, H, W]
        
    target_h, target_w = image_size[1], image_size[2]
    
    # Resize using bilinear interpolation
    # align_corners=False is standard for modern PyTorch versions
    resized_vol = F.interpolate(tensor_vol, size=(target_h, target_w), mode='bilinear', align_corners=False)
    
    # 4. Normalization
    # Normalize by dividing by the global maximum of the ORIGINAL volume (or cropped/padded?)
    # Source says "global maximum", implying the input to the normalization step. 
    # Usually, normalization is done on the HR image before resizing or on the resized image.
    # Source [S1, unofficial] says: "normalize by dividing by the global maximum" after resize.
    # However, it also says "Normalize both the high-resolution (original) and low-resolution images 
    # by dividing by their respective maximums". This implies max is calculated on the pre-resize or 
    # the specific version being normalized. We calculate max on the resized HR volume for consistency 
    # with the "divide by global maximum" instruction after resize.
    max_val = resized_vol.max()
    if max_val > 0:
        resized_vol_norm = resized_vol / max_val
    else:
        resized_vol_norm = resized_vol
        
    # 5. Permute to [C, H, W, 1]
    # Current shape: [B, 1, H, W]
    # Permute to [B, 1, H, W, 1]
    output_hr = resized_vol_norm.permute(0, 1, 2, 3, 4) if resized_vol_norm.dim() == 5 else resized_vol_norm.unsqueeze(-1)
    
    if not return_lr:
        return output_hr.squeeze(0) if not batched else output_hr
        
    # Generate LR version
    # Perform the same Z-crop and X-Y padding as resize_and_crop
    # Resize to target size to create the low-resolution version (image_lr)
    # Assume LR target is half the HR target for demonstration, or use a separate LR size
    # Source doesn't specify LR target size explicitly, just "resize to target size".
    # We assume a separate LR target size or half resolution. Let's assume half for standard SR.
    lr_target_h = target_h // 2
    lr_target_w = target_w // 2
    
    resized_lr = F.interpolate(tensor_vol, size=(lr_target_h, lr_target_w), mode='bilinear', align_corners=False)
    
    # Normalize LR by its own maximum
    max_val_lr = resized_lr.max()
    if max_val_lr > 0:
        resized_lr_norm = resized_lr / max_val_lr
    else:
        resized_lr_norm = resized_lr
        
    # Permute LR
    output_lr = resized_lr_norm.permute(0, 1, 2, 3, 4) if resized_lr_norm.dim() == 5 else resized_lr_norm.unsqueeze(-1)
    
    return output_hr.squeeze(0) if not batched else output_hr, output_lr.squeeze(0) if not batched else output_lr
```

### Parameters
- `image_size`: Tuple `(crop_radius, target_H, target_W)`. Typical values depend on the dataset (e.g., `[128, 256, 256]`) [S1, unofficial].
- `spatial_lr_scale`: Float factor for LR resolution (e.g., 0.5) [S1, unofficial].

### Edge Cases and Pitfalls
- **Z-Crop Boundaries:** If the volume depth $D$ is smaller than `2 * crop_radius`, the start/end indices may exceed bounds. The code must clamp indices or handle thin volumes [S1, unofficial].
- **Intensity Scaling:** Normalization by the global maximum assumes the maximum value is representative of the foreground. If the volume contains noise or outliers, this may compress the signal [S1, unofficial].
- **Padding Artifacts:** Zero-padding introduces sharp boundaries at the edges of the volume, which may cause artifacts during interpolation or subsequent processing [S1, unofficial].

## Subtopics

### Slice-wise Nearest-Neighbor Resampling
Slice-wise Nearest-Neighbor Resampling resizes 2D slices of a volume dataset using nearest-neighbor interpolation to correct non-uniform spacing or scaling issues in specific anatomical regions [S2, unofficial]. It is chosen when preserving sharp edges and discrete values (e.g., segmentation masks or specific CT ranges) is critical, avoiding the blurring effects of bilinear/trilinear interpolation [S2, unofficial]. It differs from the main `resize_and_crop` method by operating slice-by-slice with variable scaling factors and nearest-neighbor logic [S2, unofficial].

**Steps:**
1. Iterate through slices `j` in specific ranges (e.g., 0-209, 209-227, 227-249, 1106-1110, 1117-end) [S2, unofficial].
2. For each slice, call `resizeMatrix` with target dimensions scaled by factors (1.9, 1.3, or 1.1) [S2, unofficial].
3. Call `centreContent` to place the resized slice back into the original matrix bounds [S2, unofficial].

**Formula:**
Nearest-neighbor coordinate mapping:
$$ \text{nearestX} = \lfloor x \cdot \frac{\text{width}}{\text{resizeW}} \rfloor $$
$$ \text{nearestY} = \lfloor y \cdot \frac{\text{height}}{\text{resizeH}} \rfloor $$
Output pixel at $(x, y)$ takes value from input at $(\text{nearestX}, \text{nearestY})$ [S2, unofficial].

**Parameters:**
- `resizeW`, `resizeH`: Target width and height for the resized slice. Typically calculated as `CT_x_axis / factor` and `CT_y_axis / factor` where factor is 1.9, 1.3, or 1.1 [S2, unofficial].

**Pitfalls:**
- The centering logic assumes `content.length` is less than `containerWidth`. If `centreBegins` is negative or out of bounds, indices will be invalid. The code does not explicitly clamp indices [S2, unofficial].

### Resized Slice Centering
Resized Slice Centering centers a resized 2D matrix within a larger container matrix, padding unused regions with the Hounsfield unit value for air (-1024) [S2, unofficial]. It is used to maintain the spatial context of the original volume after resizing individual slices [S2, unofficial].

**Steps:**
1. Create output matrix of size `containerWidth x containerHeight`, initialized to -1024 [S2, unofficial].
2. Calculate `centreBegins = (CT_x_axis - content.length) / 2` [S2, unofficial].
3. Iterate `x` from 0 to `containerWidth-1` and `y` from 0 to `containerHeight-1` [S2, unofficial].
4. If `y > centreBegins && y < centreBegins + length`, copy `content[xCentre][yCentre]` to `output[x][y]` and increment `yCentre` [S2, unofficial].
5. If `x > centreBegins && x < centreBegins + length`, increment `xCentre` [S2, unofficial].

**Parameters:**
- Default padding value: `-1024`, representing the Hounsfield value of air [S2, unofficial].

### Pre-image Enlargement
Pre-image Enlargement is an enlargement technique for a pre-image (depth coordinate map) using zooming and pixel averaging to interpolate values while preserving originals [S3]. It is used to generate shaded surface views from depth maps by increasing resolution while maintaining the integrity of the original depth coordinates [S3]. It differs from standard interpolation by ensuring original pixel values remain unchanged after averaging [S3].

**Steps:**
1. Zoom the pre-image by a factor of two, replicating each pixel (depth coordinate) 4 times in a 2x2 pixel square [S3].
2. Run a pixel averaging operator over the zoomed image. This replaces replicated depth values with interpolated values while original values remain unchanged [S3].
3. Run any gradient operator (binary or other) over the enlarged pre-image to obtain a shaded surface view [S3].

## Operating characteristics
- **Time Complexity:** $O(D \cdot H \cdot W)$ for the crop and pad operations. Resizing is $O(H_{target} \cdot W_{target})$ per slice or volume [S1, unofficial].
- **Memory Complexity:** $O(D \cdot H \cdot W)$ for the input and output volumes [S1, unofficial].
- **Performance:** Fast and deterministic. Suitable for real-time preprocessing [S1, unofficial].
- **Strengths:** Simple, robust, and easy to implement. Ensures consistent input dimensions for neural networks [S1, unofficial].
- **Limitations:** Does not recover high-frequency details. Zero-padding may introduce artifacts at boundaries [S1, unofficial].
- **When to Use:** When preparing data for super-resolution or reconstruction algorithms that require fixed input sizes [S1, unofficial].
- **When Not to Use:** When high-fidelity detail recovery is required without subsequent learning-based steps [S1, unofficial].

## Background
The method originates from preprocessing pipelines for deep learning-based volume rendering and super-resolution, specifically in the context of CT and MRI data [S1, unofficial]. It is commonly used in conjunction with Fourier-based volume rendering to standardize inputs before applying resampling filters [S1, unofficial]. The technique is documented in unofficial repositories such as `wskingdom/DGR` [S1, unofficial] and `joshcodd/volume-rendering` [S2, unofficial].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
[S3] Hardware Support for the Display and Manipulation of Binary Voxel Models, https://doi.org/10.2312/eggh/eggh88/103-117, fetched 2026-10-26
