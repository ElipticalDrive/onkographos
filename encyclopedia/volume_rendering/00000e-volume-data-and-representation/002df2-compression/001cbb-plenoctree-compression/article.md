# PlenOctree Compression

Also called: Median-cut Quantization and DEFLATE Compression, PlenOctree Compression Strategy, SH-9 PlenOctree Compression

## Overview
PlenOctree Compression is a file-size reduction strategy designed to make PlenOctrees feasible for in-browser rendering by minimizing occupied voxels and applying direct compression to the tree structure [S1, unofficial]. It distinguishes itself from general value compression [6-value-compression.md] by specifically targeting the spherical harmonic (SH) coefficients of PlenOctrees using median-cut quantization and DEFLATE encoding, and from sparse structures [5-sparse-structures.md] by compressing the entire tree including pointers rather than just the data values.

## How it works
**Inputs:** An uncompressed PlenOctree containing SH coefficients (RGB vectors) and density ($\sigma$) values.
**Outputs:** A compressed binary file containing the quantized codebooks, leaf pointers, and DEFLATE-compressed tree structure.

**Data Structures:**
1.  **Codebook:** A lookup table of size $2^{16} \times 3$ storing RGB colors. Data type: `float16` (IEEE 754 half-precision). One codebook is stored per SH basis function.
2.  **Leaf Pointers:** For each leaf node in the PlenOctree, a pointer to an index in the corresponding SH basis function's codebook. Data type: `int16` (16-bit signed integer).
3.  **Tree Structure:** The full PlenOctree hierarchy, including internal node pointers and leaf data (pointers and $\sigma$), compressed as a byte stream.

**Parameters:**
*   **SH Degree:** Limited to degree 2 (SH-9). This reduces the number of coefficients per voxel compared to SH-16 or SH-25 [S1, unofficial].
*   **Bounding Box:** A looser bounding box is used during the initial PlenOctree construction to reduce the total number of occupied voxels [S1, unofficial].
*   **Quantization Levels:** $2^{16}$ (65,536) colors.

**Steps:**
1.  **Quantize SH Coefficients:**
    *   For each voxel in the PlenOctree, retain the density value $\sigma$ as a standard floating-point value (no quantization specified for $\sigma$).
    *   For each SH basis function $m$, extract the RGB coefficient vector $\mathbf{k}_m \in \mathbb{R}^3$.
    *   Apply the **median-cut algorithm** to quantize all $\mathbf{k}_m$ vectors across the entire tree into $2^{16}$ representative colors.
    *   Generate a codebook of size $2^{16} \times 3$ for each SH basis function.
    *   Replace each voxel's $\mathbf{k}_m$ with an index (pointer) into the corresponding codebook. The index is stored as an `int16`.
2.  **Compress Tree:**
    *   Serialize the entire PlenOctree structure, including internal nodes, leaf pointers (to codebooks), and $\sigma$ values.
    *   Apply the **DEFLATE algorithm** (as implemented in ZLIB) to the serialized byte stream [S1, unofficial].
3.  **Store:**
    *   Save the DEFLATE-compressed tree stream.
    *   Save the codebooks (one per SH basis function) as `float16` arrays.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import zlib
import numpy as np

def compress_plenoctree(tree):
    """
    Compress a PlenOctree using SH-9, median-cut quantization, and DEFLATE.
    
    Args:
        tree: A PlenOctree object containing nodes with:
              - sigma: density value (float)
              - sh_coeffs: list of RGB vectors for each SH basis function (degree 2 -> 9 coeffs)
    
    Returns:
        compressed_data: bytes, the DEFLATE-compressed tree stream
        codebooks: dict, mapping SH_basis_index -> numpy array of shape (65536, 3), dtype float16
    """
    num_sh_basis = 9  # SH-9
    num_colors = 65536 # 2^16
    
    # Initialize codebooks and pointers storage
    # codebooks[basis_idx] will store the final 2^16 x 3 codebook
    codebooks = {i: None for i in range(num_sh_basis)}
    
    # Step 1: Collect all RGB coefficients for each SH basis function to build codebooks
    # We need to run median-cut on the global set of coefficients for each basis
    all_coeffs = {i: [] for i in range(num_sh_basis)}
    
    # Traverse tree to collect coefficients
    for node in tree.all_leaves():
        for basis_idx in range(num_sh_basis):
            rgb = node.sh_coeffs[basis_idx] # Shape (3,)
            all_coeffs[basis_idx].append(rgb)
            
    # Build codebooks using median-cut for each basis
    for basis_idx in range(num_sh_basis):
        coeffs_array = np.array(all_coeffs[basis_idx], dtype=np.float32)
        # Median-cut algorithm implementation [G]
        # 1. Start with bounding box of all colors
        # 2. Repeatedly split the largest dimension of the largest box until 2^16 boxes
        # 3. Center of each box is a codebook entry
        codebook = median_cut_quantize(coeffs_array, num_colors)
        codebooks[basis_idx] = codebook.astype(np.float16)
        
    # Step 2: Replace coefficients with pointers
    # Create a new tree structure with pointers instead of raw RGB
    compressed_tree_nodes = []
    
    for node in tree.all_leaves():
        new_node = Node()
        new_node.sigma = node.sigma # Keep sigma as is
        
        # Quantize each SH basis coefficient to nearest codebook entry
        new_node.pointers = []
        for basis_idx in range(num_sh_basis):
            rgb = node.sh_coeffs[basis_idx]
            # Find index in codebook[basis_idx] closest to rgb [G]
            # Euclidean distance in RGB space
            codebook = codebooks[basis_idx]
            dists = np.linalg.norm(codebook - rgb, axis=1)
            idx = np.argmin(dists)
            new_node.pointers.append(idx)
            
        compressed_tree_nodes.append(new_node)
        
    # Step 3: Serialize and Compress
    # Serialize the tree structure (pointers and sigma) into a byte stream
    # The exact serialization format is not specified, but must include pointers and sigma [G]
    # Assuming a simple binary layout: [sigma (float32), ptr0 (int16), ..., ptr8 (int16)] per leaf
    # Plus internal node structure. For this example, we assume a flat list of leaves for simplicity
    # or a standard tree serialization. The source says "entire tree, including pointers".
    
    # Placeholder for tree serialization logic [G]
    # In practice, this would serialize the node hierarchy.
    # Here we serialize just the leaf data for demonstration of the compression step.
    leaf_data = []
    for node in compressed_tree_nodes:
        leaf_data.append(node.sigma)
        leaf_data.extend(node.pointers)
        
    # Convert to bytes
    # sigma: float32 (4 bytes), pointers: int16 (2 bytes each)
    # Total per leaf: 4 + 9*2 = 22 bytes
    byte_stream = np.array(leaf_data, dtype=np.float32).tobytes() # Simplified: only sigma for demo
    # Note: The pointers must be included in the actual byte stream.
    # Correct serialization would pack sigma and pointers together.
    
    # Re-do serialization properly
    packed = bytearray()
    for node in compressed_tree_nodes:
        packed.extend(node.sigma.to_bytes(4, byteorder='little', signed=False)) # Assuming float32
        for ptr in node.pointers:
            packed.extend(ptr.to_bytes(2, byteorder='little', signed=True))
            
    # Step 4: Apply DEFLATE
    compressed_data = zlib.compress(bytes(packed))
    
    return compressed_data, codebooks

def median_cut_quantize(colors, num_colors):
    """
    Standard median-cut algorithm for color quantization.
    """
    # Implementation of median-cut [G]
    # 1. Sort colors by each dimension
    # 2. Create initial box containing all colors
    # 3. Loop until num_colors boxes:
    #    a. Find box with largest range (max - min) in any dimension
    #    b. Split that box along the dimension with the largest range
    #    c. Split point is the median of the colors in that dimension
    # 4. Compute centroid of each box
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using `float16` for codebooks reduces memory but may introduce quantization error in the codebook values themselves. The source specifies `float16` [S1, unofficial].
*   **Pointer Size:** `int16` limits the codebook size to 65,536 entries. If more colors are needed, this method cannot scale without changing the pointer type.
*   **Decompression:** The entire tree must be fully decompressed before rendering in the web renderer [S1, unofficial].

## Operating characteristics
*   **Compression Ratio:** Reduces file size by 20-30 times compared to the uncompressed PlenOctree [S1, unofficial].
*   **Performance (SH-9):**
    *   File Size: 1.00 GB
    *   Rendering FPS: 262
    *   PSNR: 31.44
    *   SSIM: 0.951
    *   LPIPS: 0.065
    [S1, unofficial]
*   **Comparison with Higher SH Degrees:**
    *   SH-16: 1.93 GB, 168 FPS, PSNR 31.57, SSIM 0.952, LPIPS 0.063
    *   SH-25: 2.68 GB, 128 FPS, PSNR 31.56, SSIM 0.951, LPIPS 0.063
    [S1, unofficial]
*   **Strengths:** Significant file size reduction enabling in-browser rendering; simple compression pipeline (median-cut + DEFLATE); high rendering FPS due to reduced data transfer and simpler SH evaluation.
*   **Limitations:** Lossy compression (quantization); requires full decompression before rendering; SH-9 may lose high-frequency detail compared to SH-16/25, though the PSNR drop is minimal (0.13 dB).
*   **When to use:** When in-browser rendering of NeRFs is required and bandwidth/storage is a constraint.
*   **When not to use:** When maximum visual fidelity is required and bandwidth is not a constraint (use SH-16/25).

## Background
PlenOctree Compression was introduced in the paper "PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs)" by Yu et al. [S1, unofficial]. The method was developed to address the large file sizes of PlenOctrees, which were originally too large for efficient in-browser viewing. The approach combines established techniques (median-cut quantization, DEFLATE compression) with specific adaptations for PlenOctree data structures (SH coefficients, tree pointers).

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
