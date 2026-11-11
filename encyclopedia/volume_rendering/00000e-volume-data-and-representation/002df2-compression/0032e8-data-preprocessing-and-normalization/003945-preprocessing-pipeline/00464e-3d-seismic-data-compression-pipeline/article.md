# 3D Seismic Data Compression Pipeline

Also called: .sdc v3 3D Volume Compression Pipeline, SDC3D, SeismicDataCompressor pipeline, compressSegyToSdc3D

## Overview
The 3D Seismic Data Compression Pipeline is a lossy compression method for 3D seismic data that processes volumes in 3D blocks (cubes), applying normalization, closed-loop differential pulse-code modulation (DPCM) with quantization, and Deflate compression to reduce file size while preserving trace integrity [S1, unofficial]. It differs from its sibling [VolumeSegmenter Pipeline](11584-volumesegmenter-pipeline.md) and [Volume Partitioning Pipeline](11585-volume-partitioning-pipeline.md) by operating on fixed 3D spatial-temporal blocks rather than semantic segments or arbitrary partitions, and from the legacy [**.sdc v2 Trace Compression Pipeline**](#-sdc-v2-trace-compression-pipeline) by using 3D predictive coding instead of 1D trace-level encoding [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A SEG-Y dataset (binary format), a `CompressionProfile` (defining bit depth and compression level), and block dimensions (`blockInline`, `blockXline`, `blockTime`).
- Output: An `.sdc` (v3) file containing a header, an index table, and compressed payloads, along with a `CompressionResult` object containing metrics (PSNR, ratios).

**Data Structures:**
1.  **CompressedVolumeBlock3D**:
    -   `blockMeta`: `VolumeBlock3D` object (contains `inlineStartIndex`, `xlineStartIndex`, `timeStartIndex`, `inlineCount`, `xlineCount`, `timeCount`).
    -   `min`: `float` (global minimum of the block before normalization).
    -   `max`: `float` (global maximum of the block before normalization).
    -   `payload`: `byte[]` (compressed quantized DPCM data).
2.  **Sdc3D File Layout**:
    -   `SdcHeader`: `version=3`, `traceCount`, `samplesPerTrace`.
    -   `VolumeDimensions`: `inlineCount`, `xlineCount`, `timeCount`.
    -   `BlockDimensions`: `blockInline`, `blockXline`, `blockTime`.
    -   `IndexTable`: Array of `blockCount` entries. Each entry contains: `inlineStartIndex`, `xlineStartIndex`, `timeStartIndex`, `inlineCount`, `xlineCount`, `timeCount`, `min`, `max`, `payloadSize`.
    -   `CompressedPayloads`: Array of `blockCount` byte arrays, corresponding to the index table.

**Parameters:**
-   `blockInline`: Integer. Default 8 if <= 0.
-   `blockXline`: Integer. Default 8 if <= 0.
-   `blockTime`: Integer. Default `timeCount` if <= 0 or > `timeCount`.
-   `bitDepth`: Integer from `CompressionProfile`. Controls quantization precision.
-   `compressionLevel`: Integer from `CompressionProfile`. Controls Deflate compression level.

**Steps:**
1.  **Read Input**: Load the SEG-Y dataset. Determine `traceCount` and `samplesPerTrace`. Calculate `segyBytes` (raw file size) and `rawDataBytes` (`traceCount * samplesPerTrace * 4` bytes, assuming 32-bit floats).
2.  **Partition into Blocks**: Divide the 3D volume into non-overlapping blocks of size `blockInline x blockXline x blockTime`. Handle edge cases where volume dimensions are not multiples of block sizes by padding or truncating (implementation detail: [G] typically pads with zeros or clips).
3.  **Process Each Block**:
    a.  **Calculate Min/Max**: Iterate over all samples in the block to find global `min` and `max`.
    b.  **Normalize**: Map samples to `[-1, 1]`.
        -   If `max == min` (degenerate case), set all normalized values to `0f`.
        -   Otherwise: $norm = \frac{v - min}{max - min} \times 2 - 1$.
    c.  **Closed-Loop DPCM with Quantization**: Apply 3D differential encoding.
        -   Predict the current sample based on previously processed neighbors (e.g., inline, xline, time predecessors).
        -   Calculate the difference: $diff = v_{norm} - v_{pred}$.
        -   Quantize the difference based on `bitDepth`.
        -   **Crucial**: Use the *quantized* reconstructed value for subsequent predictions (closed-loop), not the original normalized value. This ensures decoder synchronization without transmitting original floats.
    d.  **Compress Payload**: Compress the quantized DPCM data using Java's `Deflate` algorithm with the specified `compressionLevel`. Store `min`, `max`, and the compressed `payload` in a `CompressedVolumeBlock3D` structure.
4.  **Write SDC3D File**:
    a.  Write `SdcHeader`.
    b.  Write `VolumeDimensions` and `BlockDimensions`.
    c.  Write `IndexTable`: For each block, write its metadata (`min`, `max`, dimensions, start indices) and `payloadSize`.
    d.  Write `CompressedPayloads`: Append the raw bytes of each block's payload.
5.  **Reconstruct and Evaluate (Optional/Verification)**:
    a.  Read the SDC3D file.
    b.  For each block, decompress payload, de-quantize, and apply inverse DPCM to reconstruct normalized values.
    c.  Denormalize using stored `min` and `max`: $v_{rec} = \frac{norm_{rec} + 1}{2} \times (max - min) + min$.
    d.  Reconstruct traces from the volume.
    e.  Compute PSNR per trace between original and reconstructed traces. Track `min`, `max`, `mean` PSNR.
6.  **Return Result**: Return `CompressionResult` with `segyBytes`, `sdcBytes`, `ratioFile`, `ratioData`, `savingsPercent`, and PSNR metrics.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_segy_to_sdc3d(segy_path, sdc_path, profile, block_inline=8, block_xline=8, block_time=None):
    # Step 1: Read Input
    dataset = read_segy(segy_path)
    trace_count = dataset.trace_count
    samples_per_trace = dataset.samples_per_trace
    time_count = samples_per_trace
    inline_count = dataset.inline_count
    xline_count = dataset.xline_count
    
    segy_bytes = get_file_size(segy_path)
    raw_data_bytes = trace_count * samples_per_trace * 4  # 4 bytes per float32
    
    if block_time is None or block_time <= 0 or block_time > time_count:
        block_time = time_count
        
    # Step 2: Partition and Process
    index_table = []
    payloads = []
    
    for i_start in range(0, inline_count, block_inline):
        for j_start in range(0, xline_count, block_xline):
            for k_start in range(0, time_count, block_time):
                # Determine actual block dimensions (handle edges)
                i_end = min(i_start + block_inline, inline_count)
                j_end = min(j_start + block_xline, xline_count)
                k_end = min(k_start + block_time, time_count)
                
                i_len = i_end - i_start
                j_len = j_end - j_start
                k_len = k_end - k_start
                
                # Extract block data
                block_data = extract_block(dataset, i_start, j_start, k_start, i_len, j_len, k_len)
                
                # Step 3a: Calculate Min/Max
                min_val = min(block_data.flatten())
                max_val = max(block_data.flatten())
                
                # Step 3b: Normalize
                if max_val == min_val:
                    norm_data = zeros_like(block_data)
                else:
                    range_val = max_val - min_val
                    norm_data = ((block_data - min_val) / range_val) * 2 - 1
                
                # Step 3c: Closed-Loop DPCM with Quantization
                # Assuming a simple 3D predictor: previous time sample, or inline/xline neighbors
                # For simplicity, assume 1D DPCM along time for this pseudocode, 
                # but the source specifies 3D DPCM. 
                # We will implement a generic 3D closed-loop DPCM.
                
                quantized_diffs = zeros_like(norm_data)
                reconstructed = zeros_like(norm_data)
                
                # Iterate in processing order (e.g., i, j, k)
                for i in range(i_len):
                    for j in range(j_len):
                        for k in range(k_len):
                            # Get original normalized value
                            v_norm = norm_data[i, j, k]
                            
                            # Predict based on neighbors (closed-loop uses reconstructed values)
                            # Example predictor: average of previous time, inline, xline if available
                            pred = 0.0
                            if k > 0: pred += reconstructed[i, j, k-1]
                            if i > 0: pred += reconstructed[i-1, j, k]
                            if j > 0: pred += reconstructed[i, j-1, k]
                            # Normalize prediction (e.g., divide by number of neighbors)
                            neighbors = (1 if k > 0 else 0) + (1 if i > 0 else 0) + (1 if j > 0 else 0)
                            if neighbors > 0:
                                pred = pred / neighbors
                            else:
                                pred = 0.0 # First sample or edge case
                            
                            # Calculate difference
                            diff = v_norm - pred
                            
                            # Quantize difference
                            # Assuming bit_depth determines the range of quantized values
                            # For example, if bit_depth=8, quantize to [-128, 127]
                            # This is a simplification; actual implementation may vary
                            quantized_diff = quantize(diff, profile.bit_depth)
                            
                            quantized_diffs[i, j, k] = quantized_diff
                            
                            # Reconstruct for closed-loop
                            reconstructed[i, j, k] = pred + dequantize(quantized_diff, profile.bit_depth)
                
                # Step 3d: Compress Payload
                payload_bytes = deflate_compress(quantized_diffs.tobytes(), profile.compression_level)
                
                # Store block info
                block_meta = VolumeBlock3D(i_start, j_start, k_start, i_len, j_len, k_len)
                index_table.append({
                    'meta': block_meta,
                    'min': min_val,
                    'max': max_val,
                    'payload_size': len(payload_bytes)
                })
                payloads.append(payload_bytes)
    
    # Step 4: Write SDC3D File
    write_sdc3d_header(sdc_path, trace_count, samples_per_trace, inline_count, xline_count, time_count, block_inline, block_xline, block_time)
    write_index_table(sdc_path, index_table)
    write_payloads(sdc_path, payloads)
    
    # Step 5: Evaluate (Reconstruct and Compute PSNR)
    reconstructed_dataset = read_sdc3d(sdc_path)
    psnr_values = []
    for t_idx in range(trace_count):
        original_trace = get_trace(dataset, t_idx)
        reconstructed_trace = get_trace(reconstructed_dataset, t_idx)
        psnr = compute_psnr(original_trace, reconstructed_trace)
        psnr_values.append(psnr)
        
    psnr_min = min(psnr_values)
    psnr_max = max(psnr_values)
    psnr_mean = mean(psnr_values)
    
    # Step 6: Return Result
    sdc_bytes = get_file_size(sdc_path)
    ratio_file = sdc_bytes / segy_bytes
    ratio_data = sdc_bytes / raw_data_bytes
    savings_percent = (1.0 - ratio_file) * 100.0
    
    return CompressionResult(
        segy_path=segy_path,
        sdc_path=sdc_path,
        segy_bytes=segy_bytes,
        sdc_bytes=sdc_bytes,
        raw_data_bytes=raw_data_bytes,
        trace_count=trace_count,
        samples_per_trace=samples_per_trace,
        ratio_file=ratio_file,
        ratio_data=ratio_data,
        savings_percent=savings_percent,
        psnr_first=psnr_values[0] if psnr_values else 0,
        psnr_mean=psnr_mean,
        psnr_min=psnr_min,
        psnr_max=psnr_max
    )
```

## Subtopics

### [3D Volume Block Compression (Closed-Loop)](11729-3d-volume-block-compression-closed-loop.md)
This method compresses 3D seismic data blocks by normalizing samples to a fixed range, applying closed-loop differential pulse-code modulation (DPCM) with quantization, and packing the result with Deflate compression. It differs from open-loop DPCM by feeding the reconstructed (quantized) value back into the predictor, ensuring the decoder can exactly reproduce the quantization steps without requiring the original float values during decoding. It is a core component of the 3D Seismic Data Compression Pipeline [11727-3d-seismic-data-compression-pipeline.md].

### .sdc v2 Trace Compression Pipeline
This is a legacy variant that processes data trace-by-trace rather than in 3D blocks. It reads a SEG-Y dataset, compresses each trace individually using a `CompressionProfile`, and writes an `.sdc` file. It computes compression ratios based on `segyBytes` and `rawDataBytes` (`traceCount * samplesPerTrace * 4`) and evaluates quality via PSNR per trace [S1, unofficial]. It is simpler but lacks the 3D spatial redundancy exploitation of the v3 pipeline.

## Operating characteristics
-   **Time Complexity**: $O(N)$ where $N$ is the total number of samples, as each sample is processed once for normalization, DPCM, and quantization. Deflate compression is also roughly linear in practice.
-   **Memory Complexity**: $O(B)$ where $B$ is the block size, as only one block is processed at a time. The entire volume is not held in memory during compression.
-   **Measured Performance**: Not given in the sources.
-   **Scaling**: Scales well with block size; larger blocks may yield better compression due to more spatial redundancy but increase memory usage per block.
-   **Strengths**: Preserves trace integrity (important for seismic interpretation), lossy but controllable via bit depth, efficient for large 3D volumes.
-   **Limitations**: Lossy compression may introduce artifacts; closed-loop DPCM requires careful implementation to avoid error propagation; block boundaries may cause discontinuities if not handled.
-   **When to Use**: When compressing large 3D seismic datasets for storage or transmission, and trace-level fidelity is important.
-   **When Not to Use**: When lossless compression is required, or when data has very low spatial correlation (e.g., random noise).

## Background
The 3D Seismic Data Compression Pipeline is part of the SeismicDataCompressor project, an unofficial open-source tool for compressing seismic data [S1, unofficial]. It represents an evolution from the 1D trace-based compression (.sdc v2) to a more efficient 3D block-based approach, leveraging spatial redundancy in seismic volumes.

## Sources
[S1] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
