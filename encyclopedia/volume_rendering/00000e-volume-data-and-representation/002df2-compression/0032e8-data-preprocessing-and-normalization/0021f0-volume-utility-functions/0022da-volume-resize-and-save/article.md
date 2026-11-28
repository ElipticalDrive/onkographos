# Volume Resize and Save
## Overview
This method resizes a 3D volume using bilinear interpolation, normalizes intensity values to the $[0, 1]$ range, and optionally saves the data as PNG slices or a raw uint16 file [S1, unofficial][S2, unofficial]. It differs from [Image Resizing Utilities](11843-image-resizing-utilities.md) by including specific normalization and fixed rotation steps, and from [Vector Normalization](3145-vector-normalization.md) by handling 3D tensor structures and file I/O [G].
## How it works
**Inputs and Outputs:**
- `volume`: 3D array of shape $(D, H, W)$ or $(H, W, D)$ containing floating-point voxel values.
- `target_size`: Integer or tuple specifying the new dimensions.
- `save_root`: Optional string path for saving output.
- `format`: String, either `'png_slices'` or `'raw_uint16'`.
- Output: Resized and normalized volume, optionally saved to disk.

**Data Structures:**
- `volume`: Floating-point array (float32 or float64).
- `output_dir`: Directory created if `save_root` is provided.

**Steps:**
1. **Resize:** Apply bilinear interpolation to resize the volume to `target_size` [S1, unofficial].
2. **Normalize:** Scale volume values to the range $[0, 1]$ using min-max scaling [S2, unofficial].
3. **Rotate:** Rotate the volume 90 degrees around axes 1 and 2 [S1, unofficial].
4. **Save:** If `save_root` is provided, create the directory and save the data according to `format` [S1, unofficial][S2, unofficial].

**Formulas:**
Min-max normalization is applied as:
$$ v_{norm} = \frac{v - v_{min}}{v_{max} - v_{min}} $$
where $v_{min} = \min(v)$ and $v_{max} = \max(v)$ [S2, unofficial].

**Parameters:**
- `target_size`: Typically $(128, 128, 128)$ or $(96, 96, 96)$ [S2, unofficial].
- `format`: `'png_slices'` or `'raw_uint16'`.

**Edge Cases and Pitfalls:**
- Constant volume ($v_{min} == v_{max}$): Division by zero occurs; normalized values are set to $0$ [G].
- Precision: Standard floating-point arithmetic is used for normalization and scaling [G].
- Boundaries: Interpolation handles boundary conditions according to the backend implementation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import os
from scipy.ndimage import zoom

def resize_and_save(volume, target_size, save_root=None, format='png_slices'):
    # Step 1: Resize using bilinear interpolation (order=1)
    # Sources do not specify the exact interpolation backend [G]
    zoom_factors = np.array(target_size) / np.array(volume.shape)
    resized = zoom(volume, zoom_factors, order=1)
    
    # Step 2: Normalize to [0, 1] using min-max scaling
    v_min = np.min(resized)
    v_max = np.max(resized)
    
    # Guard against constant volume to avoid division by zero
    if v_max == v_min:
        normalized = np.zeros_like(resized)
    else:
        normalized = (resized - v_min) / (v_max - v_min)
        
    # Step 3: Rotate 90 degrees around axes 1 and 2
    # Sources specify rotation around axes (1, 2) [S1, unofficial]
    rotated = np.rot90(normalized, k=1, axes=(1, 2))
    
    # Step 4: Save
    if save_root is not None:
        os.makedirs(save_root, exist_ok=True)
        
    if format == 'png_slices':
        # Iterate over the depth dimension (axis 0)
        for i in range(rotated.shape[0]):
            slice_data = rotated[i]
            # Scale to [0, 255] and cast to uint8
            slice_uint8 = (slice_data * 255).astype(np.uint8)
            # Save as PNG (implementation detail omitted; standard image library used) [G]
            # e.g., imageio.imwrite(os.path.join(save_root, f'slice_{i}.png'), slice_uint8)
    elif format == 'raw_uint16':
        # Scale normalized values to [0, 65535] and cast to uint16
        vol_scaled = rotated * 65535.0
        vol_uint16 = vol_scaled.astype(np.uint16)
        # Write raw binary data
        if save_root:
            with open(os.path.join(save_root, 'volume.raw'), 'wb') as f:
                f.write(vol_uint16.tobytes())
```

## Subtopics

### ImageJ Export
This variant targets compatibility with ImageJ by normalizing and saving a reconstructed volume as a raw uint16 file [S2, unofficial]. It loads the volume from a numpy file, normalizes values to $[0, 1]$, scales them to $[0, 65535]$, casts to `uint16`, and writes a raw binary file [S2, unofficial]. The resulting file is imported into ImageJ as raw data with specified dimensions (e.g., $128 \times 128 \times 128$ or $96 \times 96 \times 96$) and type `uint16` [S2, unofficial].

## Operating characteristics
- Time complexity is dominated by interpolation and normalization, scaling linearly with the number of voxels [G].
- Memory complexity is $O(N)$ for storing the resized volume [G].
- Strengths include standardizing output for visualization tools like ImageJ [S2, unofficial].
- Limitations include loss of precision due to quantization to uint8 or uint16, and a fixed rotation that may not suit all coordinate systems [G].

## Background
This method is used in post-processing pipelines for 3D reconstruction and CT reconstruction projects to prepare data for visualization [S2, unofficial]. It originated as a utility function within deep learning-based volume rendering workflows [G].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
