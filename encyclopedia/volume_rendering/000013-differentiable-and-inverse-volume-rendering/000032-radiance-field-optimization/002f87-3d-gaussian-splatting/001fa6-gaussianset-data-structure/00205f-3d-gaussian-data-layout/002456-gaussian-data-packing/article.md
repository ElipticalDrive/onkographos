# Gaussian Data Packing

Also called: GaussianData::pack

## Overview
Gaussian Data Packing is a method for compacting 3D Gaussian Splatting primitives into a dense, GPU-friendly binary buffer for efficient rendering and sorting. It solves the problem of memory alignment and bandwidth efficiency by interleaving position, covariance, and color data into a fixed-size `uint32_t` array, while storing raw floating-point positions in a separate parallel buffer. It is distinguished from its sibling [Gaussian Data Structure](2273-gaussian-data-structure.md) by its specific bit-packing strategy for covariance matrices and its separation of position data into integer-reinterpreted and raw float formats.

## How it works
**Inputs:**
- A list of $n$ Gaussian primitives. Each primitive $i$ has:
  - Position $\mathbf{p}_i = (x, y, z)$ as `float`.
  - Scale $\mathbf{s}_i = (s_x, s_y, s_z)$ as `float`.
  - Rotation quaternion $\mathbf{q}_i = (w, x, y, z)$ as `float`.
  - Color $\mathbf{c}_i = (r, g, b, a)$ as `float` in range $[0, 1]$.

**Outputs:**
- `packedData`: A `std::vector<uint32_t>` of size $8n$.
- `worldPositions`: A `std::vector<float>` of size $3n$.

**Data Structures:**
The `packedData` buffer stores two `uvec4` per Gaussian. The layout for Gaussian $i$ (indices $8i$ to $8i+7$) is:
- `packedData[8i+0]`: Position X (bit-cast `float` to `uint32_t`).
- `packedData[8i+1]`: Position Y (bit-cast `float` to `uint32_t`).
- `packedData[8i+2]`: Position Z (bit-cast `float` to `uint32_t`).
- `packedData[8i+3]`: Selection flag (constant `0`).
- `packedData[8i+4]`: Covariance $\sigma_{xx}$ (16-bit) and $\sigma_{xy}$ (16-bit).
- `packedData[8i+5]`: Covariance $\sigma_{xz}$ (16-bit) and $\sigma_{yy}$ (16-bit).
- `packedData[8i+6]`: Covariance $\sigma_{yz}$ (16-bit) and $\sigma_{zz}$ (16-bit).
- `packedData[8i+7]`: Color RGBA (8-bit per channel, packed into `uint32_t`).

The `worldPositions` buffer stores the raw floating-point coordinates:
- `worldPositions[3i+0]`: $x$
- `worldPositions[3i+1]`: $y$
- `worldPositions[3i+2]`: $z$

**Steps:**

1. **Initialize Buffers:**
   Allocate `packedData` with size $8n$ and `worldPositions` with size $3n$.

2. **Iterate over Gaussians:**
   For each Gaussian $i$ from $0$ to $n-1$:

3. **Pack Position:**
   - Extract $x, y, z$ from $\mathbf{p}_i$.
   - Store raw floats in `worldPositions`:
     $$ \text{worldPositions}[3i+0] = x $$
     $$ \text{worldPositions}[3i+1] = y $$
     $$ \text{worldPositions}[3i+2] = z $$
   - Reinterpret the bit pattern of each float component as a `uint32_t` using `std::memcpy` or equivalent bit-cast. Let $u_x, u_y, u_z$ be these `uint32_t` values.
   - Store in `packedData`:
     $$ \text{packedData}[8i+0] = u_x $$
     $$ \text{packedData}[8i+1] = u_y $$
     $$ \text{packedData}[8i+2] = u_z $$

4. **Store Selection Flag:**
   - Set `packedData[8i+3] = 0`. This acts as a placeholder or selection flag [S1, unofficial].

5. **Compute Covariance Matrix:**
   - Convert rotation quaternion $\mathbf{q}_i$ to a $3 \times 3$ rotation matrix $\mathbf{R}$ [S1, unofficial].
   - Create a diagonal scale matrix $\mathbf{S} = \text{diag}(s_x, s_y, s_z)$.
   - Compute the transformation matrix $\mathbf{M} = \mathbf{R} \mathbf{S}$.
   - Compute the symmetric $3 \times 3$ covariance matrix $\mathbf{\Sigma} = \mathbf{M}^T \mathbf{M}$. The elements are:
     $$ \sigma_{xx} = M_{00}^2 + M_{10}^2 + M_{20}^2 $$
     $$ \sigma_{xy} = M_{00}M_{01} + M_{10}M_{11} + M_{20}M_{21} $$
     $$ \sigma_{xz} = M_{00}M_{02} + M_{10}M_{12} + M_{20}M_{22} $$
     $$ \sigma_{yy} = M_{01}^2 + M_{11}^2 + M_{21}^2 $$
     $$ \sigma_{yz} = M_{01}M_{02} + M_{11}M_{12} + M_{21}M_{22} $$
     $$ \sigma_{zz} = M_{02}^2 + M_{12}^2 + M_{22}^2 $$
   - Store these six values in an array $\sigma[0..5]$ corresponding to $[\sigma_{xx}, \sigma_{xy}, \sigma_{xz}, \sigma_{yy}, \sigma_{yz}, \sigma_{zz}]$ [S1, unofficial].

6. **Pack Covariance:**
   - Pack $\sigma_{xx}$ and $\sigma_{xy}$ into a single `uint32_t` using 16-bit half-precision packing (e.g., `packHalf2x16`). The lower 16 bits contain $\sigma_{xx}$ and the upper 16 bits contain $\sigma_{xy}$ (or vice versa, depending on the specific `packHalf2x16` implementation; standard GLSL `packHalf2x16` packs the first component in the lower 16 bits).
     $$ \text{packedData}[8i+4] = \text{packHalf2x16}(\sigma_{xx}, \sigma_{xy}) $$
   - Pack $\sigma_{xz}$ and $\sigma_{yy}$:
     $$ \text{packedData}[8i+5] = \text{packHalf2x16}(\sigma_{xz}, \sigma_{yy}) $$
   - Pack $\sigma_{yz}$ and $\sigma_{zz}$:
     $$ \text{packedData}[8i+6] = \text{packHalf2x16}(\sigma_{yz}, \sigma_{zz}) $$

7. **Pack Color:**
   - Convert color components $r, g, b, a$ from float $[0, 1]$ to 8-bit integers $[0, 255]$.
   - Pack into a `uint32_t` with R in the least significant byte and A in the most significant byte:
     $$ \text{packedColor} = (r \ll 0) \ | \ (g \ll 8) \ | \ (b \ll 16) \ | \ (a \ll 24) $$
   - Store in `packedData`:
     $$ \text{packedData}[8i+7] = \text{packedColor} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import math

def pack_half_2x16(x: float, y: float) -> int:
    """Pack two 16-bit floats into a single 32-bit integer."""
    # Convert float to 16-bit half-precision bits
    # Note: This is a simplified representation. In C++, glm::packHalf2x16 handles this.
    # Here we assume a helper that returns uint16_t bits.
    x_bits = float_to_half_bits(x)
    y_bits = float_to_half_bits(y)
    return (y_bits << 16) | x_bits

def float_to_half_bits(f: float) -> int:
    """Convert a float to 16-bit half-precision integer representation."""
    # Using struct to pack as IEEE 754 half precision
    # Note: Python's struct doesn't directly support 'e' (half) in all versions/platforms for packing to int.
    # A robust implementation would use a library or bit manipulation.
    # For the purpose of this article, we assume a valid bit-cast exists.
    # Standard GLSL packHalf2x16 packs the first argument in the lower 16 bits.
    return 0 # Placeholder for actual half-precision conversion logic [G]

def gaussian_data_packing(gaussians: list) -> tuple:
    """
    Pack Gaussian data into packedData and worldPositions buffers.
    
    Args:
        gaussians: List of dicts with keys 'pos' (vec3), 'scale' (vec3), 
                   'rot' (quat), 'color' (vec4).
    
    Returns:
        packedData: list of uint32
        worldPositions: list of float
    """
    n = len(gaussians)
    packed_data = [0] * (8 * n)
    world_positions = [0.0] * (3 * n)
    
    for i in range(n):
        g = gaussians[i]
        pos = g['pos']
        scale = g['scale']
        rot = g['rot']
        color = g['color']
        
        # 1. Store raw positions
        world_positions[3*i + 0] = pos.x
        world_positions[3*i + 1] = pos.y
        world_positions[3*i + 2] = pos.z
        
        # 2. Bit-cast positions to uint32
        ux = float_to_uint32(pos.x)
        uy = float_to_uint32(pos.y)
        uz = float_to_uint32(pos.z)
        
        packed_data[8*i + 0] = ux
        packed_data[8*i + 1] = uy
        packed_data[8*i + 2] = uz
        
        # 3. Selection flag
        packed_data[8*i + 3] = 0
        
        # 4. Compute Covariance
        # Convert quaternion to rotation matrix R
        R = quat_to_mat3(rot)
        
        # Scale matrix S
        S = [
            [scale.x, 0, 0],
            [0, scale.y, 0],
            [0, 0, scale.z]
        ]
        
        # M = R * S
        M = mat3_mul(R, S)
        
        # Sigma = M^T * M
        # M is 3x3. M^T is transpose.
        # Sigma_ij = sum_k (M_ki * M_kj)
        
        sigma_xx = M[0][0]**2 + M[1][0]**2 + M[2][0]**2
        sigma_xy = M[0][0]*M[0][1] + M[1][0]*M[1][1] + M[2][0]*M[2][1]
        sigma_xz = M[0][0]*M[0][2] + M[1][0]*M[1][2] + M[2][0]*M[2][2]
        sigma_yy = M[0][1]**2 + M[1][1]**2 + M[2][1]**2
        sigma_yz = M[0][1]*M[0][2] + M[1][1]*M[1][2] + M[2][1]*M[2][2]
        sigma_zz = M[0][2]**2 + M[1][2]**2 + M[2][2]**2
        
        # 5. Pack Covariance
        packed_data[8*i + 4] = pack_half_2x16(sigma_xx, sigma_xy)
        packed_data[8*i + 5] = pack_half_2x16(sigma_xz, sigma_yy)
        packed_data[8*i + 6] = pack_half_2x16(sigma_yz, sigma_zz)
        
        # 6. Pack Color
        r_int = int(color.r * 255) & 0xFF
        g_int = int(color.g * 255) & 0xFF
        b_int = int(color.b * 255) & 0xFF
        a_int = int(color.a * 255) & 0xFF
        
        packed_color = (r_int) | (g_int << 8) | (b_int << 16) | (a_int << 24)
        packed_data[8*i + 7] = packed_color
        
    return packed_data, world_positions

def float_to_uint32(f: float) -> int:
    """Bit-cast float to uint32."""
    return struct.unpack('I', struct.pack('f', f))[0]

def quat_to_mat3(q: dict) -> list:
    """Convert quaternion (w,x,y,z) to 3x3 rotation matrix."""
    w, x, y, z = q['w'], q['x'], q['y'], q['z']
    # Standard quaternion to rotation matrix conversion
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    
    return [
        [1 - 2*(yy + zz), 2*(xy - wz), 2*(xz + wy)],
        [2*(xy + wz), 1 - 2*(xx + zz), 2*(yz - wx)],
        [2*(xz - wy), 2*(yz + wx), 1 - 2*(xx + yy)]
    ]

def mat3_mul(A: list, B: list) -> list:
    """Multiply two 3x3 matrices."""
    C = [[0,0,0], [0,0,0], [0,0,0]]
    for i in range(3):
        for j in range(3):
            for k in range(3):
                C[i][j] += A[i][k] * B[k][j]
    return C
```

**Parameters:**
- $n$: Number of Gaussians.
- Texture dimensions (for variant): Width 2048, Height $\lceil n / 1024 \rceil$.

**Edge Cases and Pitfalls:**
- **Precision:** Covariance values are stored as 16-bit floats. Large covariance values may lose precision or overflow. [S1, unofficial]
- **Bit-Casting:** The position bit-cast relies on IEEE 754 representation. Endianness must be consistent between packing and unpacking. [S1, unofficial]
- **Covariance Symmetry:** The covariance matrix is symmetric. Only the upper triangle (plus diagonal) is stored, mapped to $\sigma_{xx}, \sigma_{xy}, \sigma_{xz}, \sigma_{yy}, \sigma_{yz}, \sigma_{zz}$. [S1, unofficial]

## Subtopics

### [SPZ Packed Format](3926-spz-packed-format.md)
SPZ Packed Format is a binary storage format for 3D Gaussian splats designed for low-precision, non-interleaved memory layouts with ZSTD compression support. It solves the problem of efficient serialization and transmission of large-scale Gaussian datasets by separating attributes into parallel buffers, allowing for optimized entropy coding (e.g., g-zipping) and variable spherical harmonics (SH) degrees up to 4. It is distinguished from interleaved formats by its parallel vector storage and from PNG-based compression by its native binary structure and ZSTD integration. See [SPZ Packed Format](3926-spz-packed-format.md).

### [SPZ Packed Gaussian Unpacking](3927-spz-packed-gaussian-unpacking.md)
SPZ Packed Gaussian Unpacking decodes compressed Gaussian Splat attributes (positions, scales, rotations, alphas, colors, and spherical harmonics) from the SPZ file format into floating-point arrays, handling coordinate system conversion and quantization inversion. It differs from sibling methods like PNG-based compression by using a binary layout with ZSTD compression (v4) or gzip (v1-3) and specific fixed-point/quantized encodings for each attribute type. See [SPZ Packed Gaussian Unpacking](3927-spz-packed-gaussian-unpacking.md).

### [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md)
A compression technique for trained 3D Gaussian splats that quantizes parameters to 8-bit or 16-bit and stores them as lossless PNG images, optionally using spatial sorting (Morton code) to improve compression ratios. It serves as a port of the PyTorch `gsplat` compression module to MLX. It achieves approximately 10x compression on standard 3DGS models. Unlike the PyTorch upstream which uses `torchpq` for k-means clustering of higher-order spherical harmonics, this method falls back to NPZ for higher-order bands due to library availability constraints. See [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md).

### Gaussian Splat Texture Packing
A variant of Gaussian Data Packing that maps the packed `uint32_t` data into a 2D OpenGL texture for GPU access. It maps each Gaussian to two horizontal columns in a texture of width 2048. The x-coordinate is `(index & 0x3ff) << 1` and the y-coordinate is `index >> 10`. The texture format is `GL_RGBA32UI` with `GL_RGBA_INTEGER` type and `GL_UNSIGNED_INT` data type. Filters are `GL_NEAREST` for min/mag, and wrap mode is `GL_CLAMP_TO_EDGE` for both S and T axes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(n)$ to pack $n$ Gaussians, as each Gaussian is processed in constant time.
- **Memory Complexity:** $O(n)$ for `packedData` ($8 \times 4 = 32$ bytes per Gaussian) and $O(n)$ for `worldPositions` ($3 \times 4 = 12$ bytes per Gaussian). Total 44 bytes per Gaussian.
- **Performance:** The packing is CPU-bound and typically fast for moderate $n$. The resulting buffer is optimized for GPU texture uploads and direct memory access.
- **Strengths:** Compact representation, efficient GPU memory usage, fast packing/unpacking.
- **Limitations:** Loss of precision in covariance storage (16-bit). Position data is stored twice (bit-cast and raw float), increasing memory usage.
- **When to use:** When rendering 3D Gaussian Splats on GPU with OpenGL/Vulkan, especially when sorting and rasterization require fast access to covariance and position data.
- **When not to use:** When high-precision covariance is required, or when memory bandwidth is not a bottleneck and simpler data structures are preferred.

## Background
Gaussian Data Packing is part of the 3D Gaussian Splatting rendering pipeline, specifically designed for efficient GPU-based rendering. It is implemented in the `gsplat` library, an unofficial OpenGL implementation of 3D Gaussian Splatting [S1, unofficial]. The method leverages GLSL packing functions (`packHalf2x16`) and bit-casting to optimize data transfer and storage.

## Sources
[S1] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
