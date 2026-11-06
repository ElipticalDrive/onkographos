# PlenOctree Compression

Also called: Median-cut Quantization and DEFLATE Compression, PlenOctree Compression Strategy, SH-9 PlenOctree Compression

## Overview
PlenOctree Compression is a file-size reduction strategy designed to make PlenOctrees feasible for in-browser rendering by minimizing occupied voxels and applying direct compression to the tree structure [S1, unofficial]. It distinguishes itself from general value compression [6-value-compression.md] by specifically targeting the spherical harmonic (SH) coefficients of PlenOctrees using median-cut quantization and DEFLATE encoding, and from sparse structures [5-sparse-structures.md] by compressing the entire tree including pointers rather than just the data values [S1, unofficial].

## How it works
**Inputs:** An uncompressed PlenOctree containing SH coefficients (RGB vectors) and density ($\sigma$) values [S1, unofficial].
**Outputs:** A compressed binary file containing the quantized codebooks, leaf pointers, and DEFLATE-compressed tree structure [S1, unofficial].

**Data Structures:**
1. **Codebook:** A lookup table of size $2^{16} \times 3$ storing RGB colors [S1, unofficial]. Data type: `float16` (IEEE 754 half-precision) [S1, unofficial]. One codebook is stored per SH basis function [S1, unofficial].
2. **Leaf Pointers:** For each leaf node in the PlenOctree, a pointer to an index in the corresponding SH basis function's codebook [S1, unofficial]. Data type: `int16` (16-bit signed integer) [S1, unofficial].
3. **Tree Structure:** The full PlenOctree hierarchy, including internal node pointers and leaf data (pointers and $\sigma$), compressed as a byte stream [S1, unofficial].

**Parameters:**
* **SH Degree:** Limited to degree 2 (SH-9) [S1, unofficial]. This reduces the number of coefficients per voxel compared to SH-16 or SH-25 [S1, unofficial].
* **Bounding Box:** A looser bounding box is used during the initial PlenOctree construction to reduce the total number of occupied voxels [S1, unofficial].
* **Quantization Levels:** $2^{16}$ (65,536) colors [S1, unofficial].

**Steps:**
1. **Quantize SH Coefficients:**
   * For each voxel in the PlenOctree, retain the density value $\sigma$ as a standard floating-point value (no quantization specified for $\sigma$) [S1, unofficial].
   * For each SH basis function $m$, extract the RGB coefficient vector $\mathbf{k}_m \in \mathbb{R}^3$ [S1, unofficial].
   * Apply the **median-cut algorithm** to quantize all $\mathbf{k}_m$ vectors across the entire tree into $2^{16}$ representative colors [S1, unofficial].
   * Generate a codebook of size $2^{16} \times 3$ for each SH basis function [S1, unofficial].
   * Replace each voxel's $\mathbf{k}_m$ with an index (pointer) into the corresponding codebook. The index is stored as an `int16` [S1, unofficial].
2. **Compress Tree:**
   * Serialize the entire PlenOctree structure, including internal nodes, leaf pointers (to codebooks), and $\sigma$ values [S1, unofficial].
   * Apply the **DEFLATE algorithm** (as implemented in ZLIB) to the serialized byte stream [S1, unofficial].
3. **Store:**
   * Save the DEFLATE-compressed tree stream [S1, unofficial].
   * Save the codebooks (one per SH basis function) as `float16` arrays [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import zlib
import math

def median_cut_quantize(colors, num_colors):
    """Standard median-cut color quantization [G]."""
    if not colors:
        return [[0.0, 0.0, 0.0] for _ in range(num_colors)]
    boxes = [list(range(len(colors)))]
    while len(boxes) < num_colors:
        max_range = -1.0
        max_idx = -1
        max_dim = -1
        for i, box in enumerate(boxes):
            if not box: continue
            min_c = [min(colors[j][d] for j in box) for d in range(3)]
            max_c = [max(colors[j][d] for j in box) for d in range(3)]
            rng = [max_c[d] - min_c[d] for d in range(3)]
            dim = rng.index(max(rng))
            if rng[dim] > max_range:
                max_range = rng[dim]
                max_idx = i
                max_dim = dim
        box = boxes.pop(max_idx)
        box_sorted = sorted(box, key=lambda idx: colors[idx][max_dim])
        mid = len(box_sorted) // 2
        boxes.append(box_sorted[:mid])
        boxes.append(box_sorted[mid:])
    codebook = []
    for box in boxes:
        if box:
            avg = [sum(colors[j][d] for j in box) / len(box) for d in range(3)]
            codebook.append(avg)
        else:
            codebook.append([0.0, 0.0, 0.0])
    while len(codebook) < num_colors:
        codebook.append([0.0, 0.0, 0.0])
    return codebook

def compress_plenoctree(tree_leaves, num_sh_basis=9, num_colors=65536):
    """
    Compress a PlenOctree using SH-9, median-cut quantization, and DEFLATE.
    Args:
        tree_leaves: list of dicts with keys 'sigma' (float), 'sh_coeffs' (list of 9 RGB vectors)
        num_sh_basis: number of SH basis functions (default 9 for SH-9)
        num_colors: quantization levels (default 2^16)
    Returns:
        compressed_data: bytes, DEFLATE-compressed tree stream
        codebooks: dict mapping basis_idx -> list of [r, g, b] (num_colors, 3)
    """
    # Step 1: Collect coefficients and build codebooks
    all_coeffs = {i: [] for i in range(num_sh_basis)}
    for leaf in tree_leaves:
        for b in range(num_sh_basis):
            all_coeffs[b].append(leaf['sh_coeffs'][b])

    codebooks = {}
    for b in range(num_sh_basis):
        codebooks[b] = median_cut_quantize(all_coeffs[b], num_colors)

    # Step 2: Replace coefficients with pointers
    leaf_data = []
    for leaf in tree_leaves:
        sigma = leaf['sigma']
        pointers = []
        for b in range(num_sh_basis):
            rgb = leaf['sh_coeffs'][b]
            # Find index in codebook[b] closest to rgb [G]
            best_idx = -1
            best_dist = float('inf')
            for idx, cb_rgb in enumerate(codebooks[b]):
                dist = math.sqrt(sum((rgb[d] - cb_rgb[d])**2 for d in range(3)))
                if dist < best_dist:
                    best_dist = dist
                    best_idx = idx
            pointers.append(best_idx)
        leaf_data.append((sigma, pointers))

    # Step 3: Serialize tree structure
    # Layout per leaf: sigma (float32, 4B) + 9 pointers (int16, 2B each) = 22 bytes
    # The source specifies compressing the entire tree, including pointers [S1, unofficial].
    byte_stream = bytearray()
    for sigma, pointers in leaf_data:
        byte_stream.extend(struct.pack('<f', sigma))
        for ptr in pointers:
            byte_stream.extend(struct.pack('<h', ptr))

    # Step 4: Apply DEFLATE compression
    compressed_data = zlib.compress(bytes(byte_stream))
    return compressed_data, codebooks
```

**Edge Cases and Pitfalls:**
* **Precision:** Using `float16` for codebooks reduces memory but may introduce quantization error in the codebook values themselves [S1, unofficial].
* **Pointer Size:** `int16` limits the codebook size to 65,536 entries. If more colors are needed, this method cannot scale without changing the pointer type [S1, unofficial].
* **Decompression:** The entire tree must be fully decompressed before rendering in the web renderer [S1, unofficial].

## Operating characteristics
* **Compression Ratio:** Reduces file size by 20-30 times compared to the uncompressed PlenOctree [S1, unofficial].
* **Performance (SH-9):** File Size: 1.00 GB, Rendering FPS: 262, PSNR: 31.44, SSIM: 0.951, LPIPS: 0.065 [S1, unofficial].
* **Comparison with Higher SH Degrees:** SH-16: 1.93 GB, 168 FPS, PSNR 31.57, SSIM 0.952, LPIPS 0.063; SH-25: 2.68 GB, 128 FPS, PSNR 31.56, SSIM 0.951, LPIPS 0.063 [S1, unofficial].
* **Strengths:** Significant file size reduction enabling in-browser rendering; simple compression pipeline (median-cut + DEFLATE); high rendering FPS due to reduced data transfer and simpler SH evaluation [S1, unofficial].
* **Limitations:** Lossy compression (quantization); requires full decompression before rendering; SH-9 may lose high-frequency detail compared to SH-16/25, though the PSNR drop is minimal (0.13 dB) [S1, unofficial].
* **When to use:** When in-browser rendering of NeRFs is required and bandwidth/storage is a constraint [S1, unofficial].
* **When not to use:** When maximum visual fidelity is required and bandwidth is not a constraint (use SH-16/25) [S1, unofficial].

## Background
PlenOctree Compression was introduced in the paper "PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs)" by Yu et al. [S1, unofficial]. The method was developed to address the large file sizes of PlenOctrees, which were originally too large for efficient in-browser viewing [S1, unofficial]. The approach combines established techniques (median-cut quantization, DEFLATE compression) with specific adaptations for PlenOctree data structures (SH coefficients, tree pointers) [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
