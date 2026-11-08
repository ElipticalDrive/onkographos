# Gaussian Data Packing

Also called: GaussianData::pack

## Overview
Gaussian Data Packing serializes 3D Gaussian Splat attributes into a fixed-width binary buffer to facilitate efficient GPU-side access during rasterization [S1, unofficial]. It differs from sibling methods like [SPZ Packed Format](3926-spz-packed-format.md) by using an interleaved, texture-friendly layout with a fixed width of 8 `uint32_t` per Gaussian, rather than parallel buffers or image-based compression [S1, unofficial].

## How it works
**Inputs:** A set of $N$ 3D Gaussians, each defined by position $\mathbf{p} = (x, y, z)$, scale $\mathbf{s} = (s_x, s_y, s_z)$, rotation quaternion $\mathbf{q} = (w, x, y, z)$, and color $\mathbf{c} = (r, g, b, a)$ [S1, unofficial].

**Outputs:** 
- `packedData`: A `std::vector<uint32_t>` of size $N \times 8$, storing two `uvec4` per Gaussian [S1, unofficial].
- `worldPositions`: A `std::vector<float>` of size $N \times 3$, storing raw float positions [S1, unofficial].

**Data Structures:** For the $i$-th Gaussian ($0 \le i < N$), `packedData` stores 8 `uint32_t` values: indices $0$–$2$ hold the bit-reinterpretations of $x, y, z$; index $3$ is a selection flag set to $0$; indices $4$–$6$ hold half-precision packed covariance elements; index $7$ holds the packed RGBA color [S1, unofficial].

**Steps:**
1. Initialize `packedData` with size $8N$ and `worldPositions` with size $3N$ [S1, unofficial].
2. Loop over each Gaussian $i$ from $0$ to $N-1$:
   a. Extract $\mathbf{p}, \mathbf{s}, \mathbf{q}, \mathbf{c}$ [S1, unofficial].
   b. Store raw floats in `worldPositions`: `worldPositions[3i] = x`, `worldPositions[3i+1] = y`, `worldPositions[3i+2] = z` [S1, unofficial].
   c. Reinterpret $x, y, z$ as `uint32_t` via `std::memcpy` (or equivalent bit-cast) and store in `packedData[i*8+0]`, `packedData[i*8+1]`, `packedData[i*8+2]` [S1, unofficial].
   d. Set `packedData[i*8+3] = 0` [S1, unofficial].
   e. Convert $\mathbf{q}$ to a $3 \times 3$ rotation matrix $\mathbf{R}$ [S1, unofficial].
   f. Create a diagonal scale matrix $\mathbf{S} = \text{diag}(s_x, s_y, s_z)$ [S1, unofficial].
   g. Compute $\mathbf{M} = \mathbf{R} \mathbf{S}$ [S1, unofficial].
   h. Compute symmetric covariance elements $\sigma_{jk}$:
      $$ \sigma_{xx} = M_{00}^2 + M_{10}^2 + M_{20}^2 $$
      $$ \sigma_{xy} = M_{00}M_{01} + M_{10}M_{11} + M_{20}M_{21} $$
      $$ \sigma_{xz} = M_{00}M_{02} + M_{10}M_{12} + M_{20}M_{22} $$
      $$ \sigma_{yy} = M_{01}^2 + M_{11}^2 + M_{21}^2 $$
      $$ \sigma_{yz} = M_{01}M_{02} + M_{11}M_{12} + M_{21}M_{22} $$
      $$ \sigma_{zz} = M_{02}^2 + M_{12}^2 + M_{22}^2 $$ [S1, unofficial].
   i. Pack $\sigma_{xx}, \sigma_{xy}$ into `packedData[i*8+4]`, $\sigma_{xz}, \sigma_{yy}$ into `packedData[i*8+5]`, and $\sigma_{yz}, \sigma_{zz}$ into `packedData[i*8+6]` using `packHalf2x16` [S1, unofficial].
   j. Pack color into `packedData[i*8+7]`: `packedColor = (r & 0xFF) | ((g & 0xFF) << 8) | ((b & 0xFF) << 16) | ((a & 0xFF) << 24)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def pack_half2x16(x: float, y: float) -> int:
    """Pack two IEEE 754 half-precision floats into a 32-bit integer."""
    x_bytes = struct.pack('e', x)
    y_bytes = struct.pack('e', y)
    return struct.unpack('<I', x_bytes + y_bytes)[0]

def gaussian_data_packing(gaussians: list) -> tuple:
    """
    Pack Gaussian data into the specified format.
    
    Args:
        gaussians: List of dicts with keys 'pos' (x,y,z), 'scale' (x,y,z), 
                   'rot' (w,x,y,z), 'color' (r,g,b,a).
    
    Returns:
        packed_data: List of uint32_t
        world_positions: List of float
    """
    n = len(gaussians)
    packed_data = [0] * (n * 8)
    world_positions = [0.0] * (n * 3)
    
    for i in range(n):
        g = gaussians[i]
        x, y, z = g['pos']
        sx, sy, sz = g['scale']
        w, qx, qy, qz = g['rot']
        r, gc, b, a = g['color']
        
        world_positions[3*i] = x
        world_positions[3*i+1] = y
        world_positions[3*i+2] = z
        
        packed_data[8*i] = struct.unpack('I', struct.pack('f', x))[0]
        packed_data[8*i+1] = struct.unpack('I', struct.pack('f', y))[0]
        packed_data[8*i+2] = struct.unpack('I', struct.pack('f', z))[0]
        
        packed_data[8*i+3] = 0
        
        # Convert quaternion to rotation matrix R
        norm = (w*w + qx*qx + qy*qy + qz*qz) ** 0.5
        w, qx, qy, qz = w/norm, qx/norm, qy/norm, qz/norm
        R = [
            [1 - 2*(qy*qy + qz*qz), 2*(qx*qy - w*qz), 2*(qx*qz + w*qy)],
            [2*(qx*qy + w*qz), 1 - 2*(qx*qx + qz*qz), 2*(qy*qz - w*qx)],
            [2*(qx*qz - w*qy), 2*(qy*qz + w*qx), 1 - 2*(qx*qx + qy*qy)]
        ]
        
        # M = R * S
        M = [
            [R[0][0]*sx, R[0][1]*sy, R[0][2]*sz],
            [R[1][0]*sx, R[1][1]*sy, R[1][2]*sz],
            [R[2][0]*sx, R[2][1]*sy, R[2][2]*sz]
        ]
        
        sigma_xx = M[0][0]**2 + M[1][0]**2 + M[2][0]**2
        sigma_xy = M[0][0]*M[0][1] + M[1][0]*M[1][1] + M[2][0]*M[2][1]
        sigma_xz = M[0][0]*M[0][2] + M[1][0]*M[1][2] + M[2][0]*M[2][2]
        sigma_yy = M[0][1]**2 + M[1][1]**2 + M[2][1]**2
        sigma_yz = M[0][1]*M[0][2] + M[1][1]*M[1][2] + M[2][1]*M[2][2]
        sigma_zz = M[0][2]**2 + M[1][2]**2 + M[2][2]**2
        
        packed_data[8*i+4] = pack_half2x16(sigma_xx, sigma_xy)
        packed_data[8*i+5] = pack_half2x16(sigma_xz, sigma_yy)
        packed_data[8*i+6] = pack_half2x16(sigma_yz, sigma_zz)
        
        packed_color = (int(r) & 0xFF) | ((int(gc) & 0xFF) << 8) | ((int(b) & 0xFF) << 16) | ((int(a) & 0xFF) << 24)
        packed_data[8*i+7] = packed_color
        
    return packed_data, world_positions
```

**Edge Cases and Pitfalls:**
- **Quaternion Normalization:** The input quaternion must be normalized before matrix conversion to ensure valid rotation [G].
- **Half-Precision Range:** Covariance elements stored as half-floats may underflow for very small values or overflow for very large values [G].

## Subtopics

### [Gaussian Splat Texture Packing](3283-gaussian-splat-texture-packing.md)
Gaussian Splat Texture Packing stores 3D Gaussian Splat attributes in a 2D OpenGL texture to facilitate efficient GPU-side access during rasterization. It solves the problem of transferring large sets of Gaussian parameters from CPU memory to the GPU by packing 8-byte chunks of Gaussian data into a `GL_RGBA32UI` texture. It differs from sibling methods like [SPZ Packed Format](3926-spz-packed-format.md) by using a specific row-major texture layout with a fixed width of 1024 Gaussians per row, whereas [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md) uses lossy image compression.

### [SPZ Packed Format](3926-spz-packed-format.md)
SPZ Packed Format is a binary storage format for 3D Gaussian splats designed for low-precision, non-interleaved memory layouts with ZSTD compression support. It solves the problem of efficient serialization and transmission of large-scale Gaussian datasets by separating attributes into parallel buffers, allowing for optimized entropy coding (e.g., g-zipping) and variable spherical harmonics (SH) degrees up to 4. It is distinguished from interleaved formats by its parallel vector storage and from PNG-based compression by its native binary structure and ZSTD integration.

### [SPZ Packed Gaussian Unpacking](3927-spz-packed-gaussian-unpacking.md)
SPZ Packed Gaussian Unpacking decodes compressed Gaussian Splat attributes (positions, scales, rotations, alphas, colors, and spherical harmonics) from the SPZ file format into floating-point arrays, handling coordinate system conversion and quantization inversion. It differs from sibling methods like PNG-based compression by using a binary layout with ZSTD compression (v4) or gzip (v1-3) and specific fixed-point/quantized encodings for each attribute type.

### [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md)
A compression technique for trained 3D Gaussian splats that quantizes parameters to 8-bit or 16-bit and stores them as lossless PNG images, optionally using spatial sorting (Morton code) to improve compression ratios. It serves as a port of the PyTorch `gsplat` compression module to MLX. It achieves approximately 10x compression on standard 3DGS models. Unlike the PyTorch upstream which uses `torchpq` for k-means clustering of higher-order spherical harmonics, this method falls back to NPZ for higher-order bands due to library availability constraints.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of Gaussians [S1, unofficial].
- **Memory Complexity:** $O(N)$ for `packedData` ($8N$ `uint32_t`) and `worldPositions` ($3N$ `float`) [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear scaling with the number of Gaussians [S1, unofficial].
- **Strengths:** Compact fixed-width layout, direct GPU memory mapping, simple structure [S1, unofficial].
- **Limitations:** Covariance precision limited to half-float, requires CPU-side matrix computation [S1, unofficial].
- **When to use:** Efficient CPU-to-GPU transfer for rasterization pipelines [S1, unofficial].
- **When not to use:** When disk storage compression or variable-length encoding is required [S1, unofficial].

## Background
The method is described in [S1, unofficial], which implements CPU-side packing of Gaussian attributes for transfer to OpenGL textures.

## Sources
[S1] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
