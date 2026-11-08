# 3D Image Resize and Crop

Also called: 3D Image Resize and Crop for Super-Resolution

## Overview
This method prepares 3D volumetric data for super-resolution tasks by standardizing spatial dimensions and generating paired low-resolution (LR) and high-resolution (HR) inputs. It solves the problem of inconsistent input shapes and intensity scales in neural network training. It sets itself apart from pure interpolation methods by explicitly enforcing square spatial symmetry via zero-padding and providing synchronized LR/HR pairs for contrastive or diffusion-based learning [S1, unofficial].

## How it works
**Inputs:**
- `volume`: 3D numpy array of shape $(D, H, W)$, typically float32.
- `image_size`: Tuple $(H_{target}, W_{target})$. Note: The source code uses `image_size` for the target spatial dimensions, while the volume's Z-dimension is derived from the input data's depth.
- `spatial_lr_scale`: Float scalar defining the downsampling ratio for the LR version.

**Outputs:**
- `img_hr`: High-resolution tensor of shape $(C, H_{target}, W_{target}, 1)$.
- `img_lr`: Low-resolution tensor of shape $(C, H_{target}, W_{target}, 1)$.

**Data Structures:**
- PyTorch tensors are used for the final output.
- Intermediate processing uses numpy arrays.

**Steps:**
1.  **Z-Dimension Cropping:** Crop the volume along the Z-axis (axis 0) to a central region. The number of slices retained is determined by `image_size[0]` (which corresponds to the target height/width in the source code's logic, though semantically it acts as the crop depth). Specifically, crop to `image_size[0]` slices centered around the middle index of the original volume's depth.
    *   *Note:* The source code snippet `self.image_size = (volume.shape[0], self.image_size, self.image_size)` suggests `image_size` might be a scalar for spatial dims, but the crop logic `image_size[0]/2` implies `image_size` is a tuple where the first element defines the Z-crop size. We follow the explicit instruction: "Crop the Z-dimension... to a central region defined by image_size[0]/2 slices around the center index."
2.  **Spatial Padding (Square Enforcement):** Check if the X (axis 1) and Y (axis 2) dimensions are equal. If not, pad the smaller dimension with zeros to match the larger dimension. The original content is centered within the padded region.
3.  **Conversion and Upsampling (HR Path):**
    - Convert the cropped and padded volume to a PyTorch tensor with shape $[B, C, H, W]$. Typically $B=1, C=1$.
    - Resize using bilinear interpolation to the target spatial dimensions `image_size[1]` x `image_size[2]`.
    - Normalize the tensor by dividing by the global maximum intensity of the volume.
4.  **HR Permutation:** Permute dimensions to $[C, H, W, 1]$ for the final HR output.
5.  **LR Generation:**
    - Perform the same Z-crop and X-Y padding as in steps 1-2 on the original volume.
    - Convert to tensor.
    - Resize to the target size (which is effectively the LR resolution if `spatial_lr_scale` is applied before, or the source implies the LR version is created by resizing the *original* or *cropped* volume to a lower resolution defined by `spatial_lr_scale`). The notes state: "Convert to tensor and resize to target size to create the low-resolution version". Given `spatial_lr_scale`, the LR resolution is likely `image_size / spatial_lr_scale`. However, the note says "resize to target size". We assume the "target size" for LR is derived from `spatial_lr_scale` applied to the original spatial dimensions, or the code snippet implies a specific workflow where `image_lr` is a downsampled version. *Correction based on standard SR pipelines:* Usually, LR is the downsampled input. The note says "resize to target size to create the low-resolution version". This is ambiguous. Let's look at `spatial_lr_scale`. If `image_size` is the HR target, the LR target is `image_size / spatial_lr_scale`. The note might mean "resize to [HR_target / scale]". Let's assume the LR tensor is resized to `floor(image_size / spatial_lr_scale)`.
    - Normalize the LR image by dividing by its own global maximum.
    - Permute to $[C, H, W, 1]$.
6.  **Return:** Return `img_hr` and `img_lr`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def resize_and_crop(volume, image_size, spatial_lr_scale):
    """
    volume: np.array of shape (D, H, W)
    image_size: tuple (H_hr, W_hr)
    spatial_lr_scale: float
    """
    # 1. Z-Crop
    D, H, W = volume.shape
    center_z = D // 2
    crop_depth = image_size[0] # Assuming image_size[0] is the desired Z-depth for crop
    start_z = max(0, center_z - crop_depth // 2)
    end_z = min(D, start_z + crop_depth)
    
    # Handle case where crop_depth > D
    if crop_depth >= D:
        cropped = volume
    else:
        cropped = volume[start_z:end_z, :, :]
        
    # 2. Pad X/Y to be square
    H_c, W_c = cropped.shape[1], cropped.shape[2]
    max_dim = max(H_c, W_c)
    
    pad_h = (max_dim - H_c) // 2
    pad_w = (max_dim - W_c) // 2
    
    # Pad with zeros
    padded = np.pad(cropped, ((0, 0), (pad_h, max_dim - H_c - pad_h), (pad_w, max_dim - W_c - pad_w)), mode='constant')
    
    # 3. HR Path
    # Convert to tensor [B, C, H, W]
    hr_tensor = torch.from_numpy(padded).unsqueeze(0).unsqueeze(0).float() # [1, 1, H, W]
    
    # Resize to target HR size
    hr_resized = torch.nn.functional.interpolate(hr_tensor, size=(image_size[1], image_size[2]), mode='bilinear', align_corners=False)
    
    # Normalize by global max
    hr_max = hr_resized.max()
    if hr_max > 0:
        hr_normalized = hr_resized / hr_max
    else:
        hr_normalized = hr_resized
        
    # Permute to [C, H, W, 1]
    img_hr = hr_normalized.permute(0, 1, 3, 2).squeeze(0) # [1, H, W, 1]
    
    # 4. LR Path
    # Re-crop and pad the original volume (or use the cropped version? Notes say "Perform the same...")
    # We use the cropped/padded volume from step 2 for consistency, but resize to LR size
    lr_tensor = torch.from_numpy(padded).unsqueeze(0).unsqueeze(0).float()
    
    # Calculate LR size
    lr_h = int(image_size[1] / spatial_lr_scale)
    lr_w = int(image_size[2] / spatial_lr_scale)
    
    lr_resized = torch.nn.functional.interpolate(lr_tensor, size=(lr_h, lr_w), mode='bilinear', align_corners=False)
    
    # Normalize by LR max
    lr_max = lr_resized.max()
    if lr_max > 0:
        lr_normalized = lr_resized / lr_max
    else:
        lr_normalized = lr_resized
        
    # Permute to [C, H, W, 1]
    img_lr = lr_normalized.permute(0, 1, 3, 2).squeeze(0) # [1, lr_h, lr_w, 1]
    
    return img_hr, img_lr
```

**Parameters:**
- `spatial_lr_scale`: Typical value 2 or 4 for 2x or 4x super-resolution.
- `image_size`: Depends on the target resolution, e.g., (128, 128, 128) if Z is also resized, but here Z is cropped.

**Edge Cases:**
- If `volume.max() == 0`, normalization results in NaN or 0. Guard against division by zero [G].
- If `crop_depth` exceeds volume depth, the entire volume is used [G].
- Precision: Float32 is used in the source [S1, unofficial].

## Subtopics

### [Zero Padding for Aliasing Reduction](6382-zero-padding-for-aliasing-reduction.md)
A preprocessing step that extends the spatial volume with zeros to suppress overlapping between the central projection image and surrounding replicas in the spatial domain. It reduces aliasing artifacts caused by imperfect resampling filters by ensuring that energy in the stop band projects zeros instead of dataset data, thereby nullifying artifacts. This method sets itself apart from filter design improvements by trading increased memory usage for artifact suppression without additional computational overhead during the rendering pass.

## Operating characteristics
- **Time Complexity:** Dominated by the interpolation step, $O(H \cdot W \cdot D)$ for the initial crop/pad and $O(H_{target} \cdot W_{target})$ for the resize.
- **Memory Complexity:** $O(H \cdot W \cdot D)$ for the input volume. The padding increases memory slightly if dimensions are not square.
- **Strengths:** Ensures consistent input shapes for neural networks; handles non-square volumes gracefully; provides synchronized LR/HR pairs.
- **Limitations:** Zero-padding introduces artificial boundaries that may affect gradient-based learning if not masked; bilinear interpolation may blur high-frequency details compared to learned upsamplers.
- **When to use:** When preparing data for 3D super-resolution models that require square spatial inputs and paired LR/HR data.

## Background
Originated in the context of Deep Gaussian Splatting and 3D reconstruction pipelines, specifically within the DGR (Deep Gaussian Reconstruction) framework. The method is part of a data preparation pipeline that includes FBP reconstruction and Gaussian initialization. It is used to standardize inputs for neural networks that process volumetric data or projections.

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
