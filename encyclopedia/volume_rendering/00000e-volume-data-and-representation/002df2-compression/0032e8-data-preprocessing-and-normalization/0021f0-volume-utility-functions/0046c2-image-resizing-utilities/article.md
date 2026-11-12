# Image Resizing Utilities

Also called: nerfies image_utils

## Overview
Image Resizing Utilities provide a suite of deterministic functions for manipulating image tensors, including resizing (downsampling, upsampling, rescaling), type conversion, and geometric adjustments. They solve the problem of preparing image data for downstream volume rendering or NeRF training pipelines by ensuring consistent data types and dimensions. What sets this implementation apart from general-purpose libraries is its strict adherence to integer-scale operations where possible to avoid interpolation artifacts, and its integration with camera parameter scaling to maintain geometric consistency between resized images and their corresponding camera intrinsics.

## How it works
The module operates on NumPy arrays representing images. Inputs are typically 2D (grayscale) or 3D (H, W, C) arrays. Outputs are arrays of the same rank but potentially different dimensions or data types.

### Data Structures
- **Image**: A NumPy array.
  - Grayscale: shape `(H, W)`, dtype `uint8`, `uint16`, `float32`, or `float64`.
  - Color: shape `(H, W, 3)`, same dtypes.
- **Scale Factor**: A float or integer.

### Algorithms

#### 1. `make_divisible`
Ensures image dimensions are multiples of a `divisor`. This is critical for neural network architectures requiring specific stride alignments.

1.  Let $H$ be height, $W$ be width.
2.  If $H \pmod{\text{divisor}} == 0$ and $W \pmod{\text{divisor}} == 0$, return image.
3.  Else, compute:
    $$ H_{new} = H - (H \pmod{\text{divisor}}) $$
    $$ W_{new} = W - (W \pmod{\text{divisor}}) $$
4.  Return `image[:H_new, :W_new]`.

#### 2. `downsample_image`
Reduces image size by an integer factor. Uses area interpolation to preserve average intensity.

1.  Let $S$ be the scale factor (integer).
2.  If $S == 1$, return image.
3.  Check if $H \pmod S == 0$ and $W \pmod S == 0$. If not, raise `ValueError`.
4.  Compute output dimensions:
    $$ H_{out} = H // S $$
    $$ W_{out} = W // S $$
5.  Resize using `cv2.resize` with interpolation mode `INTER_AREA`.

#### 3. `upsample_image`
Increases image size by an integer factor.

1.  Let $S$ be the scale factor (integer).
2.  If $S == 1$, return image.
3.  Compute output dimensions:
    $$ H_{out} = H \times S $$
    $$ W_{out} = W \times S $$
4.  Resize using `cv2.resize` with interpolation mode `INTER_AREA`. [G] *Note: While `INTER_CUBIC` is common for upscaling, this specific utility uses `INTER_AREA` for consistency and speed, though it may appear softer.*

#### 4. `rescale_image`
General resizing by a float factor.

1.  If $S \le 0$, raise `ValueError`.
2.  If $S == 1.0$, return image.
3.  If $S$ is an integer, call `upsample_image(image, S)`.
4.  If $1/S$ is an integer AND the resulting dimensions are integers (i.e., $H \pmod S == 0$ and $W \pmod S == 0$), call `downsample_image(image, int(1/S))`.
5.  Else (non-integer scale or non-divisible dimensions):
    a. Compute raw target dimensions:
       $$ H_{raw} = H \times S $$
       $$ W_{raw} = W \times S $$
    b. Round up to nearest even integer to ensure compatibility with many downsampling kernels:
       $$ H_{out} = \lceil H_{raw} \rceil \text{ adjusted to even} $$
       $$ W_{out} = \lceil W_{raw} \rceil \text{ adjusted to even} $$
       *Implementation detail*: If $\lceil H_{raw} \rceil$ is odd, add 1.
    c. Resize using `cv2.resize` with interpolation mode `INTER_AREA`.
    d. Emit a warning that dimensions were adjusted to even numbers.

#### 5. Type Conversion Functions
These functions normalize data ranges to standard types.

**`image_to_uint8`**
1.  If `dtype == uint8`, return image.
2.  If `dtype` is not floating point, raise `ValueError`.
3.  Return `(image * 255).clip(0, 255).astype(uint8)`.

**`image_to_uint16`**
1.  If `dtype == uint16`, return image.
2.  If `dtype` is not floating point, raise `ValueError`.
3.  Return `(image * 65535).clip(0, 65535).astype(uint16)`.

**`image_to_float32`**
1.  If `dtype == float32`, return image.
2.  Cast to `float32`.
3.  If original `dtype` was `uint8`, divide by 255.
4.  If original `dtype` was `uint16`, divide by 65535.
5.  If original `dtype` was `float64` or `float16`, return as is (already float32 cast).
6.  Else raise `ValueError`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import cv2
import math

def make_divisible(image: np.ndarray, divisor: int) -> np.ndarray:
    h, w = image.shape[:2]
    if h % divisor == 0 and w % divisor == 0:
        return image
    new_h = h - (h % divisor)
    new_w = w - (w % divisor)
    return image[:new_h, :new_w]

def downsample_image(image: np.ndarray, scale: int) -> np.ndarray:
    if scale == 1:
        return image
    h, w = image.shape[:2]
    if h % scale != 0 or w % scale != 0:
        raise ValueError(f"Image dimensions {h}x{w} must be divisible by scale {scale}")
    new_h = h // scale
    new_w = w // scale
    return cv2.resize(image, (new_w, new_h), interpolation=cv2.INTER_AREA)

def upsample_image(image: np.ndarray, scale: int) -> np.ndarray:
    if scale == 1:
        return image
    h, w = image.shape[:2]
    new_h = h * scale
    new_w = w * scale
    return cv2.resize(image, (new_w, new_h), interpolation=cv2.INTER_AREA)

def rescale_image(image: np.ndarray, scale_factor: float) -> np.ndarray:
    if scale_factor <= 0:
        raise ValueError("Scale factor must be positive.")
    if scale_factor == 1.0:
        return image
    
    h, w = image.shape[:2]
    
    # Integer scale up
    if scale_factor == int(scale_factor):
        return upsample_image(image, int(scale_factor))
    
    # Integer scale down (if divisible)
    inv_scale = 1.0 / scale_factor
    if inv_scale == int(inv_scale):
        int_scale = int(inv_scale)
        if h % int_scale == 0 and w % int_scale == 0:
            return downsample_image(image, int_scale)
            
    # General case
    raw_h = h * scale_factor
    raw_w = w * scale_factor
    
    # Round up to nearest even
    new_h = int(math.ceil(raw_h))
    new_w = int(math.ceil(raw_w))
    if new_h % 2 != 0:
        new_h += 1
    if new_w % 2 != 0:
        new_w += 1
        
    # Warning omitted for brevity in pure logic, but implied by spec
    return cv2.resize(image, (new_w, new_h), interpolation=cv2.INTER_AREA)

def image_to_uint8(image: np.ndarray) -> np.ndarray:
    if image.dtype == np.uint8:
        return image
    if not np.issubdtype(image.dtype, np.floating):
        raise ValueError("Image must be floating point for conversion to uint8")
    return np.clip(image * 255, 0, 255).astype(np.uint8)

def image_to_uint16(image: np.ndarray) -> np.ndarray:
    if image.dtype == np.uint16:
        return image
    if not np.issubdtype(image.dtype, np.floating):
        raise ValueError("Image must be floating point for conversion to uint16")
    return np.clip(image * 65535, 0, 65535).astype(np.uint16)

def image_to_float32(image: np.ndarray) -> np.ndarray:
    if image.dtype == np.float32:
        return image
    
    # Cast to float32 first
    img_f32 = image.astype(np.float32)
    
    if image.dtype == np.uint8:
        return img_f32 / 255.0
    elif image.dtype == np.uint16:
        return img_f32 / 65535.0
    elif image.dtype in [np.float64, np.float16]:
        return img_f32
    else:
        raise ValueError(f"Unsupported dtype {image.dtype} for float32 conversion")
```

## Subtopics

### [Depth Estimation Data Resizing](12247-depth-estimation-data-resizing.md)
A preprocessing utility to resize depth estimation samples (image, disparity, mask) to a target size while optionally maintaining aspect ratio and ensuring dimensions are multiples of a specific value.

## Operating characteristics
- **Time Complexity**: $O(H \times W)$ for all resizing operations, as they involve iterating over pixels. Type conversion is also $O(H \times W)$.
- **Memory Complexity**: $O(H \times W)$ for the output array.
- **Performance**: `INTER_AREA` is generally faster than `INTER_CUBIC` or `INTER_LANCZOS4` but provides lower quality for significant upsampling. It is optimal for downsampling.
- **Strengths**: Deterministic behavior for integer scales; strict type enforcement prevents silent precision loss; integration with camera geometry scaling (see `Camera.rescale`).
- **Limitations**: `INTER_AREA` may produce blurry results when upsampling significantly. The "adjust to even" logic in `rescale_image` can introduce slight aspect ratio distortions if not accounted for in downstream processing.
- **When to use**: Preprocessing for NeRF/Volume Rendering pipelines where consistent tensor shapes and normalized float ranges are required.
- **When not to use**: When high-fidelity artistic upscaling is needed (use `INTER_LANCZOS4`); when exact non-even dimensions are strictly required without cropping.

## Background
This module originates from the **Nerfies** project by Google [S1, unofficial]. It was developed to handle the specific data preprocessing needs of dynamic NeRF training, where images of varying sizes and types must be normalized and resized while maintaining geometric correspondence with camera parameters. The `make_divisible` and specific resizing logic are tailored to ensure compatibility with neural network architectures that require specific stride alignments (e.g., 8x, 16x).

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
