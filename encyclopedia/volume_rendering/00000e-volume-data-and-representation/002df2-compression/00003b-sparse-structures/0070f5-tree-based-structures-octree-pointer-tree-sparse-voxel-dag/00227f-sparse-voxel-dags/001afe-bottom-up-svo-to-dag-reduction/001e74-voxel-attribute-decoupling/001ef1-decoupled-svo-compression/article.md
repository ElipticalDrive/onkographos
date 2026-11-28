# Decoupled SVO Compression

Also called: Palette Compression for SVOs, SVO Compression Scheme

## Overview
Decoupled SVO Compression is a memory-efficient compression scheme for Sparse Voxel Octrees (SVOs) that separates geometric topology from voxel attributes (such as color, normals, or reflectance) [S1]. It solves the problem of high memory usage in colored voxel scenes by allowing independent compression of geometry and attributes, achieving significantly lower memory footprints than standard SVOs or naive colored DAGs [S1]. It sets itself apart from sibling methods like Moxel DAGs by using a single running sum for index calculation rather than two, reducing storage overhead for scenes with vast numbers of empty leaves [S1].

## How it works
The method operates in three phases: decoupling, topology compression, and attribute compression.

**Inputs and Outputs**
*   **Input:** A Sparse Voxel Octree (SVO) or Directed Acyclic Graph (DAG) representing a 3D scene, where each leaf node contains geometric data and voxel attributes (e.g., 12-bit or 24-bit colors, 32-bit normals).
*   **Output:** A compressed data structure consisting of a compacted geometry component and a separate, compressed attribute array.

**Data Structures**
The geometry is stored in a compacted format using the following fields:
1.  **Childmasks:** 8-bit masks indicating the presence of children, replacing the original 24-bit padding [S1].
2.  **Pointers:** Compressed child pointers using implicit first-child offsets and per-level byte-precise offset compression [S1].
3.  **Offsets:** Implicitly stored first-child offsets to reduce pointer size [S1].
4.  **Attribute Array:** A dense array storing quantized and palette-compressed attributes, indexed by offsets stored in the geometry structure [S1].

**Steps**

1.  **Decouple Topology and Attributes**
    Separate the SVO/DAG topology (child pointers, childmasks) from the voxel attributes (colors, normals) [S1]. The geometry structure retains only the structural information required for traversal, while attributes are moved to a separate dense array [S1].

2.  **Compress Topology**
    Apply the following compression techniques to the geometry structure [S1]:
    *   **Implicit First Child Offset:** Store the offset to the first child implicitly, reducing pointer storage requirements [S1].
    *   **Per-Level Byte-Precise Offset Compression:** Compress offsets on a per-level basis with byte precision [S1].
    *   **Pointer Compression:** Compress child pointers using the per-level byte-precise method [S1].
    *   **8-bit Childmasks:** Replace the original 24-bit childmasks (which included padding) with 8-bit masks [S1].
    *   **Pointer Entropy Encoding:** Apply entropy encoding to the compressed pointers to further reduce size [S1].

3.  **Compress Attributes**
    Compress the separated attribute array using a combination of quantization and lossless palette compression [S1].
    *   **Quantization:** Quantize attribute values (e.g., colors) to reduce precision if necessary, though the method supports high-precision inputs [S1].
    *   **Palette Compression:** Exploit spatial coherence by replacing global material indices with local palette indices [S1]. This allows the bit-width per voxel to adapt to the local entropy of the attribute data [S1].
    *   **Lossless Encoding:** Use a lossless palette approach to ensure high quality, particularly for normals and high-bit-depth colors [S1].

4.  **Store Final Structure**
    Store the compressed geometry and attribute arrays separately. During traversal, reconstruct attribute indices using the offsets stored in the geometry structure [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decoupled_svo_compression(svo_tree, attributes):
    """
    Compresses an SVO by decoupling geometry and attributes.
    
    Args:
        svo_tree: The original Sparse Voxel Octree/DAG structure.
        attributes: List of voxel attributes corresponding to leaf nodes.
        
    Returns:
        compressed_geometry: Compressed topology data.
        compressed_attributes: Compressed attribute array.
    """
    
    # Step 1: Decouple
    # Extract topology (pointers, masks) and attributes separately
    topology = svo_tree.extract_topology()
    attr_list = svo_tree.extract_attributes()
    
    # Step 2: Compress Topology
    # Apply 8-bit childmasks
    topology.childmasks = compress_childmasks(topology.childmasks, bits=8)
    
    # Apply implicit first child offset
    topology = apply_implicit_first_child_offset(topology)
    
    # Apply per-level byte-precise offset compression
    topology.offsets = compress_offsets_per_level(topology.offsets)
    
    # Apply pointer compression
    topology.pointers = compress_pointers(topology.pointers)
    
    # Apply pointer entropy encoding
    topology.pointers = entropy_encode(topology.pointers)
    
    compressed_geometry = topology
    
    # Step 3: Compress Attributes
    # Quantize attributes if needed (e.g., 24-bit color to 12-bit for comparison, 
    # but method supports 24-bit color and 32-bit normal losslessly)
    # Note: Quantization is part of the palette compression pipeline
    quantized_attrs = quantize_attributes(attr_list)
    
    # Apply lossless palette compression exploiting spatial coherence
    compressed_attributes = palette_compress(quantized_attrs)
    
    return compressed_geometry, compressed_attributes

def compress_childmasks(masks, bits):
    """Reduces childmask bit-width from 24 to specified bits (typically 8)."""
    # Implementation detail: Truncate or pack bits as per source [S1]
    return masks[:bits] # Placeholder for specific bit-packing logic

def compress_offsets_per_level(offsets):
    """Compresses offsets on a per-level basis with byte precision."""
    # Implementation detail: Group offsets by level and apply delta/byte compression
    pass

def compress_pointers(pointers):
    """Compresses child pointers using per-level byte-precise method."""
    # Implementation detail: Apply delta encoding and byte packing
    pass

def entropy_encode(data):
    """Applies entropy encoding (e.g., Huffman, Arithmetic) to data."""
    # Standard entropy encoding implementation
    pass

def quantize_attributes(attrs):
    """Quantizes attribute values."""
    # Implementation detail: Map high-precision values to lower precision bins
    pass

def palette_compress(attrs):
    """
    Compresses attributes using a lossless palette scheme.
    Replaces global indices with local palette indices.
    """
    # Implementation detail: Build local palettes based on spatial coherence
    # and map attributes to palette indices
    pass
```

**Parameters**
*   **Color Precision:** 12-bit colors are used for the main comparison in the source [S1]. However, the method supports 24-bit colors and 32-bit ONV normals for higher quality [S1].
*   **Attribute Precision:** 24-bit colors and 32-bit ONV normals are used for palette compression to ensure better quality than ESVO's lossy schemes [S1].

**Edge Cases and Pitfalls**
*   **ESVO Compatibility:** ESVO implementation failed to load the Arena scene, so no data is reported for it in the comparison [S1].
*   **Contour Data:** ESVOs require contour data to avoid cutting off attributes during traversal; without contours, quality degrades significantly [S1]. The decoupled method avoids this issue by storing attributes separately and reconstructing them during traversal [S1].

## Subtopics

### [Palette Compression](1873-palette-compression.md)
Palette compression is a variable-length compression scheme for the attribute array in decoupled Sparse Voxel Octrees (SVOs) that reduces memory footprint by exploiting spatial coherence and attribute redundancy. It differs from fixed-size array storage by replacing global material indices with local palette indices, allowing the bit-width per voxel to adapt to the local entropy of the attribute data.

### [Decoupled DAG with Palette Compression](1893-decoupled-dag-with-palette-compression.md)
Decoupled DAG with Palette Compression is a memory-efficient compression scheme for sparse voxel scenes that separates geometric topology from voxel attributes (such as color, normals, or reflectance). It encodes the scene structure using a Directed Acyclic Graph (DAG) to maximize geometric redundancy, while storing attributes in a dense array that is compressed via a palette-based scheme. This approach solves the problem of high memory usage in colored voxel scenes, where traditional DAGs fail to merge nodes because attribute differences prevent subtree equality. It sets itself apart from sibling methods like Moxel DAGs by using a single running sum for index calculation rather than two, reducing storage overhead for scenes with vast numbers of empty leaves.

## Operating characteristics
*   **Time and Memory Complexity:** The method achieves significant memory reduction. For the Citadel scene, the memory footprint is 2957 MB total (348 MB geometry + 2609 MB attributes), which is 0.65 bytes/voxel, representing a 22% compression rate compared to standard SVO [S1]. For the City scene, it is 5889 MB total (186 MB geometry + 5703 MB attributes), 0.59 bytes/voxel, 21% compression rate [S1]. For San Miguel, it is 3415 MB total (316 MB geometry + 3099 MB attributes), 0.24 bytes/voxel, 8.7% compression rate [S1]. For Arena, it is 1029 MB total (591 MB geometry + 438 MB attributes), 0.33 bytes/voxel, 11% compression rate [S1].
*   **Measured Performance:** The method outperforms SVOs (2.93 B/v Citadel), PSVOs (1.79 B/v), ESVOs (2.29 B/v), and CDAGs (1.22 B/v) for the Citadel scene [S1].
*   **Scaling:** The method is used for real-time rendering of colored SVOs with up to 17 levels (128K^3 resolution) on GPU [S1].
*   **Strengths and Limitations:** The method achieves lossless compression rates of 38.6% for dense datasets like a Christmas tree, approaching state-of-the-art methods for dense datasets (29.4%) [S1]. When applying filtering to remove scanning noise, rates below 10% are achieved [S1]. The method reduces memory usage from 4.49 times (Citadel) up to 11.5 times (San Miguel) compared to standard SVO implementations [S1].
*   **When to Use:** Suitable for real-time rendering applications including hard shadows, reflections (secondary rays), color bleeding (single-bounce global illumination), and dense dataset compression [S1].

## Background
The method was presented in a paper by Timothy Kol et al. [S1]. It relies on the decoupling of geometry from additional voxel data, enabling separate compression methods for topology and voxel attributes [S1]. The approach introduces little overhead and outperforms state-of-the-art SVO compression methods [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
