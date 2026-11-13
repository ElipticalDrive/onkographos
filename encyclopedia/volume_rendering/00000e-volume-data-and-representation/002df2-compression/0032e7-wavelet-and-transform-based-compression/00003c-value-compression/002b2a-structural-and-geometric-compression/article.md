# Structural and Geometric Compression

Also called: Data Structure and Compression

## Overview
Structural and Geometric Compression refers to methods that compress volume data by exploiting the spatial structure, connectivity, and geometric properties of the underlying scene representation, rather than treating the volume as a uniform grid of scalar values. Unlike [Value compression](6-value-compression.md) which focuses on transform-based redundancy reduction of scalar fields, this approach targets the explicit geometric structures (such as isosurfaces, depth layers, or feature-based maps) extracted from or inherent in the volume. It sets itself apart by storing connectivity, normals, and depth relationships directly, often enabling efficient on-the-fly rendering or selective reconstruction of specific features.

## How it works
The implementation of Structural and Geometric Compression varies significantly depending on the target representation (isosurfaces, depth layers, or feature maps). The following sections detail the specific algorithms found in the source literature.

### General Data Structures and Storage
For depth-layer based representations (e.g., particle data), the data structure stores multiple layers of depth, surface links, and normals per voxel column [S1].
*   **Depth Values:** Stored in 32-bit floating-point format [S1].
*   **Surface Links:** Links to the four direct neighbors of each depth value are stored in one byte each, totaling a 32-bit word [S1].
*   **Normals:** Compressed to a 32-bit word using octahedron normal encoding with two 16-bit normalized integers [S1].
*   **Parallel Storage:** Normals are stored in parallel to surface depth values to allow easy access during rendering [S1].

### BLIC (Bi-Level Isosurface Compression)
BLIC is a simple algorithm to compress isosurface data extracted by isosurface algorithms from scalar functions defined on volume grids [S3].

**Inputs:** Isosurface mesh (connectivity and geometry) derived from a scalar volume grid.
**Outputs:** Compressed bitstream containing connectivity, geometry, and optional normals.

**Steps:**
1.  **Connectivity Encoding:** Encode mesh connectivity using a context-based arithmetic coder closely related to the JBIG binary image compression standard [S3].
2.  **Primary Geometry Encoding:** Encode a substantial proportion of geometric information (e.g., vertex positions along grid edges) to a fraction of a bit per marching cubes vertex using the same arithmetic coding mechanism [S3].
3.  **Refined Geometry Encoding:** Efficiently encode remaining optional geometric information, which specifies the location of each marching cubes vertex more precisely along its supporting intersecting grid edge, in scan-order using the same mechanism [S3].
4.  **Normal Handling (Optional):**
    *   *Encoder-computed:* Compute vertex normals as normalized gradient vectors, quantize, and include them in the bitstream after entropy encoding [S3].
    *   *Decoder-computed:* Omit normals from the bitstream; the decoder computes them in a post-processing smoothing step [S3].
    *   *Decision:* The choice is determined by trade-offs associated with an in-core vs. out-of-core decoder structure [S3].

**Pseudocode for BLIC Encoding:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blic_encode(mesh):
    # Step 1: Encode connectivity
    connectivity_bits = arithmetic_encode_connectivity(mesh.topology)
    
    # Step 2: Encode primary geometry (fraction of bits per vertex)
    primary_geom_bits = arithmetic_encode_geometry(mesh.vertices, precision='low')
    
    # Step 3: Encode refined geometry in scan-order
    refined_geom_bits = arithmetic_encode_scan_order(mesh.vertices, precision='high')
    
    # Step 4: Handle normals based on decoder constraint
    if decoder_supports_encoder_normals:
        normals = compute_normalized_gradients(mesh)
        normal_bits = arithmetic_encode_quantized(normals)
    else:
        normal_bits = []
        
    return concatenate(connectivity_bits, primary_geom_bits, refined_geom_bits, normal_bits)
```

### Selective Feature-Based Map Expansion
This method uses descriptor-based models to selectively decompress parts of a compressed voxel map (e.g., TSDF maps) based on geometric similarity [S2].

**Inputs:** Compressed voxel map with descriptors; a set of model descriptors representing features of interest (e.g., floors).
**Outputs:** Expanded voxel blocks for regions matching the models.

**Steps:**
1.  **Model Creation:** Create descriptor-based models for geometries of interest by encoding TSDFs over a range of transformations. For example, encode TSDFs of horizontal planes at 15 different offsets, generating one 64-element vector for each [S2].
2.  **Similarity Comparison:** Compare each compressed map descriptor to the model descriptors by computing the squared norm of their differences [S2].
3.  **Selective Decompression:** Decompress only those blocks whose similarity distance is beneath a specified threshold [S2].
4.  **Dimensionality Matching:** Leverage PCA's ordered linear subspace to match feature vectors of different dimensionality. For instance, match a 32-dimensional feature descriptor against the first half of a 64-dimensional feature descriptor [S2].

**Pitfalls:**
*   Descriptors are not invariant to affine transformations. If they were, decompression would not reproduce the field in the correct location or orientation [S2].
*   Models must be created by sampling TSDFs over the range of transformations to which the model should be invariant [S2].

## Subtopics

### [Fractal Compression](5719-fractal-compression.md)
Fractal compression is a lossy volume compression method that exploits self-similarity within the dataset by mapping each "range block" to a larger, most similar "domain block" from the same volume, storing only the affine transformation required to convert the domain block into the range block. Unlike vector quantization, which stores a reference to a codeword, fractal compression stores the geometric relationship between two different-sized blocks, allowing for high compression ratios at the cost of extreme encoding complexity and iterative reconstruction during decoding.

### [Parallel Edgebreaker](7789-parallel-edgebreaker.md)
Parallel Edgebreaker is a parallel compression algorithm for isosurfaces where each part of the mesh is encoded independently from the others, based on the Edgebreaker algorithm. It solves the problem of efficiently transmitting or storing large isosurface meshes in distributed environments by leveraging parallel processing. Unlike sequential compression, which processes the mesh as a single unit, this method partitions the mesh to allow concurrent encoding, trading a slight increase in file size (~10%) for significant reductions in transmission and processing time.

### Selective Feature-Based Map Expansion
A variant of descriptor-based compression for voxel maps that uses PCA-generated feature vectors to identify and selectively decompress regions of interest (e.g., floors, walls) by comparing compressed block descriptors against a small set of model descriptors [S2]. It allows for handling multiple levels of compression by matching feature vectors of different dimensionality (e.g., 32-dim against 64-dim) due to the ordered nature of the PCA subspace [S2].

### BLIC (Bi-Level Isosurface Compression)
A simple algorithm to compress isosurface data extracted from volume grids, encoding connectivity and geometry to a fraction of a bit per vertex using JBIG-related arithmetic coding, with optional support for encoder-computed or decoder-computed vertex normals [S3].

## Operating characteristics
*   **BLIC:** Achieves high compression rates with extreme simplicity [S3]. The choice of normal computation affects memory usage: encoder-computed normals increase bitstream size but reduce decoder complexity, while decoder-computed normals save bits but require post-processing [S3].
*   **Selective Feature-Based Expansion:** Allows for adaptive resolution rendering where only relevant features are decompressed [S2]. The use of PCA allows for flexible dimensionality matching, supporting multi-level compression schemes [S2].
*   **Depth/Link Structure:** Provides efficient on-the-fly rendering by storing depth and normals in parallel, allowing ray casting to access surface properties directly from the compressed structure [S1].
*   **Fractal Compression:** Offers high compression ratios but suffers from extreme encoding complexity and iterative reconstruction costs [S1].
*   **Parallel Edgebreaker:** Reduces transmission and processing time significantly through parallelism, at the cost of a ~10% increase in file size compared to sequential methods [S1].

## Background
*   **BLIC:** Introduced in "BLIC: Bi-Level Isosurface Compression" [S3].
*   **Selective Feature-Based Map Expansion:** Described in "Compressed Voxel-Based Mapping Using Unsupervised Learning" [S2].
*   **Depth/Link Compression:** Described in "Memory-Efficient On-The-Fly Voxelization of Particle Data" [S1].
*   **Fractal Compression:** Referenced in [S1] as a method exploiting self-similarity.
*   **Parallel Edgebreaker:** Referenced in [S1] as a parallel variant of the Edgebreaker algorithm.

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
[S2] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
[S3] BLIC: Bi-Level Isosurface Compression, https://doi.org/10.5555/602099.602170, fetched 2026-10-26
