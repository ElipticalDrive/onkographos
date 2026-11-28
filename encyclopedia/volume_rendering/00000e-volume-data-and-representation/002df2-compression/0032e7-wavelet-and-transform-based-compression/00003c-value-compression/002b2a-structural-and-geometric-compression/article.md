# Structural and Geometric Compression

Also called: Data Structure and Compression

## Overview
Structural and Geometric Compression encompasses methods that encode volume-derived geometric primitives—such as isosurfaces, depth layers, and feature maps—by exploiting their spatial connectivity and structural properties rather than treating the volume as a uniform scalar grid [S1, S2, S3]. Unlike [Value compression](6-value-compression.md), which applies linear transforms to scalar fields, these techniques store explicit geometric relationships (e.g., mesh connectivity, surface links, or PCA-based feature descriptors) to enable efficient on-the-fly rendering or selective feature reconstruction [S1, S2, S3].

## How it works
**Inputs and Outputs:** 
- *BLIC:* Input is an isosurface mesh (connectivity and geometry) extracted from a scalar volume grid; output is a compressed bitstream containing connectivity, geometry, and optional normals [S3].
- *Selective Feature-Based:* Input is a compressed voxel map with descriptors and a set of model descriptors representing features of interest; output is a set of expanded voxel blocks for regions matching the models [S2].
- *Depth/Link Structure:* Input is voxelized particle data; output is parallel arrays of depth, surface links, and normals per voxel column [S1].

**Data Structures:**
- *Depth Values:* Stored in 32-bit floating-point format [S1].
- *Surface Links:* Links to the four direct neighbors of each depth value are stored in one byte each, packed into a 32-bit word [S1].
- *Normals:* Compressed to a 32-bit word using octahedron normal encoding with two 16-bit normalized integers [S1].
- *Layout:* Normals are stored in parallel to surface depth values to allow easy access during rendering, reusing surface link storage logic for on-the-fly connectivity computation [S1].

**Numbered Steps:**
1. **Connectivity & Geometry Encoding (BLIC):** Encode mesh topology using a context-based arithmetic coder closely related to the JBIG binary image compression standard [S3]. Encode a substantial proportion of geometric information to a fraction of a bit per marching cubes vertex using the same mechanism [S3].
2. **Refined Geometry Encoding (BLIC):** Efficiently encode remaining optional geometric information, which specifies the location of each marching cubes vertex more precisely along its supporting intersecting grid edge, in scan-order using the same mechanism [S3].
3. **Normal Handling (BLIC):** Optionally compute vertex normals as normalized gradient vectors $\mathbf{n} = \frac{\nabla f}{\|\nabla f\|}$, quantize them, and include them in the bitstream after entropy encoding, or omit them for the decoder to compute in a post-processing smoothing step [S3].
4. **Model Creation (Selective):** Create descriptor-based models for geometries of interest by encoding TSDFs over a range of transformations (e.g., 15 different offsets for horizontal planes, generating one 64-element vector each) [S2].
5. **Similarity Matching (Selective):** Compare each compressed map descriptor $\mathbf{v}_b$ to the model descriptors $\mathbf{v}_m$ by computing the squared norm of their differences: $d = \|\mathbf{v}_b - \mathbf{v}_m\|^2 = \sum_i (v_{b,i} - v_{m,i})^2$ [S2].
6. **Selective Decompression (Selective):** Decompress only those blocks whose similarity distance $d$ is beneath a specified threshold $\tau$ [S2].
7. **Dimensionality Matching (Selective):** Leverage PCA's ordered linear subspace to match feature vectors of different dimensionality (e.g., match a 32-dimensional feature descriptor against the first half of a 64-dimensional feature descriptor) [S2].

**Parameters:**
- *Normal Computation Mode:* Encoder-computed vs. decoder-computed vertex normals, determined by trade-offs between bitstream size and in-core vs. out-of-core decoder memory constraints [S3].
- *Similarity Threshold $\tau$:* Controls the fidelity of selective reconstruction [S2].
- *Descriptor Dimensionality:* Typically 64 elements per vector, generated from TSDF sampling [S2].

**Edge Cases and Pitfalls:**
- Descriptors are not invariant to affine transformations; if they were, decompression would not reproduce the field in the correct location or orientation [S2].
- Not given in the sources for precision limits, boundary conditions, or degenerate input handling.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blic_encode(mesh, encoder_computes_normals):
    # Step 1: Encode connectivity with context-based arithmetic coder (JBIG-related)
    connectivity_bits = arithmetic_encode(mesh.topology) # Standard arithmetic coding [G]
    # Step 2: Encode primary geometry (fraction of bits per vertex)
    primary_geom_bits = arithmetic_encode(mesh.vertices, precision='low') # Standard arithmetic coding [G]
    # Step 3: Encode refined geometry in scan-order
    refined_geom_bits = arithmetic_encode(mesh.vertices, precision='high', order='scan') # Standard arithmetic coding [G]
    # Step 4: Handle normals
    if encoder_computes_normals:
        # Compute normalized gradient vectors for each vertex [G]
        normals = [normalize(cross(edge1, edge2)) for edge1, edge2 in mesh.faces]
        normal_bits = arithmetic_encode(normals, quantize=True) # Standard arithmetic coding [G]
    else:
        normal_bits = []
    return connectivity_bits + primary_geom_bits + refined_geom_bits + normal_bits

def selective_feature_expand(compressed_map, model_descriptors, threshold):
    expanded_blocks = []
    for block in compressed_map:
        # Compare descriptor to models via squared norm of differences
        distances = [sum((b - m)**2 for b, m in zip(block.descriptor, model)) for model in model_descriptors]
        min_dist = min(distances)
        # Decompress only if similarity distance is beneath threshold
        if min_dist < threshold:
            expanded_blocks.append(decompress_block(block)) # Standard decompression [G]
    return expanded_blocks
```

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
- **BLIC:** Achieves high compression rates with extreme simplicity [S3]. The choice of normal computation affects memory usage: encoder-computed normals increase bitstream size but reduce decoder complexity, while decoder-computed normals save bits but require post-processing [S3].
- **Selective Feature-Based Expansion:** Allows for adaptive resolution rendering where only relevant features are decompressed [S2]. The use of PCA allows for flexible dimensionality matching, supporting multi-level compression schemes [S2].
- **Depth/Link Structure:** Provides efficient on-the-fly rendering by storing depth and normals in parallel, allowing ray casting to access surface properties directly from the compressed structure [S1].
- **Fractal Compression:** Offers high compression ratios but suffers from extreme encoding complexity and iterative reconstruction costs [S1]. Not given in the sources for measured performance or scaling.
- **Parallel Edgebreaker:** Reduces transmission and processing time significantly through parallelism, at the cost of a ~10% increase in file size compared to sequential methods [S1]. Not given in the sources for measured performance or scaling.

## Background
- **BLIC:** Introduced in "BLIC: Bi-Level Isosurface Compression" [S3].
- **Selective Feature-Based Map Expansion:** Described in "Compressed Voxel-Based Mapping Using Unsupervised Learning" [S2].
- **Depth/Link Compression:** Described in "Memory-Efficient On-The-Fly Voxelization of Particle Data" [S1].
- **Fractal Compression:** Described as a lossy method exploiting self-similarity [S1]. Not given in the sources for origin or specific paper references.
- **Parallel Edgebreaker:** Described as a parallel variant of the Edgebreaker algorithm [S1]. Not given in the sources for origin or specific paper references.

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
[S2] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
[S3] BLIC: Bi-Level Isosurface Compression, https://doi.org/10.5555/602099.602170, fetched 2026-10-26
