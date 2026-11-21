# G-PCC PlenOctree Compression

Also called: Compressed PlenOctrees (C-PO), GeS-TM Compression, PlenOctree Attribute Compression via G-PCC, PlenOctree Compression with GeS-TM, RAHT/Predlift PlenOctree Compression

## Overview
G-PCC PlenOctree Compression is a method for compressing PlenOctree models by treating the voxelized tree nodes as a point cloud (PC) and applying Geometry-based Point Cloud Coding (G-PCC) standards [S1, S2]. It encodes the geometry (voxel positions) losslessly to preserve the octree structure and encodes the attributes (opacity $\sigma$ and spherical harmonic coefficients $k$) lossily using Region Adaptive Hierarchical Transform (RAHT) or Predlift transforms [S1, S2]. It is distinguished from sibling methods like [Octree File I/O Format](5032-octree-file-i-o-format.md) by its specific use of G-PCC attribute coders on PlenOctree parameters, and from [Tolerance-based Voxel Fusion](5866-tolerance-based-voxel-fusion.md) by its focus on bitstream compression rather than structural simplification.

## How it works
**Inputs:** A PlenOctree of depth $M$ containing $N$ active voxels. Each voxel $n$ has grid coordinates $x_{oct}(n) = [X_n, Y_n, Z_n]$ and attributes $s(n) = [\sigma_n, k^0_{n,0}, k^{-1}_{n,1}, k^0_{n,1}, \dots, k^\ell_{n,\ell}]$, where $\sigma_n$ is opacity and $k^m_{n,\ell} \in \mathbb{R}^3$ are spherical harmonic (SH) coefficients [S1, S2].

**Outputs:** A G-PCC bitstream containing lossless geometry and lossy attributes, which can be decoded to reconstruct the PlenOctree [S1, S2].

**Data Structures:**
The PlenOctree is mapped to a voxelized point cloud $P$ of $N$ points.
*   **Geometry Coordinates ($x_{pc}$):** For each point $n$, the 3D coordinates are calculated as:
    $$x_{pc} = x_{oct} \times 2^M - 0.5$$
    This maps the integer grid indices to a continuous coordinate space suitable for point cloud coding [S1, S2].
*   **Attributes ($s$):** A vector containing $\sigma$ and all SH coefficients for the point.

**Steps:**

1.  **Geometry Encoding (Lossless):**
    Encode the geometry coordinates $x_{pc}$ losslessly. This is typically done using octree representation or direct geometry quantization via octrees, ensuring the original PlenOctree structure is preserved [S1, S2]. The geometry of the point cloud is not altered [S1].

2.  **Attribute Scaling:**
    Scale each attribute channel individually to a fixed integer range $[0, T]$ to prepare for quantization [S1].
    *   For opacity $\sigma$: $T = 2047$. This value is chosen based on the range of $\alpha$ and an expected grid size of 512 [S1].
    *   For SH DC coefficients ($\ell=0, m=0$): $T = 1023$ [S1].
    *   For SH AC coefficients ($\ell \neq 0$): $T = 511$ [S1].
    *   **Sign Handling:** For SH coefficients, scaling is performed on the absolute values $|k|$. The signs of the coefficients are extracted and sent as side information in the bitstream [S1].

3.  **Attribute Transform and Quantization:**
    Encode the scaled attributes $s$ using a G-PCC attribute coder. Two primary transforms are used [S1, S2]:
    *   **RAHT (Region Adaptive Hierarchical Transform):** Performs a hierarchical transform on the point cloud attributes based on the octree structure. It is generally preferred for PlenOctrees due to the low correlation between neighboring voxel attributes [S1].
    *   **Predlift (Prediction and Lifting):** A predictive coding scheme. It typically yields larger file sizes than RAHT for this specific data type [S1].
    *   **Quantization:** Apply quantization using a Quantization Parameter (QP). Standard G-PCC test conditions use QP values from the set $\{22, 28, 34, 40, 46, 51\}$, where lower QP indicates higher quality and lower bitrate [S2].
    *   **Note on KLT:** Applying Karhunen-Loève Transform (KLT) over the SH coefficients (e.g., "RAHT+KLT") is not recommended as the SH decomposition already decorrelates the channels sufficiently, and KLT does not improve performance [S1].

4.  **Entropy Coding:**
    Entropy encode the lossless geometry bitstream and the quantized/transformed attribute bitstreams to produce the final compressed output [S2].

5.  **Decoding and Reconstruction:**
    *   Decode the geometry to recover $x_{pc}$ losslessly.
    *   Decode the attributes, apply inverse quantization, and unscale the values using the inverse of the scaling factors $T$ [S1].
    *   Reconstruct the SH coefficients by applying the stored signs back to the decoded absolute values [S1].
    *   Map the decoded point cloud $\hat{P}$ back to the PlenOctree structure using the lossless geometry $x_{pc}$ and the lossy attributes $\hat{\sigma}$ and $\hat{k}$ [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_plenoctree(plen_octree, depth_M, QP):
    """
    Compress a PlenOctree using G-PCC standards.
    plen_octree: List of nodes, each node n has:
        - x_oct: [X, Y, Z] integer grid coordinates
        - sigma: float opacity
        - sh_coeffs: list of vectors k_l_m in R^3
    depth_M: Integer depth of the octree
    QP: Integer Quantization Parameter (e.g., 22, 28, ..., 51)
    """
    N = len(plen_octree)
    geometry_coords = []
    attributes_raw = []
    sh_signs = [] # Store signs for SH coefficients

    # Step 1 & 2: Map to Point Cloud and Scale Attributes
    for n in range(N):
        node = plen_octree[n]
        
        # Calculate PC coordinates
        # x_pc = x_oct * 2^M - 0.5
        x_pc = [coord * (2 ** depth_M) - 0.5 for coord in node.x_oct]
        geometry_coords.append(x_pc)
        
        # Prepare attributes: sigma and SHs
        sigma = node.sigma
        shs = node.sh_coeffs
        
        # Scale Sigma
        # T_sigma = 2047
        sigma_scaled = int(sigma * 2047)
        
        # Scale SHs
        shs_scaled = []
        current_sh_signs = []
        for k_vec in shs:
            # Handle signs separately
            signs = [1 if val >= 0 else -1 for val in k_vec]
            current_sh_signs.append(signs)
            
            # Scale absolute values
            # T_dc = 1023 for l=0, T_ac = 511 for l!=0
            # Assuming shs are ordered by l, m. 
            # Simplified logic: check if l=0 (first element usually DC)
            # In practice, index mapping depends on SH ordering convention.
            # Here we assume the first coefficient is DC (l=0, m=0)
            if k_vec is the DC coefficient:
                t_val = 1023
            else:
                t_val = 511
            
            abs_vals = [abs(val) for val in k_vec]
            scaled_vals = [int(val * t_val) for val in abs_vals]
            shs_scaled.append(scaled_vals)
            
        attributes_raw.append((sigma_scaled, shs_scaled))
        sh_signs.append(current_sh_signs)

    # Step 3: Geometry Encoding (Lossless)
    # Encode geometry using octree representation
    geometry_bitstream = encode_geometry_lossless(geometry_coords)

    # Step 4: Attribute Encoding (Lossy via RAHT/Predlift)
    # Flatten attributes for point cloud coder
    # Note: G-PCC coders expect a list of attribute vectors per point
    attribute_vectors = []
    for sigma_s, shs_s in attributes_raw:
        # Combine sigma and all SH coefficients into one vector
        # Vector = [sigma_s, shs_s[0][0], shs_s[0][1], shs_s[0][2], ...]
        vec = [sigma_s] + [val for sublist in shs_s for val in sublist]
        attribute_vectors.append(vec)

    # Apply RAHT transform and quantization
    # The specific RAHT implementation depends on the G-PCC reference software (e.g., GeS-TM)
    # Here we represent the transform and quantization abstractly
    transformed_attrs = raht_transform(attribute_vectors)
    quantized_attrs = quantize(transformed_attrs, QP)
    
    # Encode signs as side information
    signs_bitstream = encode_signs(sh_signs)
    
    # Entropy encode attributes
    attribute_bitstream = entropy_encode(quantized_attrs)

    # Combine bitstreams
    final_bitstream = combine(geometry_bitstream, attribute_bitstream, signs_bitstream)
    return final_bitstream

def decompress_plenoctree(bitstream, depth_M):
    """
    Decompress a G-PCC PlenOctree bitstream.
    """
    # Step 5: Decode Geometry
    geometry_coords = decode_geometry_lossless(bitstream.geometry)
    
    # Step 5: Decode Attributes
    quantized_attrs = entropy_decode(bitstream.attributes)
    transformed_attrs = dequantize(quantized_attrs, bitstream.QP)
    attribute_vectors = raht_inverse_transform(transformed_attrs)
    
    signs = decode_signs(bitstream.signs)
    
    # Reconstruct PlenOctree
    plen_octree = []
    for n in range(len(geometry_coords)):
        x_pc = geometry_coords[n]
        
        # Recover grid coordinates
        # x_oct = (x_pc + 0.5) / 2^M
        # Since x_pc was derived from integer x_oct, this should return integers
        x_oct = [(coord + 0.5) / (2 ** depth_M) for coord in x_pc]
        x_oct = [int(round(coord)) for coord in x_oct]
        
        # Unscale attributes
        sigma_scaled = attribute_vectors[n][0]
        sigma = sigma_scaled / 2047.0
        
        shs_scaled = attribute_vectors[n][1:]
        shs = []
        # Reconstruct SH coefficients
        # Assuming shs_scaled are flattened lists of R^3 vectors
        # We need to know the SH order to split them back into vectors
        # This depends on the specific SH packing used in attribute_vectors
        # Placeholder for SH unpacking logic
        shs = unpack_sh_coefficients(shs_scaled, signs[n])
        
        plen_octree.append(Node(x_oct=x_oct, sigma=sigma, sh_coeffs=shs))
        
    return plen_octree
```

**Parameters:**
*   **QP (Quantization Parameter):** Values $\{22, 28, 34, 40, 46, 51\}$ control the trade-off between bitrate and quality [S2].
*   **Scaling Constants ($T$):** $\sigma \to 2047$, SH DC $\to 1023$, SH AC $\to 511$ [S1].

**Edge Cases and Pitfalls:**
*   **KLT Application:** Applying KLT to SH coefficients does not improve performance and should be avoided [S1].
*   **Precision:** The scaling step introduces quantization error. The choice of $T$ is critical; incorrect $T$ values lead to clipping or poor dynamic range utilization [S1].
*   **Structure Preservation:** Geometry must be encoded losslessly. Any loss in geometry breaks the PlenOctree structure, making it incompatible with the fast renderer [S1].

## Subtopics

### Occlusion-Aware Point Cloud Compression
A variant that reduces bitrate by encoding only the colors (attributes) of non-occluded voxels for each viewpoint [S3]. It generates a sub-point cloud $B_k$ for each viewpoint $k$ containing only points $p$ where the occlusion mask $O_k(p)$ is true [S3]. The geometry of the original point cloud is encoded losslessly once and transmitted; the geometry for each sub-PC is not transmitted [S3]. Attributes are encoded lossily using RAHT over KLT-transformed coefficients in YUV space [S3]. The decoder reconstructs the occlusion mask $O$ from transmitted camera poses $d$ and intrinsics $K$ [S3]. This scheme provides a 1.9 dB advantage in BD-PSNR compared to sending all colors (LLC) and achieves lower bitrate for similar or better quality [S3]. QP values $\{22, 28, 34, 40, 46, 51\}$ are used [S3].

## Operating characteristics
*   **Compression Ratio:** The lossy compression method provides a size reduction of almost 50 times compared to the uncompressed solution [S1]. The lossless approach reduces the model's average size by around 8 times [S1].
*   **Quality:** The RAHT-based approach is approximately three times smaller than the compressed "0.4 GB" curve of the reference PlenOctree while being more than 0.6 dB better in PSNR [S1]. The RAHT-based method outperforms the Predlift method, which is expected due to the low correlation of the neighbors' attributes [S1].
*   **Performance:** The compressed model maintains the high rendering speed of the original PlenOctree, being faster by 1.5+ times compared to some SOTA methods [S1].
*   **Strengths:** Preserves the fast rendering speed of PlenOctrees; lossless geometry ensures structural integrity; RAHT is effective for the low-correlation attributes of PlenOctrees.
*   **Limitations:** Lossy compression introduces slight performance degradation in rendering quality (PSNR) compared to uncompressed [S1]; KLT does not help [S1].

## Background
The method was proposed to address the large file size disadvantages of PlenOctrees while maintaining their rendering quality and speed [S1]. It targets the compression stage of the PlenOctree pipeline [S1]. The work builds on the original PlenOctree paper [6] and utilizes G-PCC standards (specifically GeS-TM) from MPEG [S1, S2]. It was evaluated on the NeRF-Synthetic 360 (Blender) dataset [S1, S3].

## Sources
[S1] Compression of PlenOctree Model Attributes Enabling Fast Communication and Rendering of Neural Radiance Fields, https://hal.science/hal-04195520v1, fetched 2026-10-26
[S2] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
[S3] A Comparative Assessment Of Implicit and Explicit Plenoptic Scene Representations, https://inria.hal.science/hal-04709909v1, fetched 2026-10-26
