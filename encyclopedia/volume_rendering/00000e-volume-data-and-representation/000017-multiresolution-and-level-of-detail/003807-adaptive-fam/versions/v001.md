# Adaptive-FAM

Also called: Adaptive Functional Approximation Multi-resolution, Adaptive functional approximation multi-resolution encoding, Adaptive-FAM Encoding Pipeline

## Overview
Adaptive-FAM is a multi-resolution encoding pipeline designed for the out-of-core interactive visualization of large-scale volumetric datasets [S1, unofficial]. It partitions input data into blocks, downsamples them into micro-blocks, and encodes each micro-block into a compact "micro-model" using functional approximation with dynamically optimized Normalized Control Points (NCP) [S1, unofficial]. Unlike its sibling [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) which relies on frequency-domain transforms, Adaptive-FAM uses spatial partitioning and adaptive polynomial approximation to minimize model size while maintaining rendering accuracy [S1, unofficial]. It is specifically optimized to reduce I/O caching time and input latency in out-of-core rendering frameworks [S1, unofficial].

## How it works
**Inputs:** A 3D volumetric dataset of resolution $n^3$.
**Outputs:** A collection of micro-models organized by Level of Detail (LOD), ready for out-of-core rendering.

**Data Structures:**
1.  **Micro-block:** A downsampled sub-volume of the original dataset.
2.  **Micro-model:** The encoded representation of a micro-block, consisting of a set of Normalized Control Points (NCP) and associated basis functions. The size of the micro-model is determined by the number of NCPs used.
3.  **Error Map:** A lookup structure mapping valid NCP values to their corresponding reconstruction error for a specific micro-block.

**Steps:**

1.  **Partitioning:** Partition the input 3D dataset into blocks for each LOD level. To ensure C0 continuity across block boundaries, neighboring blocks must share exactly one sample point on the boundary in each of the x, y, and z directions [S1, unofficial].
2.  **Downsampling:** Downsample each partitioned block into micro-blocks according to the resolution requirements of the specific LOD [S1, unofficial].
3.  **Adaptive Encoding:** For each micro-block, determine the optimal number of NCPs using an adaptive encoding strategy that incorporates both in-level and cross-level adaptive techniques [S1, unofficial].
    *   **In-level Adaptive Encoding:** Perform an exhaustive search to find the suitable NCP for a required error bound $e$. For a micro-block with resolution $n^3$, the encoder tests each valid NCP where $NCP \in [\text{degree} + 1, n]$. It records the corresponding error metrics to construct an error map $\text{Error}(NCP)$ [S1, unofficial]. The optimal NCP, $NCP^*$, is determined by:
        $$ NCP^* = \arg \min_{NCP} (\text{Error}(NCP) < e) $$
        A larger NCP typically results in lower error, allowing the search to be optimized (e.g., by checking larger NCPs first or using binary search if the error function is monotonic, though the source describes an exhaustive search recording a map) [S1, unofficial].
    *   **Cross-level Adaptive Encoding:** (Details not explicitly provided in the source text beyond the mention of its inclusion in the pipeline) [S1, unofficial].
4.  **Storage:** Store the micro-models. The micro-models are stored in a cache (fixed size in the reference implementation) and loaded on demand during rendering [S1, unofficial].

**Parameters:**
*   **Cache Size:** Fixed at 200 micro-blocks in the reference implementation [S1, unofficial].
*   **Sample Distance:** Used for quality evaluation, set to 0.000025 in the reference experiments [S1, unofficial].
*   **Error Bound ($e$):** The maximum allowable reconstruction error for a micro-block, used to determine $NCP^*$ [S1, unofficial].

**Pitfalls:**
*   **Highly Dynamic Data:** The benefit of Adaptive-FAM's encoding time reduction is reduced when input dataset values are highly dynamic throughout the entire volumetric space [S1, unofficial].
*   **Aggressive Compression:** Rendering accuracy falls behind other compressors (SZ3, TTHRESH) when the average compression ratio exceeds 40 [S1, unofficial].

## Subtopics

### [Out-of-Core GPU-Accelerated Multi-Resolution Rendering Framework](8169-out-of-core-gpu-accelerated-multi-resolution-rendering-frame.md)
A rendering framework consisting of three main procedures: Caching, Prefetching, and Rendering, designed to handle large-scale volumes by managing data movement between storage, RAM, and GPU.

## Operating characteristics
**Time and Memory Complexity:**
*   **Encoding Time:** Adaptive-FAM is significantly faster than the baseline Functional Approximation Multi-resolution (FAM) method. For example, on the Branch dataset, FAM took 18.29 hours while Adaptive-FAM took 2.26 hours [S1, unofficial]. On the Rotstrat dataset, FAM took 292.68 hours while Adaptive-FAM took 172.16 hours [S1, unofficial].
*   **Model Size:** Adaptive-FAM produces much smaller models than FAM. For the ML dataset, FAM required 7.7GB while Adaptive-FAM required 232MB [S1, unofficial].
*   **Rendering Time:** Rendering time is almost constant compared to the growth of caching time as data size increases, because querying time grows sub-linearly with NCP [S1, unofficial].
*   **Caching Time:** Adaptive-FAM achieves much shorter caching times than FAM due to smaller micro-model sizes [S1, unofficial].

**Measured Performance:**
*   **Hardware:** NVIDIA GTX 1080 Ti GPU (8GB VRAM), Intel Core i7-7700K CPU, 16GB DDR4 RAM, 4TB HDD (5400 RPM SATA) [S1, unofficial].
*   **Compression Ratios:** Ranged from 15.71 (Rayleigh-Taylor) to 35.31 (Flame) in tested datasets [S1, unofficial].
*   **Comparison with DS:** Adaptive-FAM is generally faster than Down Sampling (DS) for large-scale datasets, except on the Flame dataset where DS is slightly faster. DS struggles with large datasets because caching time becomes the bottleneck and query time scales linearly with micro-block size [S1, unofficial].
*   **Comparison with Compressors:** At similar compression ratios (~20), Adaptive-FAM produces fewer artifacts than ZFP, SZ3, and TTHRESH, particularly avoiding boundary artifacts [S1, unofficial]. Adaptive-FAM does not require a decompression process, avoiding decompression latency overhead which increases with compression ratio [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Superior rendering quality compared to DS; significantly reduced input latency; effective decrease in cache miss rates when used with prefetching algorithms (APPA, ForeCache, LSTM) [S1, unofficial].
*   **Limitations:** Accuracy falls behind SZ3 and TTHRESH for aggressive compression (ratio > 40); encoding time benefits are reduced for highly dynamic volumetric data [S1, unofficial].

## Background
Adaptive-FAM was proposed to address the challenge of encoding large-scale volumetric datasets efficiently for interactive visualization. Traditional functional approximation methods (FAM) can become impractical due to long encoding times and large model sizes. Adaptive-FAM introduces an adaptive encoding pipeline to optimize the number of Normalized Control Points (NCP) for each micro-block, balancing accuracy and size. It is used in a GPU-accelerated out-of-core multi-resolution framework for interactive visualization of large-scale volumetric datasets [S1, unofficial].

## Sources
[S1] Adaptive Multi-Resolution Encoding for Interactive Large-Scale Volume Visualization through Functional Approximation, https://www.researchgate.net/publication/383700795_Adaptive_Multi-Resolution_Encoding_for_Interactive_Large-Scale_Volume_Visualization_through_Functional_Approximation, fetched 2026-10-30 (unofficial)
