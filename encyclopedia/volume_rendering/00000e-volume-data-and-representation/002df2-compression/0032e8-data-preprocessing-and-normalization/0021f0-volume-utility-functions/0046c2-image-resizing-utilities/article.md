# Image Resizing Utilities

Also called: nerfies image_utils

## Overview
Image Resizing Utilities provide a suite of deterministic functions for manipulating image arrays, including resizing (downsampling, upsampling, rescaling), type conversion, and geometric adjustments. They solve the problem of preparing image data for neural network inputs or downstream processing by ensuring correct data types, dimensions, and divisibility constraints. Unlike generic resizing libraries, these utilities enforce strict integer-scale operations where possible to avoid interpolation artifacts and provide specific helpers like `make_divisible` to align dimensions to hardware or model requirements [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: NumPy arrays representing images (H, W, C) or (H, W); scaling factors (floats or ints); divisors (int).
- Outputs: NumPy arrays with modified dimensions, data types, or content.

**Data Structures:**
- Images are represented as standard NumPy arrays. Common dtypes include `uint8`, `uint16`, `float32`, and `float64`.
- No complex custom data structures are used; operations are vectorized or use standard library calls.

**Steps and Formulas:**

1.  **make_divisible(image, divisor)**
    -   Get height $H$ and width $W$ of the image.
    -   If $H \pmod{\text{divisor}} == 0$ and $W \pmod{\text{divisor}} == 0$, return the original image.
    -   Else, calculate new dimensions:
        $$ H_{new} = H - (H \pmod{\text{divisor}}) $$
        $$ W_{new} = W - (W \pmod{\text{divisor}}) $$
    -   Return the cropped image `image[:H_new, :W_new]`.

2.  **downsample_image(image, scale)**
    -   If `scale` == 1, return the image.
    -   Check if $H \pmod{\text{scale}} == 0$ and $W \pmod{\text{scale}} == 0$. If not, raise `ValueError`.
    -   Calculate output dimensions:
        $$ H_{out} = H // \text{scale} $$
        $$ W_{out} = W // \text{scale} $$
    -   Resize using `cv2.resize` with interpolation mode `INTER_AREA`.

3.  **upsample_image(image, scale)**
    -   If `scale` == 1, return the image.
    -   Calculate output dimensions:
        $$ H_{out} = H \times \text{scale} $$
        $$ W_{out} = W \times \text{scale} $$
    -   Resize using `cv2.resize` with interpolation mode `INTER_AREA`.

4.  **rescale_image(image, scale_factor)**
    -   If `scale_factor` <= 0, raise `ValueError`.
    -   If `scale_factor` == 1.0, return the image.
    -   If `scale_factor` is an integer, call `upsample_image(image, scale_factor)`.
    -   If $1/\text{scale\_factor}$ is an integer AND the resulting dimensions ($H \times \text{scale\_factor}$, $W \times \text{scale\_factor}$) are integers, call `downsample_image(image, int(1/scale_factor))`.
    -   Else, warn and compute target dimensions:
        $$ H_{target} = \lceil H \times \text{scale\_factor} \rceil $$
        $$ W_{target} = \lceil W \times \text{scale\_factor} \rceil $$
        Adjust $H_{target}$ and $W_{target}$ to the nearest even numbers.
    -   Resize using `cv2.resize` with interpolation mode `INTER_AREA`.

5.  **image_to_uint8(image)**
    -   If `image.dtype` is `uint8`, return the image.
    -   If `image.dtype` is not a floating-point type, raise `ValueError`.
    -   Return `(image * 255).clip(0, 255).astype(np.uint8)`.

6.  **image_to_uint16(image)**
    -   If `image.dtype` is `uint16`, return the image.
    -   If `image.dtype` is not a floating-point type, raise `ValueError`.
    -   Return `(image * 65535).clip(0, 65535).astype(np.uint16)`.

7.  **image_to_float32(image)**
    -   If `image.dtype` is `float32`, return the image.
    -   Cast to `float32`.
    -   If original dtype was `uint8`, divide by 255.
    -   If original dtype was `uint16`, divide by 65535.
    -   If original dtype was `float64` or `float16`, return as is.
    -   Else, raise `ValueError`.

8.  **variance_of_laplacian(image)**
    -   Convert image to grayscale.
    -   Apply Laplacian filter.
    -   Return the variance of the resulting pixel values.

**Pseudocode:**

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
        raise ValueError(f"Image dimensions ({h}x{w}) must be divisible by scale {scale}")
    out_h = h // scale
    out_w = w // scale
    return cv2.resize(image, (out_w, out_h), interpolation=cv2.INTER_AREA)

def upsample_image(image: np.ndarray, scale: int) -> np.ndarray:
    if scale == 1:
        return image
    h, w = image.shape[:2]
    out_h = h * scale
    out_w = w * scale
    return cv2.resize(image, (out_w, out_h), interpolation=cv2.INTER_AREA)

def rescale_image(image: np.ndarray, scale_factor: float) -> np.ndarray:
    if scale_factor <= 0:
        raise ValueError("Scale factor must be positive.")
    if scale_factor == 1.0:
        return image
    
    h, w = image.shape[:2]
    
    # Check if scale_factor is an integer
    if scale_factor == int(scale_factor):
        return upsample_image(image, int(scale_factor))
    
    # Check if 1/scale_factor is an integer and dimensions remain integer
    inv_scale = 1.0 / scale_factor
    if inv_scale == int(inv_scale):
        int_scale = int(inv_scale)
        if h % int_scale == 0 and w % int_scale == 0:
            return downsample_image(image, int_scale)
            
    # Fallback: compute new dimensions
    target_h = math.ceil(h * scale_factor)
    target_w = math.ceil(w * scale_factor)
    
    # Adjust to even numbers
    if target_h % 2 != 0:
        target_h += 1
    if target_w % 2 != 0:
        target_w += 1
        
    return cv2.resize(image, (target_w, target_h), interpolation=cv2.INTER_AREA)

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
    if np.issubdtype(image.dtype, np.integer):
        if image.dtype == np.uint8:
            return image.astype(np.float32) / 255.0
        elif image.dtype == np.uint16:
            return image.astype(np.float32) / 65535.0
        else:
            raise ValueError("Unsupported integer dtype for float32 conversion")
    elif np.issubdtype(image.dtype, np.floating):
        if image.dtype == np.float64 or image.dtype == np.float16:
            return image.astype(np.float32)
        else:
            return image.astype(np.float32)
    else:
        raise ValueError("Unsupported dtype for float32 conversion")

def variance_of_laplacian(image: np.ndarray) -> float:
    # Convert to grayscale if not already
    if len(image.shape) == 3:
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    else:
        gray = image
        
    # Apply Laplacian
    laplacian = cv2.Laplacian(gray, cv2.CV_64F)
    return np.var(laplacian)
```

## Subtopics

### [Depth Estimation Data Resizing](12247-depth-estimation-data-resizing.md)
A preprocessing utility to resize depth estimation samples (image, disparity, mask) to a target size while optionally maintaining aspect ratio and ensuring dimensions are multiples of a specific value. It differs from generic resizing by enforcing discrete-value preservation for semantic and depth fields and by providing specific aspect-ratio constraints ('lower_bound', 'upper_bound', 'minimal') tailored for neural network input requirements.

## Operating characteristics
- **Time Complexity:** Resizing operations depend on the interpolation method. `INTER_AREA` is generally efficient for downsampling. Complexity is roughly $O(H \times W)$ for the number of pixels processed.
- **Memory Complexity:** $O(H \times W)$ for the output image.
- **Performance:** The use of `cv2.resize` with `INTER_AREA` provides a good balance between speed and quality for resizing. Integer-scale operations avoid interpolation entirely, making them very fast.
- **Strengths:** Deterministic behavior, strict type enforcement, and specific helpers for neural network input preparation (e.g., `make_divisible`).
- **Limitations:** Relies on OpenCV (`cv2`), which may not be available in all environments. The `rescale_image` function's fallback logic involves warnings and rounding, which may introduce slight inaccuracies compared to exact integer scaling.
- **When to use:** When preparing images for NeRF or neural network pipelines that require specific dimension constraints or data types.
- **When not to use:** When high-fidelity interpolation is required for artistic resizing (use `INTER_CUBIC` or `INTER_LANCZOS4` instead of `INTER_AREA`).

## Background
The utilities originate from the Google Nerfies project, a library for dynamic NeRF (Neural Radiance Fields) processing [S1, unofficial]. They were developed to handle the specific preprocessing needs of NeRF data, such as resizing images to match camera parameters and ensuring data types are compatible with PyTorch/TensorFlow models. The code is part of the `nerfies` repository, which provides tools for capturing and processing dynamic scenes [S1, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
