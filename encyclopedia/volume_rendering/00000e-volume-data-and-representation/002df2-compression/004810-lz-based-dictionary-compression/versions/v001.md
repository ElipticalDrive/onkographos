# LZ-based Dictionary Compression

Also called: Lempel-Ziv family, LZ compression, Dictionary-based compression

## Overview
LZ-based dictionary compression is a lossless data compression technique that replaces repeated occurrences of data with references to a single copy of that data existing earlier in the input stream [S1][S2, unofficial]. It solves the problem of redundant data storage in volumetric datasets by exploiting spatial and structural repetition without requiring complex mathematical transforms [S2, unofficial]. What sets it apart from its siblings, such as [Sparse structures](5-sparse-structures.md) or [Wavelet and Transform-based Compression](6879-wavelet-and-transform-based-compression.md), is its reliance on string-matching algorithms rather than statistical modeling or frequency-domain transformation [S1][S2, unofficial]. Unlike [Arithmetic Encoding](2127-arithmetic-encoding.md), which is an entropy coding stage, LZ provides the symbol generation stage by identifying literal bytes and back-references [S1].

## Shared principles
All compression methods in this family share the following mechanism:

1.  **Redundancy Exploitation:** The core principle is to identify and eliminate redundancy in the volumetric data. This redundancy can be spatial (empty space, homogeneous regions), statistical (correlated values between neighbors), or structural (repetitive patterns in time-varying data) [G].
2.  **Transformation/Encoding:** Data is transformed into a domain where redundancy is more apparent or compact. This includes spatial subdivision (sparse structures), frequency domain transformation (wavelets), or dictionary-based representation (sparse coding, vector quantization) [G].
3.  **Quantization (Lossy):** For lossy compression, continuous or high-precision values are mapped to a discrete set of representatives. This step is critical for achieving high compression ratios but introduces reconstruction error, which must be managed via metrics like PSNR or perceptual quality [G].
4.  **Entropy Coding:** The transformed or quantized data is encoded into a bitstream using entropy coding techniques (e.g., Huffman, Arithmetic, rANS) to approach the Shannon entropy limit of the source [G].
5.  **Decompression/Reconstruction:** Rendering requires decoding the bitstream and reconstructing the volume data, either fully or partially (e.g., via LOD or adaptive loading), to provide input to the ray-marching or ray-casting pipeline [G].

## Subtopics

### LZO1X-999 Compression
- **Definition:** A variant of the Lempel-Ziv-Oberhumer (LZO) real-time compression library that uses the same Lempel-Ziv dictionary coder as gzip but is optimized for fast decompression rather than maximum compression ratio [S1].
- **Parameters:** The algorithm variant is `LZO1X-999`. It is the slowest compressor in the LZO family (up to 8 times slower than the default variant) but does not influence decompression speed [S1].
- **Comparison:** Unlike gzip or bzip2, which are optimized for maximum compression ratio but are often too slow to decompress faster than reading uncompressed data, LZO is optimized for decompression speed [S1]. In the context of time-varying volume data, the time for reading and uncompressing a data block must be less than the time needed to read the uncompressed block [S1].

### LZV Algorithm
- **Comparison:** The LZV algorithm and its improvements perform better in volume space and on differential data compared to other dictionary methods [S2, unofficial].

### Deflate Compression
- **Definition:** Compresses input bytes using Java's `Deflater` class with a specified compression level [S3, unofficial].

## Background
LZ-based compression originates from the work of Abraham Lempel and Jacob Ziv in the early 1970s (LZ77 and LZ78) [G]. In the context of volume rendering, these algorithms were adopted to handle the large data sizes of volumetric datasets, particularly for time-varying data where spatial coherence is high [S1]. The LZO library was introduced by Markus Oberhumer to address the bottleneck where decompression speed lagged behind data transfer speed [S1]. While standard LZ variants like Deflate are common in general-purpose compression, specialized variants like LZO1X-999 are chosen in volume rendering pipelines to ensure that the decompression overhead does not exceed the I/O latency of reading raw data [S1]. The LZV algorithm is noted in literature as being particularly effective for medical MRI data and differential data in volume spaces [S2, unofficial].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S3] https://github.com/rubensrudio/SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
