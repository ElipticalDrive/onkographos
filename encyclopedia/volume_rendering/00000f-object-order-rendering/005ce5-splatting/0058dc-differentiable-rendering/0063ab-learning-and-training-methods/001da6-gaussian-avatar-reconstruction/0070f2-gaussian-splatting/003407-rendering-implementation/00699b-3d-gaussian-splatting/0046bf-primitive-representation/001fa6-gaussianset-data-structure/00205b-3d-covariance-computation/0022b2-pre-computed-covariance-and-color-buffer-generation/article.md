# Pre-computed Covariance and Color Buffer Generation

Also called: Covariance and Color Pre-computation, Gaussian Parameter Pre-processing

## Overview
This method is a preprocessing step that converts raw Gaussian splat parameters (position, scale, rotation, color) into pre-computed 3x3 covariance matrices and normalized float colors to accelerate rendering [S1, unofficial]. It solves the problem of per-frame matrix multiplication and color decoding by computing these values once during load [S1, unofficial].

## How it works
**Inputs:**
- `bufferData`: A byte array containing interleaved Gaussian parameters [S1, unofficial].
- `vertexCount`: The total number of Gaussians [S1, unofficial].

**Outputs:**
- `covarianceArray`: A `Float32Array` of size `6 * vertexCount` [S1, unofficial]. Stores the 6 unique elements of the symmetric covariance matrix for each Gaussian [S1, unofficial].
- `colorArray`: A `Float32Array` of size `4 * vertexCount` [S1, unofficial]. Stores normalized RGBA color values in the range [0.0, 1.0] [S1, unofficial].

**Data Structures and Layout:**
The input `bufferData` is organized in rows of 44 bytes per vertex [S1, unofficial]. The layout per vertex is:
- Bytes 0–11: Position (XYZ), 3 `Float32` values [S1, unofficial].
- Bytes 12–23: Scale (XYZ), 3 `Float32` values [S1, unofficial].
- Bytes 24–39: Rotation (IJKW), 4 `Float32` values [S1, unofficial]. The source specifies the order IJKW corresponds to x, y, z, w components of the quaternion [S1, unofficial].
- Bytes 40–43: Color (RGBA), 4 `Uint8` values [S1, unofficial].

The output buffers are tightly packed arrays of `Float32` values [S1, unofficial].

**Steps:**
1. **Initialize Buffers:** Create `covarianceBufferData` with size `CovarianceSizeBytes * vertexCount` and `colorBufferData` with size `ColorSizeBytes * vertexCount` [S1, unofficial]. Wrap these in `Float32Array` instances named `covarianceArray` and `colorArray` [S1, unofficial].
2. **Iterate Over Vertices:** For each vertex index $i$ from $0$ to `vertexCount` - 1 [S1, unofficial]:
   2.1. **Process Color:** Calculate byte offset `baseColor = 44 * i + 40` [S1, unofficial]. Read 4 `Uint8` values from `bufferData` at `baseColor` [S1, unofficial]. Normalize each value by dividing by 255.0 [S1, unofficial]. Store the normalized floats in `colorArray` at indices `4*i` to `4*i+3` [S1, unofficial].
   2.2. **Extract Scale:** Calculate float offset `baseScale = 11 * i + 3` [S1, unofficial]. Read 3 `Float32` values from `bufferData` at `baseScale` [S1, unofficial]. Construct a diagonal scale matrix $S$ [S1, unofficial].
   2.3. **Extract Rotation:** Calculate float offset `rotationBase = 11 * i + 7` [S1, unofficial]. Read 4 `Float32` values from `bufferData` at `rotationBase` [S1, unofficial]. Interpret these as quaternion components $(q_x, q_y, q_z, q_w)$ corresponding to the I, J, K, W order [S1, unofficial]. Construct a rotation matrix $R$ [G].
   2.4. **Compute Transformation Matrix $M$:** Compute $M = R \times S$ [S1, unofficial]. Let $M_{ij}$ be the element at row $i$ and column $j$ (0-indexed) of the $3 \times 3$ matrix $M$ [S1, unofficial].
   2.5. **Compute Covariance Elements:** The symmetric covariance matrix $C$ is computed as $C = M M^T$ [S1, unofficial]. The 6 unique elements are stored in `covarianceArray` at indices `6*i` to `6*i+5`:
     $$C_{00} = M_{00}^2 + M_{01}^2 + M_{02}^2$$
     $$C_{01} = M_{00}M_{10} + M_{01}M_{11} + M_{02}M_{12}$$
     $$C_{02} = M_{00}M_{20} + M_{01}M_{21} + M_{02}M_{22}$$
     $$C_{11} = M_{10}^2 + M_{11}^2 + M_{12}^2$$
     $$C_{12} = M_{10}M_{20} + M_{11}M_{21} + M_{12}M_{22}$$
     $$C_{22} = M_{20}^2 + M_{21}^2 + M_{22}^2$$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def precompute_covariance_and_color(buffer_data, vertex_count):
    """
    Pre-computes covariance matrices and normalized colors from raw splat data.
    
    Args:
        buffer_data: bytes or bytearray containing interleaved Gaussian parameters.
        vertex_count: int, number of Gaussians.
        
    Returns:
        covariance_array: list of floats, length 6 * vertex_count.
        color_array: list of floats, length 4 * vertex_count.
    """
    covariance_array = [0.0] * (6 * vertex_count)
    color_array = [0.0] * (4 * vertex_count)
    
    # Constants for layout
    SCALE_OFFSET_FLOATS = 3
    ROTATION_OFFSET_FLOATS = 7
    COLOR_OFFSET_BYTES = 40
    
    for i in range(vertex_count):
        base_float_offset = i * 11
        base_byte_offset = i * 44
        
        # 1. Process Color
        r = buffer_data[base_byte_offset + COLOR_OFFSET_BYTES]
        g = buffer_data[base_byte_offset + COLOR_OFFSET_BYTES + 1]
        b = buffer_data[base_byte_offset + COLOR_OFFSET_BYTES + 2]
        a = buffer_data[base_byte_offset + COLOR_OFFSET_BYTES + 3]
        
        color_array[4 * i] = r / 255.0
        color_array[4 * i + 1] = g / 255.0
        color_array[4 * i + 2] = b / 255.0
        color_array[4 * i + 3] = a / 255.0
        
        # 2. Extract Scale
        sx = struct.unpack_from('<f', buffer_data, base_float_offset + SCALE_OFFSET_FLOATS)[0]
        sy = struct.unpack_from('<f', buffer_data, base_float_offset + SCALE_OFFSET_FLOATS + 1)[0]
        sz = struct.unpack_from('<f', buffer_data, base_float_offset + SCALE_OFFSET_FLOATS + 2)[0]
        
        # 3. Extract Rotation
        qx = struct.unpack_from('<f', buffer_data, base_float_offset + ROTATION_OFFSET_FLOATS)[0]
        qy = struct.unpack_from('<f', buffer_data, base_float_offset + ROTATION_OFFSET_FLOATS + 1)[0]
        qz = struct.unpack_from('<f', buffer_data, base_float_offset + ROTATION_OFFSET_FLOATS + 2)[0]
        qw = struct.unpack_from('<f', buffer_data, base_float_offset + ROTATION_OFFSET_FLOATS + 3)[0]
        
        # Compute Rotation Matrix R from Quaternion (qx, qy, qz, qw) [G]
        qx2, qy2, qz2, qw2 = qx*qx, qy*qy, qz*qz, qw*qw
        xy, xz, xw, yz, yw, zw = qx*qy, qx*qz, qx*qw, qy*qz, qy*qw, qz*qw
        
        r00 = 1 - 2 * (qy2 + qz2)
        r01 = 2 * (xy - zw)
        r02 = 2 * (xz + yw)
        
        r10 = 2 * (xy + zw)
        r11 = 1 - 2 * (qx2 + qz2)
        r12 = 2 * (yz - xw)
        
        r20 = 2 * (xz - yw)
        r21 = 2 * (yz + xw)
        r22 = 1 - 2 * (qx2 + qy2)
        
        # 4. Compute M = R * S
        # Since S is diagonal, M_ij = R_ij * S_jj
        m00, m01, m02 = r00 * sx, r01 * sy, r02 * sz
        m10, m11, m12 = r10 * sx, r11 * sy, r12 * sz
        m20, m21, m22 = r20 * sx, r21 * sy, r22 * sz
        
        # 5. Compute Covariance C = M * M^T
        c00 = m00*m00 + m01*m01 + m02*m02
        c01 = m00*m10 + m01*m11 + m02*m12
        c02 = m00*m20 + m01*m21 + m02*m22
        c11 = m10*m10 + m11*m11 + m12*m12
        c12 = m10*m20 + m11*m21 + m12*m22
        c22 = m20*m20 + m21*m21 + m22*m22
        
        covariance_array[6 * i] = c00
        covariance_array[6 * i + 1] = c01
        covariance_array[6 * i + 2] = c02
        covariance_array[6 * i + 3] = c11
        covariance_array[6 * i + 4] = c12
        covariance_array[6 * i + 5] = c22

    return covariance_array, color_array
```

**Edge Cases and Pitfalls:**
- **Quaternion Order:** The source explicitly notes that the rotation data is stored as IJKW (x, y, z, w) [S1, unofficial]. Standard libraries often expect WXYZ or XYZW [G]. Misinterpreting this order will result in incorrect rotation matrices and thus incorrect covariance shapes [S1, unofficial].
- **Matrix Multiplication Order:** The covariance is derived from $M = R \times S$ [S1, unofficial]. Since $S$ is diagonal, this scales the columns of $R$ [G]. If $S$ were applied as $S \times R$, the rows would be scaled, which is incorrect for transforming a sphere into an ellipsoid defined by scale axes aligned with the rotation basis [G].
- **Precision:** The input color is `Uint8` [S1, unofficial]. Normalization by 255.0 introduces floating-point representation errors, but this is standard for color data [G]. The covariance elements are `Float32`, which may lose precision for very small or very large scales, potentially affecting rendering stability at extreme zoom levels [G].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of Gaussians [S1, unofficial]. The computation involves a fixed number of arithmetic operations per Gaussian [G].
- **Memory Complexity:** $O(N)$ [S1, unofficial]. The output buffers require $10 \times N \times 4$ bytes (6 floats for covariance + 4 floats for color) [S1, unofficial].
- **Performance:** This preprocessing moves computation from the GPU fragment/vertex shader to the CPU (or a compute shader) during asset loading [S1, unofficial]. This reduces the per-frame cost of rendering, as the shader only needs to sample the pre-computed covariance and color values [S1, unofficial].
- **Strengths:** Eliminates per-pixel or per-vertex matrix multiplication and quaternion-to-matrix conversion during rendering [S1, unofficial]. Simplifies the shader logic [G].
- **Limitations:** Requires storing additional data (covariance matrix) in memory [S1, unofficial]. If the Gaussians are dynamic (e.g., animated), this pre-computation must be repeated every frame or whenever parameters change, negating the benefit [G].

## Background
This method is part of the implementation of 3D Gaussian Splatting for real-time web rendering, specifically within the context of Three.js and A-Frame viewers [S1, unofficial]. It is based on the work of Mark Kellogg (GaussianSplats3D) and Kevin Kwok (WebGL implementation of Gaussian Splatting) [S1, unofficial]. The technique leverages the fact that the covariance matrix for a Gaussian splat is constant for a given set of scale and rotation parameters, allowing it to be pre-computed [G].

## Sources
[S1] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
