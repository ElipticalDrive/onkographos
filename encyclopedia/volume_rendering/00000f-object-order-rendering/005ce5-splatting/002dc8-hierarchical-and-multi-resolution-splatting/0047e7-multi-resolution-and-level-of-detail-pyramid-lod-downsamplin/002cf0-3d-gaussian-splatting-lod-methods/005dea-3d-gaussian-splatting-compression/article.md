# 3D Gaussian Splatting Compression

Also called: Compact3D, EGGS (Explicitly Granular 3D Gaussian Splatting), Explicitly Granular 3D Gaussian Splatting

## Overview
3D Gaussian Splatting Compression reduces the storage footprint of 3D Gaussian Splatting scenes by compressing geometric and attribute data [S2, unofficial]. Compact3D avoids quantizing position and opacity to prevent overlapping Gaussians and visual artifacts [S1]. EGGS employs Luma-Aware and Volume-Preserving Attribute Factorization [S2, unofficial]. These methods build on [3D Gaussian Splatting LOD methods](5439-3d-gaussian-splatting-lod-methods.md) by focusing on bitstream reduction rather than hierarchical rendering structures [G].

## How it works
The EGGS pipeline compresses Gaussian means using G-PCC (Geometry-based Point Cloud Compression) [S2, unofficial]. Other attributes are handled separately or via factorization [S2, unofficial].

### Inputs and Outputs
- **Input**: A set of $N$ 3D Gaussians, each defined by a mean position $\mathbf{p} \in \mathbb{R}^3$, covariance (scale/rotation), opacity $\alpha$, and spherical harmonics coefficients for color [G].
- **Output**: A binary file containing a compressed geometry stream and a header [S2, unofficial].

### Data Structures
The output binary file structure is [S2, unofficial]:
1. **Header**: 24 bytes.
   - `means_min`: 3 floats (float32), the minimum x, y, z coordinates of the Gaussian means.
   - `means_max`: 3 floats (float32), the maximum x, y, z coordinates of the Gaussian means.
2. **Length**: 4 bytes (uint32), the size in bytes of the compressed geometry stream [S2, unofficial].
3. **Compressed Data**: Variable length, the binary stream output from the GPCC encoder [S2, unofficial].

### Parameters
- **VOXELIZE_SCALE_FACTOR**: 16. The bit depth used for voxelizing Gaussian means, defining an integer range of $[0, 2^{16}-1]$ [S2, unofficial].
- **GPCC Encoder Flags** (used for `tmc3`): `mode=0`, `trisoupNodeSizeLog2=0`, `mergeDuplicatedPoints=0`, `neighbourAvailBoundaryLog2=8`, `positionQuantisationEnabled=1`, `intra_pred_max_node_size_log2=3`, `positionQuantizationScale=1`, `inferredDirectCodingMode=3`, `maxNumQtBtBeforeOt=2`, `minQtbtSizeLog2=0`, `planarEnabled=0`, `planarModeIdcmUse=0`, `cabac_bypass_stream_enabled_flag=1` [S2, unofficial].
- **GPCC Decoder Flags**: `mode=1`, `outputBinaryPly=1` [S2, unofficial].

### Steps
#### Encoding Pipeline
1. **Voxelization of Means**: Calculate the bounding box of all Gaussian means: $\mathbf{p}_{min} = \min_i (\mathbf{p}_i)$ and $\mathbf{p}_{max} = \max_i (\mathbf{p}_i)$ [S2, unofficial]. Normalize each mean $\mathbf{p}_i = [x_i, y_i, z_i]^T$ to $[0, 1]$: $\mathbf{p}'_i = \frac{\mathbf{p}_i - \mathbf{p}_{min}}{\mathbf{p}_{max} - \mathbf{p}_{min}}$ [S2, unofficial]. Convert to 16-bit integers: $\mathbf{v}_i = \text{round}(\mathbf{p}'_i \times (2^{16} - 1))$ [S2, unofficial].
2. **Sorting by Morton Code**: Calculate the Morton code for each voxelized mean $\mathbf{v}_i$ as the dot product of the voxelized coordinates and powers of $(\text{max\_coordinate} + 1)$, where $\text{max\_coordinate} = 2^{16} - 1$ [S2, unofficial]. Sort the voxelized means $\mathbf{v}_i$ in ascending order of their Morton codes [S2, unofficial].
3. **PLY File Generation**: Write the sorted voxelized means to an ASCII PLY file containing x, y, z properties [S2, unofficial].
4. **GPCC Compression**: Execute the external GPCC encoder (`tmc3`) with the specified flags [S2, unofficial]. Input is the ASCII PLY file; output is a binary compressed stream [S2, unofficial].
5. **File Assembly**: Write the 24-byte header (`means_min`, `means_max`), followed by the 4-byte length of the compressed stream, followed by the compressed binary stream [S2, unofficial].

#### Decoding Pipeline
1. **Header and Stream Extraction**: Read the first 24 bytes to extract `means_min` and `means_max` [S2, unofficial]. Read the next 4 bytes to determine the length $L$ of the compressed stream [S2, unofficial]. Read the next $L$ bytes into a temporary binary file [S2, unofficial].
2. **GPCC Decompression**: Execute the external GPCC decoder (`tmc3`) with flags `mode=1` and `outputBinaryPly=1` [S2, unofficial]. Input is the temporary binary file; output is a reconstructed binary PLY file [S2, unofficial].
3. **Reading Voxelized Means**: Parse the binary PLY file to retrieve the voxelized means $\mathbf{v}_i$ from the x, y, z properties [S2, unofficial].
4. **Re-sorting**: Sort the decoded voxelized means by their Morton codes, as the GPCC decoder may unsort them [S2, unofficial].
5. **Devoxelization**: Convert the sorted integer voxelized means $\mathbf{v}_i$ back to float32 coordinates: $\mathbf{p}_i = \mathbf{p}_{min} + \left( \frac{\mathbf{v}_i}{2^{16} - 1} \right) \times (\mathbf{p}_{max} - \mathbf{p}_{min})$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import subprocess
import numpy as np
import os

def encode_gaussian_means(means: np.ndarray) -> bytes:
    """
    Compress 3D Gaussian means using EGGS pipeline.
    means: Nx3 array of float32 positions.
    Returns: bytes of the final file content.
    """
    # 1. Calculate bounding box
    p_min = np.min(means, axis=0)
    p_max = np.max(means, axis=0)
    range_vec = p_max - p_min
    # Handle degenerate case where range is zero
    range_vec = np.where(range_vec == 0, 1.0, range_vec)

    # 2. Voxelize
    p_norm = (means - p_min) / range_vec
    VOXEL_SCALE = 2**16 - 1
    v_int = np.round(p_norm * VOXEL_SCALE).astype(np.int32)

    # 3. Sort by Morton Code
    # Dot product of voxelized coordinates and powers of (max_coordinate + 1)
    # max_coordinate = 2^16 - 1, so base = 2^16
    base = 2**16
    morton_codes = v_int[:, 0] + v_int[:, 1] * base + v_int[:, 2] * (base ** 2)
    sort_indices = np.argsort(morton_codes)
    v_sorted = v_int[sort_indices]

    # 4. Write ASCII PLY
    ply_header = f"ply\nformat ascii 1.0\nelement vertex {len(v_sorted)}\nproperty float x\nproperty float y\nproperty float z\nend_header\n"
    ply_lines = [f"{v_sorted[i,0]} {v_sorted[i,1]} {v_sorted[i,2]}" for i in range(len(v_sorted))]
    ply_data = (ply_header + "\n".join(ply_lines) + "\n").encode('ascii')

    # 5. GPCC Encode
    with open("temp_input.ply", "wb") as f:
        f.write(ply_data)
    cmd = [
        "tmc3", "encode", "temp_input.ply", "temp_output.bin",
        "--mode", "0", "--trisoupNodeSizeLog2", "0", "--mergeDuplicatedPoints", "0",
        "--neighbourAvailBoundaryLog2", "8", "--positionQuantisationEnabled", "1",
        "--intra_pred_max_node_size_log2", "3", "--positionQuantizationScale", "1",
        "--inferredDirectCodingMode", "3", "--maxNumQtBtBeforeOt", "2",
        "--minQtbtSizeLog2", "0", "--planarEnabled", "0", "--planarModeIdcmUse", "0",
        "--cabac_bypass_stream_enabled_flag", "1"
    ]
    subprocess.run(cmd, check=True)

    with open("temp_output.bin", "rb") as f:
        compressed_data = f.read()

    # 6. Assemble File
    header = struct.pack('fff', *p_min) + struct.pack('fff', *p_max)
    length = struct.pack('I', len(compressed_data))
    final_file = header + length + compressed_data

    # Cleanup
    os.remove("temp_input.ply")
    os.remove("temp_output.bin")
    return final_file

def decode_gaussian_means(file_data: bytes) -> np.ndarray:
    """
    Decompress 3D Gaussian means from EGGS binary file.
    file_data: bytes of the file.
    Returns: Nx3 array of float32 positions.
    """
    # 1. Parse Header
    offset = 0
    p_min = np.array(struct.unpack('fff', file_data[offset:offset+12]))
    offset += 12
    p_max = np.array(struct.unpack('fff', file_data[offset:offset+12]))
    offset += 12

    # 2. Parse Length
    length = struct.unpack('I', file_data[offset:offset+4])[0]
    offset += 4
    compressed_data = file_data[offset:offset+length]

    # 3. Write Compressed Data
    with open("temp_compressed.bin", "wb") as f:
        f.write(compressed_data)

    # 4. GPCC Decode
    cmd = [
        "tmc3", "decode", "temp_compressed.bin", "temp_output.ply",
        "--mode", "1", "--outputBinaryPly", "1"
    ]
    subprocess.run(cmd, check=True)

    # 5. Read Voxelized Means from Binary PLY
    # Minimal binary PLY parser: reads header to find data type and vertex count, then reads points.
    with open("temp_output.ply", "rb") as f:
        header_lines = []
        while True:
            line = f.readline()
            header_lines.append(line)
            if line.strip() == b"end_header":
                break
        header_text = b"".join(header_lines).decode('ascii')
        # Extract vertex count
        vertex_count = int(header_text.split("element vertex ")[1].split()[0])
        # Binary PLY format: little-endian floats for x,y,z
        points = np.fromfile(f, dtype=np.float32, count=vertex_count * 3).reshape(-1, 3)
        v_decoded = np.round(points).astype(np.int32)

    # 6. Sort by Morton Code
    base = 2**16
    morton_codes = v_decoded[:, 0] + v_decoded[:, 1] * base + v_decoded[:, 2] * (base ** 2)
    sort_indices = np.argsort(morton_codes)
    v_sorted = v_decoded[sort_indices]

    # 7. Devoxelize
    range_vec = p_max - p_min
    range_vec = np.where(range_vec == 0, 1.0, range_vec)
    p_recovered = p_min + (v_sorted / (2**16 - 1)) * range_vec

    # Cleanup
    os.remove("temp_compressed.bin")
    os.remove("temp_output.ply")
    return p_recovered
```

### Edge Cases and Pitfalls
- **Unsorted Decoded Data**: The GPCC decoder does not guarantee output points are sorted by Morton code [S2, unofficial]. They must be re-sorted before devoxelization if subsequent processing relies on the original order [S2, unofficial].
- **Precision**: Voxelization uses 16-bit integers, limiting positional precision [S2, unofficial].
- **Empty Scene / Degenerate Bounding Box**: If all Gaussian means are identical, the range is zero [G]. The normalization step must handle division by zero by clamping the range to a minimum positive value [G].

## Operating characteristics
- **Quality Metrics (Compact3D on MipNeRF360)**: SSIM 0.808, PSNR 27.16, LPIPS 0.228 [S1].
- **Comparison**: Compact3D has lower SSIM (0.808) than 3DGS (0.815) and LightGaussian (0.857), and higher LPIPS (0.228) than 3DGS (0.214) and LightGaussian (0.210) [S1].
- **Strengths**: Avoids quantization of position and opacity, reducing artifacts from overlapping Gaussians [S1].
- **Limitations**: Requires external GPCC encoder/decoder (`tmc3`) [S2, unofficial]. Precision is limited by voxelization bit depth [S2, unofficial]. The exact compression ratio is not specified in the sources [S1].

## Background
- **Compact3D**: Cited as [55] in [S1]. It is a compression method that specifically avoids quantizing opacity and position to prevent visual artifacts caused by overlapping Gaussians [S1].
- **EGGS (Explicitly Granular 3D Gaussian Splatting)**: Cited in [S2, unofficial]. It uses Luma-Aware and Volume-Preserving Attribute Factorization [S2, unofficial]. The implementation relies on G-PCC for geometry compression [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
