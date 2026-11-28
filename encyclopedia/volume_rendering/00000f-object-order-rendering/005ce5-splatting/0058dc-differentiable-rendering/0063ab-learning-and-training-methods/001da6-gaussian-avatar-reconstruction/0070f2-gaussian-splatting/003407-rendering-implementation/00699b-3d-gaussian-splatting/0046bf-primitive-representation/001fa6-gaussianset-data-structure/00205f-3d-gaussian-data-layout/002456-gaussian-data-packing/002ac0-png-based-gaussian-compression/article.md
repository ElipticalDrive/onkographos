# PNG-based Gaussian Compression

Also called: Compact 3D Scene Representation via Self-Organizing Gaussian Grids, PNG Compression for 3D Gaussians, PNG Compression for Gaussian Splats, gsplat PNG Compression, gsplat_mlx compression

## Overview
PNG-based Gaussian Compression is a lossless compression technique for trained 3D Gaussian Splatting models that quantizes splat parameters into 8-bit and 16-bit PNG images [S2, unofficial]. It serves as a port of the PyTorch `gsplat` compression module to MLX [S2, unofficial]. By leveraging spatial locality (optionally enhanced via Morton code sorting) and PNG encoding, it achieves approximately 10x compression on standard 3DGS models [S2, unofficial]. It builds on [Gaussian Data Packing](3257-gaussian-data-packing.md) by mapping scalar and vector fields to image grids, differing from [Gaussian Splat Texture Packing](3283-gaussian-splat-texture-packing.md) which targets GPU texture formats, and [SPZ Packed Format](3926-spz-packed-format.md) which uses a different binary layout [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A dictionary of Gaussian splat parameters: `means` (N, 3), `quats` (N, 4), `scales` (N, 3), `opacities` (N, 1), and `sh0` (N, 3). Higher-order spherical harmonic coefficients `shN` are also supported [S2, unofficial].
- **Output:** A set of PNG image files for each parameter field and a `meta.json` file containing metadata (shape, dtype, mins, maxs) required for decompression [S1, S2, unofficial].

**Data Structures:**
- **Metadata (`meta.json`):** Stores `shape`, `dtype`, `mins`, and `maxs` for each compressed parameter [S2, unofficial].
- **PNG Images:**
  - 8-bit PNGs for `scales`, `quats`, `opacities`, and `sh0` [S2, unofficial].
  - 16-bit `means` are split into two 8-bit PNGs (low byte and high byte) [S2, unofficial].
  - Higher-order SH coefficients `shN` are stored in NPZ format if K-means clustering is unavailable [S2, unofficial].

**Parameters:**
- `use_sort` (bool): Whether to sort splats by Morton code before compression to improve spatial locality. Default: `True` [S2, unofficial].
- `verbose` (bool): Whether to print progress information. Default: `True` [S2, unofficial].

**Steps:**

1. **Preprocessing:**
   - Apply log-transform to `means`: $y = \text{sign}(x) \cdot \log(1 + |x|)$ [S2, unofficial].
   - Normalize `quats` to unit length: $q_{norm} = q / \|q\|$ [S1, unofficial].

2. **Crop to Square:**
   - Calculate side length: $n_{side} = \lfloor \sqrt{N} \rfloor$ [S2, unofficial].
   - Calculate number of Gaussians to drop: $n_{crop} = N - n_{side}^2$ [S2, unofficial].
   - If $n_{crop} > 0$:
     - Sort opacities in descending order.
     - Keep indices of the top $n_{side}^2$ Gaussians.
     - Apply this mask to all splat parameters [S2, unofficial].

3. **Sorting (Optional):**
   - If `use_sort` is `True`, sort the remaining Gaussians by their Morton code to enhance spatial locality [S2, unofficial].

4. **Quantization and PNG Encoding:**
   - **8-bit Parameters (`scales`, `quats`, `opacities`, `sh0`):**
     - Reshape parameter array to $(n_{side}, n_{side}, C)$ where $C$ is the channel dimension [S1, unofficial].
     - Compute per-channel minimum ($min$) and maximum ($max$) [S1, unofficial].
     - Quantize to 8-bit: $img = \text{clip}(\text{round}((arr - \text{mins}) / \text{ranges} \cdot 255), 0, 255)$ where $\text{ranges} = \text{maxs} - \text{mins}$ [S2, unofficial].
     - Save as PNG [S1, unofficial].
   - **16-bit Parameters (`means`):**
     - Perform 16-bit quantization: $img16 = \text{round}((arr - \text{mins}) / \text{ranges} \cdot 65535)$ [S2, unofficial].
     - Split into low byte ($img16 \& 0xFF$) and high byte ($(img16 >> 8) \& 0xFF$), stored as two separate 8-bit PNGs [S2, unofficial].
   - **Higher-order SH (`shN`):**
     - If K-means clustering is not available (e.g., in MLX without `torchpq`), compress using NPZ format [S2, unofficial].

5. **Metadata Storage:**
   - Save `shape`, `dtype`, `mins`, and `maxs` for each parameter in `meta.json` [S2, unofficial].

**Decompression:**
1. Load `meta.json` [S1, unofficial].
2. Load PNG images and reconstruct quantized arrays [S1, unofficial].
3. Inverse quantization:
   - 8-bit: $arr = \text{normalised} \cdot (\text{maxs} - \text{mins}) + \text{mins}$, where $\text{normalised} = img / 255.0$ [S2, unofficial].
   - 16-bit: Reconstruct 16-bit integer from two PNGs, then $arr = \text{normalised} \cdot (\text{maxs} - \text{mins}) + \text{mins}$, where $\text{normalised} = img16 / 65535.0$ [S2, unofficial].
4. Apply inverse log-transform to `means`: $y = \text{sign}(x) \cdot (\exp(|x|) - 1)$ [S2, unofficial].
5. Normalize `quats` if necessary (reconstruction should preserve unit length if quantization is precise) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import json
import os
from PIL import Image

def log_transform(x):
    """Apply log-transform: y = sign(x) * log(1 + |x|)"""
    return np.sign(x) * np.log1p(np.abs(x))

def inverse_log_transform(x):
    """Apply inverse log-transform: y = sign(x) * (exp(|x|) - 1)"""
    return np.sign(x) * (np.exp(np.abs(x)) - 1)

def crop_to_square(splats):
    """Crop splats to nearest perfect square by dropping lowest opacity."""
    n_gs = splats["means"].shape[0]
    n_side = int(np.sqrt(n_gs))
    n_crop = n_gs - n_side * n_side
    if n_crop == 0:
        return splats
    opacities = splats["opacities"].ravel()
    keep = np.argsort(opacities)[::-1][:n_side * n_side]
    keep.sort()
    cropped = {k: v[keep] for k, v in splats.items()}
    return cropped

def save_png(arr, path):
    """Save numpy array as PNG."""
    Image.fromarray(arr).save(path)

def load_png(path):
    """Load PNG file to numpy array."""
    return np.array(Image.open(path))

def compress_png_8bit(arr, mins, maxs):
    """Compress array to 8-bit PNG using evidence formula."""
    ranges = maxs - mins
    ranges[ranges == 0] = 1.0  # Guard against division by zero [G]
    img = np.clip(np.round((arr - mins) / ranges * 255), 0, 255).astype(np.uint8)
    return img

def decompress_png_8bit(img, mins, maxs):
    """Decompress 8-bit PNG to array."""
    normalized = img / 255.0
    return normalized * (maxs - mins) + mins

def compress_png_16bit(arr, mins, maxs):
    """Compress array to two 8-bit PNGs (low and high bytes)."""
    ranges = maxs - mins
    ranges[ranges == 0] = 1.0
    img16 = np.clip(np.round((arr - mins) / ranges * 65535), 0, 65535).astype(np.uint16)
    low_byte = (img16 & 0xFF).astype(np.uint8)
    high_byte = ((img16 >> 8) & 0xFF).astype(np.uint8)
    return low_byte, high_byte

def decompress_png_16bit(low_byte, high_byte, mins, maxs):
    """Decompress two 8-bit PNGs to array."""
    img16 = (high_byte.astype(np.uint16) << 8) | low_byte.astype(np.uint16)
    normalized = img16 / 65535.0
    return normalized * (maxs - mins) + mins

def compress_gaussians_png(splats, use_sort=True, out_dir="output"):
    """Compress Gaussian splats to PNGs."""
    os.makedirs(out_dir, exist_ok=True)
    splats["means"] = log_transform(splats["means"])
    norms = np.linalg.norm(splats["quats"], axis=-1, keepdims=True)
    splats["quats"] = splats["quats"] / norms
    splats = crop_to_square(splats)
    n_side = int(np.sqrt(splats["means"].shape[0]))

    if use_sort:
        # Morton code sorting would be applied here [G]
        pass

    meta = {}
    for param in ["scales", "quats", "opacities", "sh0"]:
        arr = splats[param]
        C = arr.shape[-1]
        arr_reshaped = arr.reshape(n_side, n_side, C)
        mins = np.min(arr_reshaped, axis=(0, 1))
        maxs = np.max(arr_reshaped, axis=(0, 1))
        img = compress_png_8bit(arr_reshaped, mins, maxs)
        save_png(img, os.path.join(out_dir, f"{param}.png"))
        meta[param] = {"shape": arr.shape, "dtype": "uint8", "mins": mins.tolist(), "maxs": maxs.tolist()}

    param = "means"
    arr = splats[param]
    C = arr.shape[-1]
    arr_reshaped = arr.reshape(n_side, n_side, C)
    mins = np.min(arr_reshaped, axis=(0, 1))
    maxs = np.max(arr_reshaped, axis=(0, 1))
    low_byte, high_byte = compress_png_16bit(arr_reshaped, mins, maxs)
    save_png(low_byte, os.path.join(out_dir, f"{param}_low.png"))
    save_png(high_byte, os.path.join(out_dir, f"{param}_high.png"))
    meta[param] = {"shape": arr.shape, "dtype": "uint16", "mins": mins.tolist(), "maxs": maxs.tolist()}

    if "shN" in splats:
        np.savez_compressed(os.path.join(out_dir, "shN.npz"), splats["shN"])
        meta["shN"] = {"dtype": "npz"}

    with open(os.path.join(out_dir, "meta.json"), "w") as f:
        json.dump(meta, f)

def decompress_gaussians_png(out_dir="output"):
    """Decompress Gaussian splats from PNGs."""
    with open(os.path.join(out_dir, "meta.json"), "r") as f:
        meta = json.load(f)
    splats = {}
    for param in ["scales", "quats", "opacities", "sh0"]:
        info = meta[param]
        img = load_png(os.path.join(out_dir, f"{param}.png"))
        mins = np.array(info["mins"])
        maxs = np.array(info["maxs"])
        arr = decompress_png_8bit(img, mins, maxs)
        arr = arr.reshape(info["shape"])
        splats[param] = arr
    param = "means"
    info = meta[param]
    low_byte = load_png(os.path.join(out_dir, f"{param}_low.png"))
    high_byte = load_png(os.path.join(out_dir, f"{param}_high.png"))
    mins = np.array(info["mins"])
    maxs = np.array(info["maxs"])
    arr = decompress_png_16bit(low_byte, high_byte, mins, maxs)
    arr = arr.reshape(info["shape"])
    splats[param] = inverse_log_transform(arr)
    if "shN" in meta:
        data = np.load(os.path.join(out_dir, "shN.npz"))
        splats["shN"] = data["arr_0"]
    return splats
```

**Edge Cases and Pitfalls:**
- If the number of Gaussians is not a perfect square, the lowest opacity Gaussians are removed, potentially losing data [S1, S2, unofficial].
- Higher-order SH coefficients (`shN`) are compressed via NPZ rather than K-means clustering if the upstream dependency `torchpq` is not available, resulting in less compression [S2, unofficial].
- Division by zero in quantization must be handled if `max == min` for a channel [G].

## Subtopics

### [Gaussian Splat Chunk Compression with Morton Reordering](12243-gaussian-splat-chunk-compression-with-morton-reordering.md)
This method reduces the storage size of trained 3D Gaussian Splatting models by reordering splats into spatially local chunks, normalizing data within each chunk to a 0..1 range, and storing the results in GPU textures using compression formats. It builds on [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md) by introducing spatial locality via Morton ordering to improve the efficiency of block-based texture compression (BCn). Unlike direct compression of raw data arrays, which suffers from poor compression ratios due to spatial discontinuity, this method aligns data with hardware compression block structures.

## Operating characteristics
- **Compression Ratio:** Approximately 10x compression on standard 3DGS models [S2, unofficial].
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Requires storing the original data during preprocessing and the compressed PNGs. Metadata is minimal [S2, unofficial].
- **Strengths:** Lossless compression of quantized data, simple file format (PNG), good compression ratios due to spatial locality [S2, unofficial].
- **Limitations:** Requires cropping to perfect square, higher-order SH compression is less efficient without K-means, PNG is not optimized for GPU texture access [S2, unofficial].
- **When to Use:** When storage size is a concern and random access to individual Gaussians is not required (or can be handled via Morton code indexing) [S2, unofficial].
- **When Not to Use:** When GPU texture compression (BCn) is preferred for rendering performance, or when higher-order SH coefficients are significant and K-means is not available [S2, unofficial].

## Background
PNG-based Gaussian Compression is a technique for compressing trained 3D Gaussian Splatting models by quantizing parameters into 8-bit and 16-bit PNG images [S2, unofficial]. It was originally developed as part of the `gsplat` library for PyTorch and later ported to MLX by the `gsplat-mlx` project [S2, unofficial]. The method leverages the spatial coherence of Gaussian splats to improve compression ratios [S2, unofficial].

## Sources
[S1] nerfstudio-project/gsplat, https://github.com/nerfstudio-project/gsplat, fetched 2026-10-27 (unofficial)
[S2] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
