# Image-Plus-Geometry Compression (IGC)

## Overview
Image-Plus-Geometry Compression (IGC) is a reference compression scheme for Surface Light Fields (SLF) where the source images are compressed using a standard video codec and the geometry is transmitted separately, with the SLF representation reconstructed at the decoder. It serves as a baseline to evaluate the efficiency of direct SLF coefficient compression methods, such as those using RAHT or TMC, by shifting the computationally expensive SLF construction step from the encoder to the decoder. Unlike direct SLF compression, which transmits compressed coefficients, IGC transmits raw visual data (images) and structural data (geometry), resulting in significantly higher decoder complexity.

## How it works
**Inputs and Outputs:**
- **Input:** A set of captured source images and their corresponding camera poses/geometry.
- **Output:** A bitstream containing compressed images and compressed geometry, which the decoder uses to reconstruct the SLF representation and render novel viewpoints.

**Data Structures:**
- **Geometry:** Compressed representation of camera positions and orientations. The specific bit-packing is not detailed in the sources, but it is treated as a separate stream from the images [S1].
- **Images:** Compressed using a video codec (specifically HEVC in the reference implementation) [S1].

**Steps:**
1. **Encoder Side:**
   1. Compress the captured source images using a video codec. The reference implementation uses HM-16.2, the reference software for HEVC, with low delay common test conditions [S1].
   2. Compress the geometry data (camera positions/orientations) [S1].
   3. Transmit the compressed images and compressed geometry to the decoder [S1].

2. **Decoder Side:**
   1. Decompress the geometry data [S1].
   2. Decompress the source images using the corresponding video decoder [S1].
   3. Construct the SLF representation from the decoded images and geometry. This involves approximating the view map $f(\omega|p)$ at each point $p$ as a linear combination of basis functions, effectively reversing the compression of the visual data into the SLF domain [S1].
   4. Render arbitrary viewpoints using the constructed SLF representation [S1].

**Parameters:**
- **Video Codec:** HEVC (High Efficiency Video Coding) [S1].
- **Reference Software:** HM-16.2 [S1].
- **Coding Mode:** Low delay common test conditions [S1].

**Edge Cases and Pitfalls:**
- The decoder complexity is significantly higher than direct SLF compression because the SLF construction must be performed in real-time or near real-time on the decoded images, rather than being pre-computed at the encoder [S1].
- Precision issues may arise if the video codec introduces artifacts that degrade the quality of the reconstructed SLF representation, although specific quantization parameters are not detailed in the sources [S1].

## Operating characteristics
- **Time Complexity:** The decoder running time is dominated by the SLF representation construction. For the Fish/Elephant datasets, IGC total decoder time is 37.68 seconds, compared to 5.82 seconds (RAHT) and 24.53 seconds (TMC) for direct SLF compression [S1].
- **Breakdown (IGC):** Geometry Decompression: 0.12s; Images Decompression: 1.92s; SLF representation: 35.13s; Rendering: 0.51s [S1].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Leverages mature, high-performance video codecs (HEVC) for image compression, potentially offering good rate-distortion performance for the image component [S1].
- **Limitations:** Significantly higher decoder complexity compared to direct SLF compression methods. The SLF construction at the decoder is computationally expensive (35.13s in the reference case) [S1].
- **When to use:** When a fair comparison with state-of-the-art video coding is required, or when encoder-side SLF construction is not feasible but decoder-side reconstruction is acceptable.
- **When not to use:** When low-latency or low-complexity decoding is required, as direct SLF compression (RAHT/TMC) is substantially faster [S1].

## Background
IGC was introduced as an alternative reference scheme for fair comparison with SLF compression methods [S1]. It highlights the trade-off between encoder and decoder complexity in light field compression. The method relies on HEVC, a state-of-the-art video coding standard at the time of the study [S1]. The performance comparison demonstrates that while IGC uses efficient image coding, the cost of reconstructing the SLF at the decoder makes it less efficient in terms of total decoder time compared to methods that compress the SLF coefficients directly [S1].

## Sources
[S1] Surface Light Field Compression Using a Point Cloud Codec, https://doi.org/10.1109/jetcas.2018.2883479, fetched 2026-10-28
