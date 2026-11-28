# Depth Estimation Data Resizing

Also called: Resize

## Overview
A preprocessing utility to resize depth estimation samples (image, disparity, mask) to a target size while optionally maintaining aspect ratio and ensuring dimensions are multiples of a specific value [S1, unofficial]. It differs from generic resizing by enforcing discrete-value preservation for semantic and depth fields and by providing specific aspect-ratio constraints ('lower_bound', 'upper_bound', 'minimal') tailored for neural network input requirements [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A dictionary `sample` containing:
  - `image`: The RGB or grayscale image to resize [S1, unofficial].
  - `disparity` (optional): Disparity map [S1, unofficial].
  - `depth` (optional): Depth map [S1, unofficial].
  - `mask` (optional): Binary or segmentation mask [S1, unofficial].
  - `semseg_mask` (optional): Semantic segmentation mask handled as a float tensor with batch/channel dimensions [S1, unofficial].
- **Parameters**:
  - `target_height` (`int`): Desired output height [S1, unofficial].
  - `target_width` (`int`): Desired output width [S1, unofficial].
  - `resize_method` (`str`): Aspect ratio handling strategy. Options: `'lower_bound'`, `'upper_bound'`, `'minimal'` [S1, unofficial].
  - `ensure_multiple_of` (`int`): Integer constraining output width and height to be multiples of this value. Defaults to 1 [S1, unofficial].
  - `image_interpolation_method` (`int`): Interpolation method for the image, defaulting to `cv2.INTER_AREA` [S1, unofficial].
  - `keep_aspect_ratio` (`bool`): Whether to preserve the original aspect ratio [S1, unofficial].
  - `resize_target` (`bool`): Whether to resize auxiliary fields (`disparity`, `depth`, `mask`, `semseg_mask`) [S1, unofficial].
- **Output**: The modified `sample` dictionary with resized fields [S1, unofficial].

**Data Structures**
- The `sample` dictionary fields are resized in-place or replaced with new arrays of shape `(new_height, new_width)` for 2D data, or `(B, C, new_height, new_width)` for tensor-based semantic masks [S1, unofficial].

**Steps**
1. **Constrain to Multiple**: The helper `constrain_to_multiple_of` rounds `x` to the nearest multiple of `self.__multiple_of` [S1, unofficial]. If the result exceeds `max_val`, it floors to the previous multiple [S1, unofficial]. If it is below `min_val`, it ceils to the next multiple [S1, unofficial].
2. **Determine Scale**: Calculate `scale_height = self.__height / height` and `scale_width = self.__width / width` [S1, unofficial].
3. **Apply Aspect Ratio Constraint**: If `keep_aspect_ratio` is true, adjust scales based on `resize_method`: 'lower_bound' fits the larger dimension to the target (minimizing scaling), 'upper_bound' fits the smaller dimension (maximizing scaling), 'minimal' picks the dimension closest to 1:1 scale [S1, unofficial].
4. **Calculate New Dimensions**: Based on `resize_method`, calculate `new_height` and `new_width` using `constrain_to_multiple_of` [S1, unofficial]. 'lower_bound' enforces `min_val`, 'upper_bound' enforces `max_val`, 'minimal' has no bounds [S1, unofficial].
5. **Resize Image**: Resize `sample['image']` to `(width, height)` using `cv2.resize` with `self.__image_interpolation_method` [S1, unofficial].
6. **Resize Targets**: If `resize_target` is true, resize `disparity`, `depth`, and `mask` fields using `cv2.INTER_NEAREST` [S1, unofficial]. Resize `semseg_mask` using `F.interpolate` with mode 'nearest' on a float tensor with added batch/channel dimensions [S1, unofficial].

**Formulas**
- Scale factors:
  $$ \text{scale\_height} = \frac{\text{target\_height}}{\text{height}} $$
  $$ \text{scale\_width} = \frac{\text{target\_width}}{\text{width}} $$
- New dimensions (before multiple constraint):
  $$ \text{new\_height} = \text{height} \times \text{scale} $$
  $$ \text{new\_width} = \text{width} \times \text{scale} $$
- Constraint helper:
  $$ \text{nearest} = \text{round}\left(\frac{x}{\text{multiple\_of}}\right) \times \text{multiple\_of} $$
  $$ \text{result} = \begin{cases} \text{floor}\left(\frac{x}{\text{multiple\_of}}\right) \times \text{multiple\_of} & \text{if } \text{nearest} > \text{max\_val} \\ \text{ceil}\left(\frac{x}{\text{multiple\_of}}\right) \times \text{multiple\_of} & \text{if } \text{nearest} < \text{min\_val} \\ \text{nearest} & \text{otherwise} \end{cases} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import cv2
import torch.nn.functional as F

def constrain_to_multiple_of(x, min_val, max_val, multiple_of):
    """Round x to the nearest multiple of multiple_of, bounded by min_val and max_val."""
    nearest = round(x / multiple_of) * multiple_of
    if nearest > max_val:
        nearest = math.floor(x / multiple_of) * multiple_of
    elif nearest < min_val:
        nearest = math.ceil(x / multiple_of) * multiple_of
    return nearest

def resize_depth_estimation_data(sample, target_height, target_width, 
                                 resize_method='lower_bound', 
                                 ensure_multiple_of=1, 
                                 image_interpolation_method=cv2.INTER_AREA,
                                 keep_aspect_ratio=True,
                                 resize_target=True):
    """
    Resize depth estimation samples to target size with aspect ratio and multiple constraints.
    """
    if resize_method not in ('lower_bound', 'upper_bound', 'minimal'):
        raise ValueError(f"Unknown value for resize_method: {resize_method}")

    height, width = sample['image'].shape[:2]
    
    # Determine Scale
    scale_height = target_height / height
    scale_width = target_width / width
    
    new_height = height
    new_width = width
    
    if keep_aspect_ratio:
        if resize_method == 'lower_bound':
            scale = max(scale_height, scale_width)
        elif resize_method == 'upper_bound':
            scale = min(scale_height, scale_width)
        elif resize_method == 'minimal':
            scale = scale_height if abs(scale_height - 1.0) < abs(scale_width - 1.0) else scale_width
            
        new_height = height * scale
        new_width = width * scale
        
        # Apply multiple constraint with method-specific bounds
        if resize_method == 'lower_bound':
            new_height = constrain_to_multiple_of(new_height, target_height, float('inf'), ensure_multiple_of)
            new_width = constrain_to_multiple_of(new_width, target_width, float('inf'), ensure_multiple_of)
        elif resize_method == 'upper_bound':
            new_height = constrain_to_multiple_of(new_height, 0, target_height, ensure_multiple_of)
            new_width = constrain_to_multiple_of(new_width, 0, target_width, ensure_multiple_of)
        elif resize_method == 'minimal':
            new_height = constrain_to_multiple_of(new_height, 0, float('inf'), ensure_multiple_of)
            new_width = constrain_to_multiple_of(new_width, 0, float('inf'), ensure_multiple_of)
    else:
        new_height = constrain_to_multiple_of(target_height, 0, float('inf'), ensure_multiple_of)
        new_width = constrain_to_multiple_of(target_width, 0, float('inf'), ensure_multiple_of)

    # Resize Image
    sample['image'] = cv2.resize(sample['image'], (int(new_width), int(new_height)), interpolation=image_interpolation_method)
    
    # Resize Targets
    if resize_target:
        if 'disparity' in sample:
            sample['disparity'] = cv2.resize(sample['disparity'], (int(new_width), int(new_height)), interpolation=cv2.INTER_NEAREST)
        if 'depth' in sample:
            sample['depth'] = cv2.resize(sample['depth'], (int(new_width), int(new_height)), interpolation=cv2.INTER_NEAREST)
        if 'mask' in sample:
            sample['mask'] = cv2.resize(sample['mask'], (int(new_width), int(new_height)), interpolation=cv2.INTER_NEAREST)
        if 'semseg_mask' in sample:
            # Add batch and channel dimensions if missing
            x = sample['semseg_mask']
            if x.ndim == 2:
                x = x.unsqueeze(0).unsqueeze(0).float()
            elif x.ndim == 3:
                x = x.unsqueeze(0).float()
            else:
                x = x.float()
            # Standard interpolation call
            sample['semseg_mask'] = F.interpolate(x, size=(int(new_height), int(new_width)), mode='nearest').squeeze(0)

    return sample
```

**Parameters with Typical Values**
- `resize_method`: `'lower_bound'` is common for ensuring minimum input size for networks [S1, unofficial].
- `ensure_multiple_of`: `1` by default, but often `8` or `16` for CNNs with downsampling [S1, unofficial].
- `image_interpolation_method`: `cv2.INTER_AREA` for downsampling, `cv2.INTER_LINEAR` for upsampling [S1, unofficial].

**Edge Cases and Pitfalls**
- Disparity, depth, and mask fields must use `cv2.INTER_NEAREST` to preserve discrete values and avoid interpolation artifacts [S1, unofficial].
- If `resize_method` is not one of `'lower_bound'`, `'upper_bound'`, or `'minimal'`, a `ValueError` is raised [S1, unofficial].
- Boundary conditions: `constrain_to_multiple_of` handles cases where the rounded value exceeds `max_val` or falls below `min_val` [S1, unofficial].

## Subtopics

### Input Padder
A utility class that pads input images so their height and width dimensions are divisible by a specified integer (default 8), used to satisfy network input constraints [S2, unofficial]. It initializes with dimensions `dims` and optional `mode` ('sintel' or other) and `divis_by` (default 8) [S2, unofficial]. It calculates padding height `pad_ht = (((self.ht // divis_by) + 1) * divis_by - self.ht) % divis_by` and width `pad_wd = (((self.wd // divis_by) + 1) * divis_by - self.wd) % divis_by` [S2, unofficial]. It sets padding list `self._pad` based on mode: if mode is 'sintel', `self._pad = [pad_wd//2, pad_wd - pad_wd//2, pad_ht//2, pad_ht - pad_ht//2]`; otherwise, `self._pad = [pad_wd//2, pad_wd - pad_wd//2, 0, pad_ht]` [S2, unofficial]. The `pad` method applies `F.pad` with `mode='replicate'` using `self._pad` to all input tensors, asserting they are 4D [S2, unofficial]. The `unpad` method reverses padding by slicing the tensor `x` using indices `c = [self._pad[2], ht-self._pad[3], self._pad[0], wd-self._pad[1]]` and returning `x[..., c[0]:c[1], c[2]:c[3]]` [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of pixels in the image, $O(H \times W)$, due to the resizing operations [G].
- **Memory Complexity**: Linear with respect to the number of pixels, $O(H \times W)$, for storing the resized images [G].
- **Measured Performance**: Not given in the sources [G].
- **Scaling**: Scales linearly with image resolution [G].
- **Strengths**: Ensures dimensions are multiples of a specific value [S1, unofficial], which is crucial for neural networks with downsampling layers [G]. Preserves discrete values for depth and mask data [S1, unofficial].
- **Limitations**: Aspect ratio constraints may result in padding or cropping if not handled explicitly (though this method only resizes) [G].
- **When to use**: When preparing depth estimation data for neural network training where input dimensions must be multiples of a specific value and aspect ratio must be preserved [S1, unofficial].
- **When not**: When exact pixel-perfect resizing without aspect ratio constraints is required [G].

## Background
- **Origin**: The method is part of the Gaussian-Splatting-Toolkit, an unofficial toolkit for Gaussian Splatting [S1, unofficial].
- **History and Development**: Developed as part of the preprocessing pipeline for depth estimation data in the context of 3D Gaussian Splatting [S1, unofficial].
- **Usage**: Used in preprocessing steps for training neural networks on depth estimation tasks [S1, unofficial].

## Sources
[S1] Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, https://github.com/Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, fetched 2026-10-27 (unofficial)
[S2] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
