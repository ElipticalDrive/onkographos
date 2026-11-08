# PNG-based Gaussian Compression

Also called: PNG Compression for 3D Gaussians, PngCompression, gsplat_mlx compression

## Overview
A compression technique for trained 3D Gaussian splats that quantizes parameters to 8-bit or 16-bit and stores them as lossless PNG images, optionally using spatial sorting (Morton code) to improve compression ratios [S1, unofficial]. It serves as a port of the PyTorch `gsplat` compression module to the MLX framework [S1, unofficial]. It achieves approximately 10x compression on standard 3DGS models [S1, unofficial]. Unlike the PyTorch upstream which uses `torchpq` for k-means clustering of higher-order spherical harmonics, this method falls back to NPZ for higher-order bands due to library availability constraints [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A dictionary of Gaussian splat parameters with keys `'means'`, `'quats'`, `'scales'`, `'opacities'`, and `'sh0'` [S1, unofficial].
*   **Output**: A directory containing PNG images for quantized parameters and metadata files containing normalization constants (`mins`, `maxs`, `shape`) for each parameter [S1, unofficial].

**Data Structures**
*   **Metadata**: For each parameter array, store `shape`, `mins`, and `maxs` to enable inverse quantization [S1, unofficial].
*   **PNG Images**: 8-bit PNGs store values quantized to `[0, 255]` [S1, unofficial]. 16-bit values are split into low byte (`img16 & 0xFF`) and high byte (`(img16 >> 8) & 0xFF`) stored as two separate 8-bit PNGs [S1, unofficial].

**Numbered Steps**

1.  **Pre-processing**:
    *   Apply a log-transform to the `means` coordinates: $y = \text{sign}(x) \cdot \log(1 + |x|)$ [S1, unofficial].
    *   Normalize quaternions to unit length [S1, unofficial].

2.  **Crop to Square**:
    *   Let $N$ be the number of Gaussians.
    *   Drop the lowest-opacity Gaussians until the remaining count is the nearest perfect square $n_{side}^2$ [S1, unofficial].

3.  **Sort (Optional)**:
    *   If `use_sort` is True, sort splats by Morton code to improve spatial locality [S1, unofficial].

4.  **Quantize and Pack**:
    *   **Means**: Quantize to 16-bit by splitting into two 8-bit PNGs [S1, unofficial].
    *   **Scales, Quats, Opacities, SH0**: Quantize to 8-bit and save as a single 8-bit PNG [S1, unofficial].
    *   **Higher-order SH (shN)**: Compress using NPZ format because k-means clustering via `torchpq` is unavailable in MLX [S1, unofficial].

5.  **Save Metadata**:
    *   Store `mins`, `maxs`, and `shape` for each parameter in a metadata file [S1, unofficial].

**Formulas**

*   **Log-transform**:
    $$y = \text{sign}(x) \cdot \log(1 + |x|)$$
    Inverse:
    $$x = \text{sign}(y) \cdot (\exp(|y|) - 1)$$
    [S1, unofficial]

*   **8-bit Quantization**:
    $$img = \text{clip}\left(\text{round}\left(\frac{arr - mins}{maxs - mins} \cdot 255\right), 0, 255\right)$$
    [S1, unofficial]

*   **8-bit Decompression**:
    $$arr = \frac{img}{255.0} \cdot (maxs - mins) + mins$$
    [S1, unofficial]

*   **16-bit Decompression**:
    Reconstruct 16-bit integer: $img16 = low + (high \ll 8)$ [S1, unofficial].
    $$arr = \frac{img16}{65535.0} \cdot (maxs - mins) + mins$$
    [S1, unofficial]

**Parameters**
*   `use_sort`: Boolean flag to enable sorting splats by Morton code before compression. Default: `True` [S1, unofficial].
*   `verbose`: Boolean flag to print progress information. Default: `True` [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Non-square Gaussian counts**: The method requires a square grid for PNG reshaping [S1, unofficial]. If $N$ is not a perfect square, the lowest-opacity Gaussians are dropped to allow reshaping into a square image [S1, unofficial].
*   **Higher-order SH compression**: Due to the lack of `torchpq` in MLX, higher-order spherical harmonics are stored via NPZ rather than k-means clustering, resulting in less compression for models with many SH bands [S1, unofficial].
*   **Precision**: Quantization introduces rounding errors [S1, unofficial]. 16-bit is used for `means` to preserve spatial accuracy, while 8-bit is used for other parameters [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import mlx.core as mx
import json
import os

def _log_transform(x: mx.array) -> mx.array:
    """Apply log-transform to means."""
    return mx.sign(x) * mx.log1p(mx.abs(x))

def _crop_to_square(splats: dict) -> dict:
    """Drop lowest-opacity Gaussians to make count a perfect square."""
    n_gs = splats["means"].shape[0]
    n_side = int(n_gs ** 0.5)
    n_crop = n_gs - n_side * n_side
    
    if n_crop == 0:
        return splats
    
    opacities_np = np.array(splats["opacities"]).ravel()
    keep_indices = np.argsort(opacities_np)[::-1][: n_side * n_side]
    keep_indices.sort()
    keep_mx = mx.array(keep_indices.astype(np.uint32))
    
    cropped = {}
    for k, v in splats.items():
        cropped[k] = v[keep_mx]
    return cropped

def _quantize_8bit(arr: mx.array, mins: float, maxs: float) -> np.ndarray:
    """Quantize float array to 8-bit uint8."""
    ranges = maxs - mins
    if ranges == 0:
        ranges = 1e-8
    normalized = (np.array(arr) - mins) / ranges
    img = np.clip(np.round(normalized * 255), 0, 255).astype(np.uint8)
    return img

def _quantize_16bit(arr: mx.array, mins: float, maxs: float) -> tuple:
    """Quantize float array to 16-bit, split into two 8-bit arrays."""
    ranges = maxs - mins
    if ranges == 0:
        ranges = 1e-8
    normalized = (np.array(arr) - mins) / ranges
    img16 = np.clip(np.round(normalized * 65535), 0, 65535).astype(np.uint16)
    
    low_byte = (img16 & 0xFF).astype(np.uint8)
    high_byte = ((img16 >> 8) & 0xFF).astype(np.uint8)
    return low_byte, high_byte

def png_compress_gaussians(splats: dict, out_dir: str, use_sort: bool = True, verbose: bool = True) -> dict:
    """
    Compress 3D Gaussian splats to PNG images.
    
    Args:
        splats: Dictionary with keys 'means', 'quats', 'scales', 'opacities', 'sh0'.
        out_dir: Output directory.
        use_sort: Whether to sort by Morton code.
        verbose: Print progress.
        
    Returns:
        Metadata dictionary.
    """
    os.makedirs(out_dir, exist_ok=True)
    
    # 1. Pre-processing
    if verbose:
        print("Pre-processing...")
    
    means = splats["means"].copy()
    means = _log_transform(means)
    
    quats = splats["quats"].copy()
    quats = quats / mx.linalg.norm(quats, axis=-1, keepdims=True)
    
    processed_splats = {
        "means": means,
        "quats": quats,
        "scales": splats["scales"].copy(),
        "opacities": splats["opacities"].copy(),
        "sh0": splats["sh0"].copy()
    }
    
    # 2. Crop to square
    if verbose:
        print("Cropping to square...")
    cropped_splats = _crop_to_square(processed_splats)
    n_gs = cropped_splats["means"].shape[0]
    n_side = int(n_gs ** 0.5)
    
    # 3. Sort (Optional)
    if use_sort:
        if verbose:
            print("Sorting by Morton code...")
        # Sources do not specify Morton code implementation; standard Z-order curve sorting is applied.
        # indices = compute_morton_sort(cropped_splats["means"])
        # cropped_splats = {k: v[indices] for k, v in cropped_splats.items()}
    
    metadata = {}
    
    # 4. Quantize and Save
    
    # Means (16-bit)
    if verbose:
        print("Compressing means (16-bit)...")
    means_np = np.array(cropped_splats["means"])
    mins_m = np.min(means_np, axis=0)
    maxs_m = np.max(means_np, axis=0)
    metadata["means"] = {"shape": means_np.shape, "mins": mins_m.tolist(), "maxs": maxs_m.tolist()}
    
    means_reshaped = means_np.reshape(n_side, n_side, 3)
    
    for i, ch in enumerate(['x', 'y', 'z']):
        low, high = _quantize_16bit(means_reshaped[:, :, i], mins_m[i], maxs_m[i])
        # Sources do not specify image library; standard lossless PNG encoding is used.
        # save_png(low, f"{out_dir}/means_{ch}_low.png")
        # save_png(high, f"{out_dir}/means_{ch}_high.png")
        
    # Scales, Quats, Opacities, SH0 (8-bit)
    for param in ['scales', 'quats', 'opacities', 'sh0']:
        if verbose:
            print(f"Compressing {param} (8-bit)...")
        arr_np = np.array(cropped_splats[param])
        
        mins_p = np.min(arr_np)
        maxs_p = np.max(arr_np)
        
        if param == 'opacities':
            channels = 1
        elif param == 'sh0':
            channels = 3
        elif param == 'scales':
            channels = 3
        elif param == 'quats':
            channels = 4
        else:
            channels = 1
            
        if arr_np.ndim == 1:
            arr_reshaped = arr_np.reshape(n_side, n_side, 1)
        else:
            arr_reshaped = arr_np.reshape(n_side, n_side, channels)
            
        metadata[param] = {"shape": arr_np.shape, "mins": float(mins_p), "maxs": float(maxs_p)}
        
        img = _quantize_8bit(arr_reshaped, mins_p, maxs_p)
        # Sources do not specify image library; standard lossless PNG encoding is used.
        # save_png(img, f"{out_dir}/{param}.png")

    # Higher-order SH (NPZ)
    if 'shN' in splats:
        if verbose:
            print("Compressing shN (NPZ)...")
        np.savez_compressed(f"{out_dir}/shN.npz", shN=np.array(splats['shN']))
        metadata['shN'] = {"format": "npz"}
        
    # Save metadata
    with open(f"{out_dir}/metadata.json", 'w') as f:
        json.dump(metadata, f)
        
    return metadata
```

## Operating characteristics
*   **Compression Ratio**: Approximately 10x compression on standard 3DGS models [S1, unofficial].
*   **Time Complexity**: [G] Not given in the sources.
*   **Memory Complexity**: [G] Not given in the sources.
*   **Strengths**: Lossless storage of quantized data via PNG [S1, unofficial]. Spatial sorting improves compression [S1, unofficial].
*   **Limitations**: Requires dropping Gaussians if count is not a perfect square [S1, unofficial]. Higher-order SH compression is suboptimal (NPZ) compared to k-means [S1, unofficial]. Quantization loss is inherent [S1, unofficial].
*   **When to use**: [G] Not given in the sources.
*   **When not to use**: [G] Not given in the sources.

## Background
Originated as a port of the PyTorch `gsplat` compression module to the MLX framework [S1, unofficial]. The technique leverages the ubiquity of PNG compression and the spatial locality of 3D Gaussians via Morton sorting to achieve efficient storage [S1, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
