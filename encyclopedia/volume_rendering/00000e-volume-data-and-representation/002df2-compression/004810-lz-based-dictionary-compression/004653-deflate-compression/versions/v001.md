# Deflate Compression
Also called: DEFLATE, Deutsch 1996, TraceBlockCodec.deflate

## Overview
Deflate Compression is a general-purpose lossless compression algorithm that compresses input bytes using a combination of LZ77 sliding window matching and Huffman coding [S1, unofficial][S2]. In the context of volume rendering and scientific data, it serves as a baseline lossless compressor for floating-point arrays [S2]. It is distinguished from specialized scientific compressors (like Szip) by its generic nature, though it can be significantly enhanced for scientific data when paired with a byte-shuffling pre-conditioner [S2].

## How it works
**Inputs and outputs:**
- Input: A stream of bytes (e.g., raw binary representation of a volume dataset).
- Output: A compressed byte stream.

**Data structures:**
The algorithm internally maintains a sliding window buffer (typically 32 KB) for back-references and dynamic Huffman trees for symbol encoding [S2]. The exact bit-packing layout of the DEFLATE stream (blocks, final bit, Huffman codes) is defined by the DEFLATE specification [S1, unofficial].

**Parameters:**
- `level`: An integer from 1 (fastest) to 9 (best compression) [S2].
- `shuffling`: A boolean flag indicating whether the byte-shuffling pre-conditioner is applied before compression [S2].

**Steps:**
1. If `shuffling` is true, apply the Shuffling pre-conditioner to the input byte stream [S2].
2. Initialize the DEFLATE compressor with the specified `level` [S1, unofficial].
3. Feed the input bytes into the compressor [S1, unofficial].
4. Retrieve the compressed output stream [S1, unofficial].

**Shuffling Pre-conditioner Steps (if enabled):**
1. Iterate through the input floating-point array [S2].
2. Extract the mantissa and exponent components from each element [S2].
3. Reorder the byte stream by placing the first byte of every element into the first chunk, the second byte of every element into the second chunk, and so on [S2].
4. Pass the reordered stream to the DEFLATE compressor [S2].

**Formulas:**
- Compression throughput: $\theta_c = s(f)/t_c$ [S2]
- Decompression throughput: $\theta_d = s(f)/t_d$ [S2]
- Relative compressed file size: $\Sigma$ [S2]
- Compression ratio: $r$ [S2]
Where $s(f)$ is the original storage space (e.g., 1 GB for a $512^3$ double array), $t_c$ is CPU time for compression, and $t_d$ is CPU time for decompression [S2].

**Edge cases and pitfalls:**
- Without shuffling, DEFLATE achieves very low compression ratios for floating-point data (e.g., $\Sigma = 96.3\%$ on HIT dataset) [S2].
- High compression levels (9) significantly reduce throughput compared to level 1 [S2].

## Subtopics

### Shuffling pre-conditioner for DEFLATE
A pre-processing step that decomposes floating-point array elements into mantissa and exponent components and reorders the byte stream to group identical bytes into contiguous chunks [S2]. It exploits the fact that neighboring points in scientific datasets have close values, producing continuous subsequences of identical entries that DEFLATE compresses well [S2]. Functionally similar to quantization [S2]. On the HIT dataset (x-velocity, $512^3$ double array, 1 GB original), shuffling + DEFLATE level 9 yields $\Sigma = 81.2\%$, $r = 1.231$, $\theta_c = 12.7$ MB/s, $\theta_d = 290.2$ MB/s [S2].

## Operating characteristics
**Time and memory complexity:**
Not given in the sources.

**Measured performance:**
On the HIT dataset (1 GB, $512^3$ double array) [S2]:
- DEFLATE level 1: $\Sigma = 96.3\%$, $r = 1.038$, $\theta_c = 23.2$ MB/s, $\theta_d = 144.1$ MB/s [S2].
- DEFLATE level 9: $\Sigma = 95.6\%$, $r = 1.046$, $\theta_c = 16.9$ MB/s, $\theta_d = 153.0$ MB/s [S2].
- Shuffling + DEFLATE level 9: $\Sigma = 81.2\%$, $r = 1.231$, $\theta_c = 12.7$ MB/s, $\theta_d = 290.2$ MB/s [S2].

**Scaling:**
Not given in the sources.

**Strengths and limitations:**
- Strengths: Shuffling enables DEFLATE to achieve the best file size reduction among tested lossless methods for scientific data ($\Sigma = 81.2\%$) [S2]. High decompression throughput is possible with shuffling [S2].
- Limitations: Without shuffling, compression is minimal [S2]. High compression levels reduce throughput [S2]. Less efficient than specialized scientific compressors like Szip in some metrics (e.g., Szip has higher compression throughput) [S2].

**When to use it and when not:**
- Use when a general-purpose lossless baseline is needed, especially if shuffling is applied to scientific floating-point data [S2].
- Do not use for scientific data without shuffling if high compression ratios are required [S2].

## Background
Deflate was introduced by Deutsch in 1996 [S2]. It is widely used for general purposes [S2]. In volume rendering and scientific computing, it has been applied as an HDF5 filter (HDF5 version 1.10.0-patch1) in evaluation suites like WaveRange [S2].

## Sources
[S1] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
[S2] WaveRange: wavelet-based data compression for three-dimensional numerical simulations on regular grids, https://doi.org/10.1007/s12650-021-00813-8, fetched 2026-10-29
