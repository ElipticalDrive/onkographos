# VolRec Interlacing


## Overview
VolRec Interlacing is a volumetric reconstruction technique that generates full-resolution 3D volumes from lower-resolution acquired volumes by combining spatial interlace/interpolation with temporal interpolation using data from previous volumes [S1]. It differs from classic OCT processing by reducing acquisition resolution to increase speed, then reconstructing full resolution via 4D spatiotemporal interpolation, rather than processing full-resolution volumes directly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of low-resolution volumes acquired in real-time (typically < 40 ms per volume) [S1]. Each low-resolution volume contains a subset of A-scans from the full sampling space [S1].
*   **Output:** A full-resolution volume at each timestamp, reconstructed by merging A-scans from the current and previous epochs [S1].

**Data Structures:**
*   **Low-Resolution Volume:** A 3D grid of voxels. In the reference implementation, dimensions are $60 \times 60 \times 8192$ [S1]. The depth dimension (8192) results from an FFT of A-scans where the complex-to-float translation constrains the size to half the original depth [S1].
*   **Full-Resolution Volume:** A 3D grid of voxels. In the reference implementation, dimensions are $300 \times 300 \times 8192$ [S1].
*   **Voxel Data Type:** Float (32-bit) for processing, potentially converted to `uint8_t` for host-to-device or device-to-host transfers to reduce bandwidth [S1].

**Parameters:**
*   **Stride ($s$):** The spacing between sampled A-scans in the full-resolution space. The reference uses a stride of 5 in the x and y dimensions [S1]. This results in a volume size 25 times smaller than the maximum resolution ($5 \times 5 = 25$) [S1].
*   **Offset ($o_x, o_y$):** Dynamic offsets applied to the low-resolution coordinates to map them to the full-resolution grid. These offsets change per epoch to ensure complete coverage of the full-resolution space over time [S1].
*   **Convolution Kernel Size:** $3 \times 3 \times 3$ [S1].

**Steps:**
1.  **Acquisition:** Acquire a low-resolution volume at epoch $N$. The physical device is configured to sample A-scans with a constant stride $s$ and a dynamic offset $(o_x, o_y)$ specific to epoch $N$ [S1].
2.  **Preprocessing:** Apply preprocessing to the low-resolution volume [S1]:
    *   Subtract the mean of each A-scan to center the data [S1].
    *   Perform an FFT for each A-scan with a depth size of 8192 voxels [S1].
    *   Apply logarithm scaling [S1].
    *   Flip horizontal and vertical axes [S1].
    *   Apply range-based pixel normalization [S1].
3.  **Mapping (Interlacing):** For each A-scan in the low-resolution volume at coordinates $(x, y)$, calculate its position $(X, Y)$ in the full-resolution volume using the transformation function [S1]:
    $$ T(x, y) = (x \cdot s + o_x, y \cdot s + o_y) $$
    where $s$ is the stride, and $o_x, o_y$ are the offsets for the current epoch [S1]. Place the A-scan data at $(X, Y)$ in the full-resolution volume [S1].
4.  **Temporal Merging:** Repeat steps 1-3 for previous epochs (e.g., $N-1, N-2, \dots$) with their respective offsets. The offsets are chosen such that the union of all sampled A-scans from the current and previous epochs covers the entire full-resolution space [S1]. For example, with a stride of 5 and 5 epochs, the offsets might be $(0,0), (1,0), (2,0), (3,0), (4,0)$ in one dimension, ensuring every position in the $5 \times 5$ grid is filled over 5 epochs [S1].
5.  **Spatio-Temporal Convolution:** Apply a custom $3 \times 3 \times 3$ convolution kernel to the interlaced full-resolution volume [S1]. The kernel weights are selected to give more relevance to neighbor voxels that are newer and closer [S1]. This step recovers the original resolution quality lost due to sample space reduction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volrec_interlacing(low_res_volumes, stride=5, num_epochs=5):
    """
    Reconstructs a full-resolution volume from a sequence of low-resolution volumes.
    
    Args:
        low_res_volumes: List of low-resolution volumes, ordered from oldest to newest.
                         Each volume is a 3D array (60x60x8192).
        stride: The stride used during acquisition (default 5).
        num_epochs: The number of previous epochs to use for reconstruction (default 5).
    
    Returns:
        full_res_volume: The reconstructed full-resolution volume (300x300x8192).
    """
    # Initialize full-resolution volume with zeros or appropriate background
    # Dimensions: stride * low_res_dim_x, stride * low_res_dim_y, depth
    # Assuming low_res_dim_x = low_res_dim_y = 60, depth = 8192
    full_res_volume = create_zero_volume(300, 300, 8192)
    
    # Calculate offsets for each epoch to cover the stride x stride grid
    # Offsets range from 0 to stride-1
    offsets = []
    for i in range(num_epochs):
        ox = i % stride
        oy = (i // stride) % stride
        offsets.append((ox, oy))
    
    # Map low-resolution A-scans to full-resolution positions
    for epoch_idx, low_res_vol in enumerate(low_res_volumes):
        if epoch_idx >= num_epochs:
            break
            
        ox, oy = offsets[epoch_idx]
        
        # Iterate over low-resolution coordinates
        for y in range(low_res_vol.shape[1]):
            for x in range(low_res_vol.shape[0]):
                # Calculate full-resolution coordinates
                full_x = x * stride + ox
                full_y = y * stride + oy
                
                # Extract A-scan data (depth dimension)
                # low_res_vol[y, x, :] contains the A-scan at (x, y)
                a_scan = low_res_vol[y, x, :]
                
                # Place A-scan in full-resolution volume
                # full_res_volume[full_y, full_x, :] = a_scan
                # Note: In memory, this might be row-major or column-major depending on implementation
                full_res_volume[full_y, full_x, :] = a_scan
                
    # Apply spatio-temporal convolution
    # Kernel size 3x3x3, weights favor newer and closer voxels
    full_res_volume = apply_spatio_temporal_convolution(full_res_volume)
    
    return full_res_volume

def apply_spatio_temporal_convolution(volume):
    """
    Applies a 3x3x3 convolution kernel to the volume.
    Weights are higher for newer and closer voxels.
    """
    # Implementation details of the kernel weights are not fully specified in sources [S1]
    # but the kernel size is 3x3x3 and it operates on the interlaced volume [S1].
    # [G] Standard 3D convolution with boundary handling (e.g., zero-padding or clamping)
    # would be required. The specific weight values are empirical and not provided [S1].
    pass
```

**Edge Cases and Pitfalls:**
*   **Data Quality Reduction:** The sample space reduction (e.g., 5x5) reduces data quality, requiring convolution to recover original resolution [S1].
*   **Precision vs. Speed:** A stride of 5 is empirically found to be a good trade-off between precision and speed [S1]. Further research can vary the stride [S1].
*   **Offset Calculation:** Offsets must be carefully calculated to ensure complete coverage of the full-resolution space over the specified number of epochs [S1].

## Subtopics

### [VolRec](20869-volrec.md)
VolRec is the overarching volumetric reconstruction technique that generates full-resolution 3D volumes from lower-resolution acquired volumes by combining spatial interlace/interpolation with temporal interpolation using data from previous volumes. It differs from classic OCT processing by reducing acquisition resolution to increase speed, then reconstructing full resolution via 4D spatiotemporal interpolation, rather than processing full-resolution volumes directly. [VolRec](20869-volrec.md)

### [VolRec Cumulative Processing](20895-volrec-cumulative-processing.md)
VolRec Cumulative Processing is a variant of the VolRec interlacing method that improves temporal consistency by merging new low-resolution data with the previously convolved full-resolution volume rather than raw interlaced data. It solves the problem of error accumulation in raw data by ensuring that the input to the interlacing step is already spatially and temporally smoothed. It differs from its sibling [VolRec Non-Cumulative Processing](20869-volrec-non-cumulative.md) by using the final output of the previous epoch as the base for merging, whereas the non-cumulative variant uses the intermediate interlaced volume. [VolRec Cumulative Processing](20895-volrec-cumulative-processing.md)

### Host-to-Device Copy Optimization via Interlacing
An optimization that moves the interlacing operation to the GPU to reduce the size of data copied from host to device, replacing the high-resolution volume copy with a low-resolution volume copy [S1]. The interlacing data is processed in parallel on the GPU [S1]. Only the low-resolution preprocessed volume is copied to the device instead of the whole high-resolution merged volume [S1]. The high-resolution volume size is $300 \times 300 \times 4096 \times 4$ bytes $\approx 1.4$ GB, while the low-resolution volume size is $60 \times 60 \times 4096 \times 4$ bytes $\approx 0.056$ GB, resulting in a 25 times smaller size [S1]. This allows for a theoretical 25× speedup for host-to-device copies [S1]. This optimization achieves a processing time of 204.83 ms (4.88 volumes per second), which is a 2.97× speedup over the naive CUDA version [S1].

## Operating characteristics
*   **Time Complexity:** The interlacing operation itself is linear with respect to the number of voxels in the low-resolution volume [S1]. The convolution step is also linear with respect to the full-resolution volume size, with a constant factor determined by the kernel size ($3 \times 3 \times 3$) [S1].
*   **Memory Complexity:** The full-resolution volume requires $300 \times 300 \times 4096 \times 4$ bytes $\approx 1.4$ GB of memory [S1]. The low-resolution volume requires $60 \times 60 \times 4096 \times 4$ bytes $\approx 0.056$ GB [S1].
*   **Measured Performance:** With the host-to-device copy optimization, the method processes a volume in 204.83 ms, achieving 4.88 volumes per second [S1]. This is a 2.97× speedup over the naive CUDA version [S1].
*   **Strengths:** Significantly reduces acquisition time by sampling at a lower resolution (25 times faster acquisition) [S1]. The host-to-device copy optimization reduces bandwidth bottlenecks [S1].
*   **Limitations:** Data quality is reduced due to sample space reduction, requiring convolution to recover resolution [S1]. The method relies on temporal consistency, assuming the scene does not change significantly between epochs [S1].
*   **When to Use:** Suitable for real-time volumetric reconstruction where acquisition speed is critical and temporal coherence is high, such as in OCT imaging [S1].
*   **When Not to Use:** Not suitable for scenes with rapid, non-temporal changes or where high spatial fidelity is required without temporal averaging [S1].

## Background
VolRec Interlacing was introduced in the paper "VolRec: 4D real-time volumetric reconstruction of OCT data" by A. Vicente-Jaén et al. [S1]. The method addresses the challenge of acquiring high-resolution 3D volumes in real-time, particularly in Optical Coherence Tomography (OCT) [S1]. By reducing the acquisition resolution and using spatiotemporal interpolation, the method achieves real-time performance while maintaining acceptable image quality [S1].

## Sources
[S1] VolRec: 4D real-time volumetric reconstruction of OCT data, https://doi.org/10.1007/s11227-025-06969-6, fetched 2026-10-26
