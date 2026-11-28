# Bitstream-oriented TexF Compression
Also called: TexF Compression using 3D Attribute Codecs

## Overview
Bitstream-oriented TexF Compression compresses the texture attributes of a Surface-aligned Texture Field (TexF) by leveraging established 3D attribute codecs [S1, unofficial]. It treats the TexF attributes as a point cloud attribute dataset and compresses them without separately transmitting voxel coordinates, relying on the assumption that the underlying mesh geometry (and thus the voxel set $Q$) is known or reconstructed independently [S1, unofficial].

## How it works
The method compresses only the texture attributes $t(q)$ associated with each voxel $q \in Q$ [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A TexF defined as $T = \{(q, t(q)) \mid q \in Q\}$, where $Q$ is the set of voxel positions derived from the reconstructed mesh geometry, and $t(q)$ are the texture attributes at each voxel [S1, unofficial].
*   **Output:** A compressed bitstream containing the encoded attributes [S1, unofficial].

**Data Structures:**
*   **Voxel Set $Q$:** A set of 3D coordinates derived from the reconstructed mesh geometry [S1, unofficial]. The coordinates are not explicitly stored in the attribute bitstream [S1, unofficial].
*   **Attributes $t(q)$:** A vector of values for each voxel [S1, unofficial].

**Steps:**
1.  **Geometry Reconstruction/Alignment:** The encoder and decoder derive the same voxel set $Q$ from the reconstructed mesh geometry [S1, unofficial].
2.  **Attribute Extraction:** Extract the texture attributes $t(q)$ for all $q \in Q$ [S1, unofficial]. If multiple surface samples map to the same voxel, their attributes are averaged to produce a single $t(q)$ [S1, unofficial].
3.  **3D Attribute Encoding:** Pass the set of attributes $\{t(q)\}$ to an established 3D attribute codec backend (e.g., Unicorn or G-PCC) [S1, unofficial]. The codec performs an analysis transform and entropy coding [S1, unofficial]. The voxel coordinates $Q$ are not separately transmitted in the attribute bitstream [S1, unofficial].
4.  **Decoding:** The decoder uses the same geometry-derived voxel set $Q$ and the established backend codec to perform entropy decoding and inverse analysis transform, reconstructing the attributes $\hat{t}(q)$ [S1, unofficial].

**Parameters:**
*   **Backend Codec:** Established 3D attribute codecs such as Unicorn or G-PCC are used [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Geometry Mismatch:** If the decoder's reconstructed geometry differs from the encoder's, the voxel set $Q$ will not match, leading to incorrect attribute assignment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_texf(mesh_geometry, texf_attributes):
    # 1. Derive voxel set Q from reconstructed mesh geometry [G]
    Q = derive_voxel_set_from_mesh(mesh_geometry)
    
    # 2. Extract attributes for each q in Q [S1, unofficial]
    attributes = [texf_attributes[q] for q in Q]
    
    # 3. Compress attributes using established 3D attribute codec (e.g., Unicorn/G-PCC) [S1, unofficial]
    # Performs analysis transform and entropy coding [S1, unofficial]
    # Voxel coordinates Q are not separately transmitted [S1, unofficial]
    bitstream = codec_backend_encode(attributes) # [S1, unofficial]
    
    return bitstream

def decode_texf(bitstream, mesh_geometry):
    # 1. Derive voxel set Q from reconstructed mesh geometry [G]
    Q = derive_voxel_set_from_mesh(mesh_geometry)
    
    # 2. Reconstruct attributes using established 3D attribute codec backend [S1, unofficial]
    # Performs entropy decoding and inverse analysis transform [S1, unofficial]
    attributes = codec_backend_decode(bitstream) # [S1, unofficial]
    
    # 3. Reconstruct TexF T = {(q, t(q)) | q in Q} [S1, unofficial]
    texf = {q: attributes[i] for i, q in enumerate(Q)}
    
    return texf
```

## Subtopics

### [GPU-resident Neural Compression for TexF](10087-gpu-resident-neural-compression-for-texf.md)
GPU-resident neural compression for TexF is a method for memory-efficient random access during rendering, using GPU-resident neural compression. It adapts multiresolution hash encoding to TexF. It replaces explicit voxel attributes with a directly queryable compressed field to avoid storing per-voxel lookup entries. Unlike bitstream-oriented TexF compression which uses established 3D attribute codecs for disk storage, GPU-resident neural compression targets memory-efficient random access during rendering.

## Operating characteristics
*   **Strengths:** Leverages established 3D attribute codecs for compression [S1, unofficial].
*   **Limitations:** Requires identical geometry reconstruction on encoder and decoder sides [S1, unofficial]. Does not compress the geometry itself [S1, unofficial].

## Background
Bitstream-oriented TexF Compression is described in "Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields" [S1, unofficial]. It applies established 3D attribute codecs to compress TexF attributes without transmitting voxel coordinates [S1, unofficial].

## Sources
[S1] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
