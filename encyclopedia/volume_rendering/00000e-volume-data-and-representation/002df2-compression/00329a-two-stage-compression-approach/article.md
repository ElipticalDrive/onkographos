# Two-stage compression approach

Also called: Hybrid CPU/GPU Compression Scheme

## Overview
The two-stage compression approach is a hybrid CPU/GPU scheme for lossless compression and streaming of time-varying volume data, designed to overcome the I/O bottlenecks of mass storage and the PCI Express bus [S1]. It sets itself apart from pure CPU decompression by offloading simple, parallelizable computations and memory copies to the GPU, while reserving complex, serial decompression logic for the CPU, thereby leveraging the GPU's high memory bandwidth for data assembly [S1].

## How it works
The method compresses data twice: first with a simple algorithm suitable for efficient GPU decompression, and second with a CPU-based compression technique [S1]. The pipeline ensures that the output of the first stage remains suitable for the second stage to maintain compression efficiency [S1].

**Inputs and Outputs:**
*   **Input:** Time-varying volume data, typically converted to 16-bit integer scalar values during preprocessing to utilize the full value range based on the dataset's minimum and maximum [S1].
*   **Output:** A compressed bitstream on disk and, upon decompression, a single 3D texture in GPU memory representing the uncompressed volume [S1].

**Data Structures:**
*   **Preprocessed Data:** 16-bit integer scalars. Floating-point simulation data is mapped to this format by spreading values according to the dataset's global minimum and maximum [S1].
*   **Intermediate Representation:** The output of the first compression stage. It must be structured such that it can be effectively compressed by the second stage (e.g., LZO) [S1].
*   **GPU Memory:** A single contiguous 3D texture. The scheme avoids multi-pass bricking and border handling overhead by assembling the data back into its original form as a single texture [S1].

**Numbered Steps:**
1.  **Preprocessing:** Convert floating-point simulation data to 16-bit integer scalars. Spread the values according to the dataset's minimum and maximum to utilize the full 16-bit range [S1].
2.  **First Compression (GPU-friendly):** Compress the preprocessed data using a simple algorithm. This algorithm must be chosen such that its decompression component runs efficiently on the GPU [S1].
3.  **Second Compression (CPU-based):** Compress the output of the first step using a CPU-based compression technique, such as LZO [S1].
4.  **Storage:** Save the doubly compressed data to disk [S1].
5.  **Decompression (CPU):** Perform the main decompression on the CPU to leverage its capacity for complex, serial algorithms [S1].
6.  **Decompression (GPU):** Use the GPU to perform simple computations and memory copies associated with the first stage's decompression and data assembly [S1].
7.  **Assembly:** Assemble the decompressed data into a single 3D texture in GPU memory, avoiding the overhead of managing multiple volume bricks or border handling [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_volume(data_float):
    """
    Compresses time-varying volume data using the two-stage approach.
    """
    # Step 1: Preprocessing
    min_val = min(data_float)
    max_val = max(data_float)
    # Spread values to utilize full 16-bit range [G]
    data_int = scale_to_16bit(data_float, min_val, max_val)
    
    # Step 2: First Compression (Simple, GPU-decompressible)
    # The specific algorithm is not named in the source, but it must be
    # simple enough for GPU parallel execution [S1].
    stage1_compressed = simple_compress(data_int)
    
    # Step 3: Second Compression (CPU-based, e.g., LZO)
    # Ensure output of step 2 is suitable for this step [S1]
    final_compressed = lzo_compress(stage1_compressed)
    
    return final_compressed

def decompress_volume(compressed_data):
    """
    Decompresses data and uploads to GPU as a single 3D texture.
    """
    # Step 5: Main Decompression on CPU
    # Unpack the CPU-based compression (e.g., LZO)
    stage1_compressed = lzo_decompress(compressed_data)
    
    # Step 6 & 7: GPU Decompression and Assembly
    # Transfer stage1_compressed to GPU [G]
    # Launch CUDA kernel to decompress stage1_compressed
    # and assemble into a single 3D texture
    gpu_texture = gpu_decompress_and_assemble(stage1_compressed)
    
    return gpu_texture
```

**Parameters:**
*   **Data Type:** 16-bit integer [S1].
*   **Compression Algorithms:** First stage is unspecified but must be GPU-friendly; second stage is typically LZO [S1].

**Edge Cases and Pitfalls:**
*   **GPU Suitability:** GPUs are poorly suited for compression because most algorithms operate serially and lack coherent branching behavior [S1]. This necessitates the hybrid approach.
*   **Preprocessing Mapping:** The simple linear spreading of float values to 16-bit integers may not be sufficient for all applications; more elaborate mappings might be needed depending on the data distribution [S1].
*   **Brick Management:** Avoiding multi-pass bricking simplifies implementation but requires sufficient GPU memory to hold the single 3D texture [S1].

## Subtopics

### [GPU Decompression Pipeline](7516-gpu-decompression-pipeline.md)
The execution engine for this approach, handling streaming and decompression by overlapping I/O, decompression, and data upload.

## Operating characteristics
*   **Time and Memory Complexity:** The scheme is designed for out-of-core streaming, allowing interactive rendering of datasets that exceed available GPU and main memory [S1]. It avoids the memory overhead of managing multiple bricks by assembling a single 3D texture [S1].
*   **Measured Performance:** Tested on datasets such as Combustion (448×704×128, 77 MB/step), Convection (512×256², 64 MB/step), and Hurricane (512²×128, 64 MB/step) [S1].
*   **Hardware Context:** The approach addresses the bottleneck of mass storage (~110-170 MB/s) and PCI Express bus (~2.5 GB/s) compared to on-device GPU memory bandwidth (e.g., 110 GB/s on NVIDIA GeForce GTX 280) [S1].
*   **Strengths:** Enables near-interactive performance for large, real-world datasets with low signal-to-noise ratio without degrading image quality (lossless) [S1]. Compatible with standard volume raycasting and existing acceleration methods [S1].
*   **Limitations:** Relies on the assumption that simple computations can be effectively parallelized on the GPU, while complex decompression remains on the CPU [S1].

## Background
The method was introduced by J. Mensmann, T. Ropinski, and K. Hinrichs in 2010 as a solution for out-of-core streaming of time-varying volume data [S1]. It addresses the limitations of direct float compression and the inefficiencies of pure CPU or pure GPU decompression strategies [S1].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
