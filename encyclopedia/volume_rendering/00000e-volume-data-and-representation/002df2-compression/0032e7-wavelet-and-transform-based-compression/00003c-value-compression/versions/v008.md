# Value compression


## Overview
Value compression reduces the storage and bandwidth requirements of volumetric data by transforming the scalar field into a set of coefficients, quantizing them, and applying entropy coding [S1][S6]. It solves the problem of high memory footprint in large-scale volumetric datasets by exploiting spatial redundancy in the frequency or multi-resolution domain [S1][S6]. It is distinguished from its siblings, such as [Structural and Geometric Compression](4959-structural-and-geometric-compression.md) (which targets explicit geometry) and [Predictive and Differential Coding](4393-predictive-and-differential-coding.md) (which operates locally in space/time), by relying on global or block-wise linear transforms (e.g., wavelets, DCT) to decorrelate data before compression [S6].

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of coefficients $C$ using a linear transform, followed by encoding and optional decoding for rendering [S1][S6]. 
1. **Transform:** The volume is decomposed using a separable 1D filter bank applied along each axis [S5, unofficial]. For a 3D volume, this results in subbands representing different frequency components [S5, unofficial]. In wavelet transforms, these are typically approximation (low-pass) and detail (high-pass) coefficients [S6]. In Laplacian pyramids, they are detail residuals between levels of a Gaussian pyramid [see [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md)].
2. **Hierarchy:** The decomposition is recursive [S6]. The approximation subband from level $j$ is further decomposed at level $j+1$, creating a tree-like structure of coefficients [S6]. This allows for multi-resolution access, where coarse data is available at the root and fine details are deeper in the tree [see [Value compression](6-value-compression.md)].
3. **Quantization and Entropy Encoding:** Coefficients are quantized (lossy) or kept precise (lossless) [S1]. Entropy coding (e.g., Huffman, Arithmetic, Run-Length) is applied to the quantized or raw coefficients to produce the compressed bitstream [S1].
4. **Reconstruction:** To render, the necessary coefficients are decoded and the inverse transform is applied [S6]. This can be done globally (full reconstruction) or locally (block-wise or point-wise) to support adaptive resolution rendering [S6].

## Subtopics

### [Rational Bézier Deblocking Filter](832-rational-b-zier-deblocking-filter.md)
A post-processing technique to reduce blocking artifacts at block boundaries in compressed volume data by projecting voxel values onto rational Bézier curves. It is distinct because it is a general, data-dependent filter requiring no a-priori knowledge of the compression technique.

### [Differential Data Stream (DDS) File Format](1754-differential-data-stream-dds-file-format.md)
A lossless compression method encoding differential values using run-length encoding and adaptive bit allocation per run. It supports two versions to handle different data sizes via an interleave parameter.

### [Attribute Quantization](1912-attribute-quantization.md)
A lossy technique reducing the precision of voxel attributes (colors, normals) to lower memory footprint. It specializes quantization based on data type (e.g., CIELAB for colors, Octahedron Normal Vectors for normals) to leverage human perception limits.

### [Zerobit Encoding Scheme](1918-zerobit-encoding-scheme.md)
A compression method for wavelet-transformed data that improves compression ratio and random access speed by exploiting spatial coherence in coefficient clusters using variable-length significance maps based on "zerobits".

### [Random Access of Compressed Data (Westermann's Method)](2269-random-access-of-compressed-data-westermann-s-method.md)
A data layout technique supporting random access to wavelet coefficients in compressed linear arrays by storing entries as linear arrays per level with leading masks and offset arrays, overcoming variable-length decoding overhead.

### [Predictive and Differential Coding](4393-predictive-and-differential-coding.md)
Reduces data redundancy by estimating a voxel's value from neighbors, computing the residual, and encoding only that difference. It operates locally or temporally rather than globally in the transform domain.

### [Standard Image Compression Formats](4394-standard-image-compression-formats.md)
A family of lossy codecs (e.g., BC7) adapted for volume rendering by treating data as dense 2D surfaces or 3D blocks. It relies on fixed-rate, block-based spatial locality and hardware-accelerated decompression.

### [Structural and Geometric Compression](4959-structural-and-geometric-compression.md)
Compresses volume data by exploiting spatial structure, connectivity, and geometric properties (e.g., isosurfaces, depth layers) rather than treating the volume as a uniform grid of scalar values.

### [Texture Format Compression](5623-texture-format-compression.md)
Uses standardized, block-based compression algorithms (e.g., BC4, BC5) originally designed for 2D image data to compress volumetric data into fixed-size bitstreams.

### [Time-Varying Volume Compression Strategies](5714-time-varying-volume-compression-strategies.md)
Addresses the bandwidth bottleneck in real-time rendering of 4D datasets by exploiting temporal coherence across frames to compress the fourth dimension.

### [Quantization](5935-quantization.md)
Reduces the precision of volumetric data coefficients by mapping continuous values to a discrete set of representatives. It includes methods that group data (Vector Quantization) or exploit structural symmetries.

### [Burrows-Wheeler Transform](6541-burrows-wheeler-transform.md)
A reversible permutation scheme that re-sorts data to organize it into a form highly compressible for subsequent entropy coding, distinguished by rearranging symbol positions rather than changing values.

### [Transform-based Block Coding](6882-transform-based-block-coding.md)
Compresses data by partitioning it into fixed-size blocks, applying a linear transform, and entropy coding coefficients. It differs from global methods by enabling localized random access.

### [Closed-Loop DPCM Decompression](11737-closed-loop-dpcm-decompression.md)
Reconstructs data using a feedback loop where the decoder's reconstructed values serve as the prediction for subsequent samples, preventing unbounded error accumulation.

### [Wavelet Transform and Decomposition](13030-wavelet-transform-and-decomposition.md)
Compresses scalar fields by decomposing them into coefficients using linear transforms (wavelets, Laplacian pyramids), differing from spatial prediction or fractal methods.

### [TT4D GPU Decoder and Renderer](14080-tt4d-gpu-decoder-and-renderer.md)
A GPU-based, on-the-fly decoding and rendering scheme for 4D time-varying data using Tensor-Train decomposition, enabling interactive visualization without full reconstruction.

### Progressive Encoding of Volumetric Data
**Definition:** A scalability mechanism for wavelet-compressed volumes where decoding can be stopped at any point in the bitstream to retrieve a lower-resolution approximation [S2].
**Steps:**
1. Apply the whole set of quantizers to the whole set of subband images before passing to the next subband [S2].
2. Scan subbands following decomposition level: all subbands at level $l$ are scanned before passing to level $l+1$ [S2].
3. During step $l$, the quantizer is applied to each image of each subband [S2].
**Comparison:** In G-PROG mode, the compression ratio is maximized, but 3-D encoding/2-D decoding functionalities are not enabled [S2].

### Zarr-based Archive Encoding
**Definition:** A storage format for fitted splats using semantic encoding and width-aware compression policies [S3, unofficial].
**Steps:**
1. Fitted splats are written to Zarr9 archives with semantic encoding (8 semantic types) [S3, unofficial].
2. Apply width-aware Blosc-zstd compression policy: byte shuffle for multi-byte integer codes; no shuffle for single-byte codes and floating-point arrays [S3, unofficial].
**Parameters:** Server requirement: Static HTTP host allowing cross-origin reads and range requests for zipped form [S3, unofficial].
**Layout:** Archive format: Directory of chunk files, or a single uncompressed `.luxar.zarr.zip` file where each chunk is a member fetched by HTTP range request without unpacking [S3, unofficial].

### [Significance Maps and Bit-wise Encoding](5740-significance-maps-and-bit-wise-encoding.md)
**Definition:** Approaches that fully avoid entropy encoding and RLE by using significance maps of wavelet coefficients and bit-wise encodings.

### [Block Truncation Coding](6187-block-truncation-coding.md)
**Definition:** An adaptation of BTC supporting hardware accelerated random access, specifically texture compression using invalid combinations.

### Volumetric Coding Scheme
**Definition:** A coding scheme separating volumetric data into model $m$ (binary volume: 1=existing, 0=transparent) and texture $T$ (luminance Y, chrominance U,V) to allow lossless model coding and scalable bitstreams [S4].
**Steps:**
1. Use luminance Y to identify the model structure. Input Y to depth-plane reduction; subsample chrominance U,V before depth-plane reduction [S4].
2. Losslessly and independently code the model $m$. Code texture $T$ lossily [S4].
3. Reorganize Y and U,V into appropriate forms (via chroma subsampling and depth-plane reduction) before applying lossy coding [S4].
4. Organize the data stream in this order: 1) Half resolution volume data ($m_h, T_h$), 2) Additional data for full resolution ($m_f, T_f$), 3) Remaining data for double resolution ($m_d, T_d$) [S4].
5. Separately code camera parameters using Huffman coding as side information [S4].
6. Independently code the background (same geometry as volume) from the volumetric data [S4].

### General Lossy Compression Flow
**Definition:** The decompression process is the reverse of the 3 stages mentioned above (Data Conversion, Quantization, Entropy Encoding) [S5, unofficial].

### 4×4×4 Block Reconstruction
**Definition:** A block-wise decoding optimization for wavelet-compressed volumes that processes voxels in 4×4×4 blocks to reuse R-slice values across adjacent blocks, reducing decoding overhead for spatially coherent access patterns [S6].
**Steps:**
1. Partition the volume into 4×4×4 blocks, each structurally containing 3 I-slices followed by 1 R-slice [S6].
2. Begin decoding a block by computing the 4×4 voxels of its R-slice [S6].
3. Compute the 4×4 voxels for each of the 3 I-slices in the current block by reusing the retained R-slice values from the block above [S6].
**Parameters:** Block dimensions: 4 × 4 × 4 voxels. Block slice composition: 3 I-slices followed by 1 R-slice [S6].
**Pitfalls:** Purely random access patterns negate the block reuse optimization; the method assumes regular or spatially coherent access [S6].
**Numbers:** Random voxel access slowdown: compressed data is 3 to 5 times slower than uncompressed data on average. Whole-volume reconstruction overhead: compressed volumes take roughly 30-40 seconds longer than original volumes [S6].
**Comparison:** Random access is 3-5x slower for compressed data, but this is acceptable compared to disk/network latency and typical non-random access patterns [S6].

### µ-law Feature Encoding
**Definition:** A non-linear transformation applied to raw sensor values before computing pixel-wise MSE loss, compressing dynamic range to emphasize shadow detail [S7].
**Formula:** Raw 14-bit sensor values $x$ are transformed as $f(x) = \log(1 + \mu \cdot x/w) / \log(1 + \mu)$, with $\mu = 5000$ and $w = 2^{14} - 1 = 16,383$ (the white level of 14-bit data) [S7].

### [Quantization and Entropy Encoding/Decoding Pipeline](11661-quantization-and-entropy-encoding-decoding-pipeline.md)
**Definition:** The compression procedure involving quantization followed by entropy encoding (range-coder) for compression, and entropy decoding for decompression.
**Steps:** Convert compressed data to non-negative values to enable compression into int8 format, then revert to original range during decompression [G].
**Parameters:** $q$: The quantization parameter. Data is multiplied by $q$ to expand the data range, enhancing quantization precision. Adjusting $q$ trades off reconstruction quality and model storage size [G].

### Entropy Coding
**Steps:**
1. Arrange quantized DCT coefficients in a '3D zigzag' order [S8].
2. Apply Run-Length Encoding (RLE) to group similar frequencies (zeros) together [S8].
3. Insert length coding zeros [S8].
4. Apply Huffman coding to the remainder [S8].
5. Code DC coefficients separately from AC coefficients [S8].
6. Encode DC coefficients using Differential Pulse Code Modulation (DPCM): encode the difference between the current DC coefficient and the previous DC coefficient (except for the first one) [S8].
7. Encode AC coefficients using RLC (Run-Length Coding) method [S8].
8. Use Huffman coding to further compress the DPCM-coded DC coefficients and RLE-coded AC coefficients [S8].

## Background
Value compression has been a cornerstone of volume rendering research since the introduction of wavelet transforms to the domain. Muraki [21] was among the first to introduce wavelet transform methods for volume data compression, noting their ability to maintain data in both spatial and frequency domains, facilitating local reconstruction [S6]. Komma et al. [KFDB07] provided extensive benchmarks of lossless compression encodings for volume data, covering entropy encodings, RLE, and wavelet transformations [S1]. The field has since expanded to include hardware-accelerated block truncation coding (BTC) variants like VTC, ETC2, and ASTC, which offer simple decompression and fast random access, albeit with lower signal-to-noise ratios compared to full transform methods [S1]. Recent developments include Tensor-Train decompositions for 4D data and neural representation compression using quantization and entropy coding pipelines [G].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
[S3] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S4] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
[S5] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S6] Wavelet Based 3D Compression for Very Large Volume Data Supporting Fast Random Access, https://www.brics.dk/RS/99/34/index.html, fetched 2026-11-04
[S7] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S8] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
