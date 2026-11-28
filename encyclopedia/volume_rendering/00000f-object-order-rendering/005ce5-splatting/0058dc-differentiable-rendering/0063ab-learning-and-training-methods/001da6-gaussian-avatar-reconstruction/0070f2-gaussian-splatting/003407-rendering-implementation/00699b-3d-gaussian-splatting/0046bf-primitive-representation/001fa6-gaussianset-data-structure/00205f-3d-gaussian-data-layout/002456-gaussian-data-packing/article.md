# Gaussian Data Packing

Also called: GaussianData::pack

## Overview
Gaussian Data Packing is a method for serializing 3D Gaussian Splatting primitives into a compact, GPU-friendly binary layout. It solves the problem of transferring large volumes of Gaussian data (position, scale, rotation, color, and precomputed covariance) from CPU memory to the GPU by packing multiple attributes into a contiguous array of 32-bit unsigned integers. It sets itself apart from sibling methods like [Gaussian Data Structure](2273-gaussian-data-structure.md) by enforcing a specific interleaved `uint32` layout optimized for direct GPU consumption, and from [GeometryState Allocation](5166-geometrystate-allocation.md) by focusing on the serialization format rather than memory management.

## How it works
**Inputs and Outputs:**
*   **Input:** A list of $n$ Gaussian primitives. Each primitive $i$ consists of:
    *   Position: $\mathbf{p} = (x, y, z)$ (float32)
    *   Scale: $\mathbf{s} = (s_x, s_y, s_z)$ (float32)
    *   Rotation: $\mathbf{q} = (w, x, y, z)$ (quaternion, float32)
    *   Color: $\mathbf{c} = (r, g, b, a)$ (float32, typically in range [0, 1])
*   **Output:**
    *   `packedData`: A `std::vector<uint32_t>` of size $8n$.
    *   `worldPositions`: A `std::vector<float>` of size $3n$, storing raw float positions.

**Data Structures:**
The `packedData` array stores two `uvec4` per Gaussian. The layout for Gaussian $i$ (indices $8i$ to $8i+7$) is:
*   `packedData[8i + 0]`: Position X (bit-cast)
*   `packedData[8i + 1]`: Position Y (bit-cast)
*   `packedData[8i + 2]`: Position Z (bit-cast)
*   `packedData[8i + 3]`: Selection flag (0)
*   `packedData[8i + 4]`: Covariance $\sigma_{xx}$ and $\sigma_{xy}$ (packed half-precision)
*   `packedData[8i + 5]`: Covariance $\sigma_{xz}$ and $\sigma_{yy}$ (packed half-precision)
*   `packedData[8i + 6]`: Covariance $\sigma_{yz}$ and $\sigma_{zz}$ (packed half-precision)
*   `packedData[8i + 7]`: Color RGBA (packed 8-bit integer)

**Steps:**
1.  Iterate $i$ from $0$ to $n-1$.
2.  Extract $\mathbf{p}, \mathbf{s}, \mathbf{q}, \mathbf{c}$ for Gaussian $i$.
3.  Store raw floats in `worldPositions`:
    $$ \text{worldPositions}[3i + 0] = x $$
    $$ \text{worldPositions}[3i + 1] = y $$
    $$ \text{worldPositions}[3i + 2] = z $$
4.  Bit-cast position components to `uint32_t` using `memcpy` (preserving IEEE 754 bit pattern):
    $$ u_x = \text{reinterpret\_cast<uint32\_t>}(x) $$
    $$ u_y = \text{reinterpret\_cast<uint32\_t>}(y) $$
    $$ u_z = \text{reinterpret\_cast<uint32\_t>}(z) $$
5.  Store $u_x, u_y, u_z$ in `packedData[8i+0]`, `packedData[8i+1]`, `packedData[8i+2]`.
6.  Store $0$ in `packedData[8i+3]` as a selection flag [S1, unofficial].
7.  Compute the 3D covariance matrix $\Sigma$. First, construct the rotation matrix $R$ from $\mathbf{q}$ and the scale matrix $S = \text{diag}(s_x, s_y, s_z)$.
    $$ M = R \cdot S $$
    The elements of $M$ are denoted $M_{row,col}$. The symmetric covariance matrix $\Sigma = M M^T$ has elements:
    $$ \sigma_{xx} = M_{0,0}^2 + M_{1,0}^2 + M_{2,0}^2 $$
    $$ \sigma_{xy} = M_{0,0}M_{0,1} + M_{1,0}M_{1,1} + M_{2,0}M_{2,1} $$
    $$ \sigma_{xz} = M_{0,0}M_{0,2} + M_{1,0}M_{1,2} + M_{2,0}M_{2,2} $$
    $$ \sigma_{yy} = M_{0,1}^2 + M_{1,1}^2 + M_{2,1}^2 $$
    $$ \sigma_{yz} = M_{0,1}M_{0,2} + M_{1,1}M_{1,2} + M_{2,1}M_{2,2} $$
    $$ \sigma_{zz} = M_{0,2}^2 + M_{1,2}^2 + M_{2,2}^2 $$
8.  Pack covariance pairs into `uint32_t` using half-precision floating-point packing (`packHalf2x16`):
    *   `packedData[8i+4]` = $\text{packHalf2x16}(\sigma_{xx}, \sigma_{xy})$
    *   `packedData[8i+5]` = $\text{packHalf2x16}(\sigma_{xz}, \sigma_{yy})$
    *   `packedData[8i+6]` = $\text{packHalf2x16}(\sigma_{yz}, \sigma_{zz})$
9.  Pack color into `uint32_t`:
    $$ \text{packedColor} = \text{uint32}(r \cdot 255) \mid (\text{uint32}(g \cdot 255) \ll 8) \mid (\text{uint32}(b \cdot 255) \ll 16) \mid (\text{uint32}(a \cdot 255) \ll 24) $$
    Store `packedColor` in `packedData[8i+7]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_gaussians(gaussians):
    """
    gaussians: list of dicts/objects with keys 'pos' (vec3), 'scale' (vec3),
               'rot' (quat), 'color' (vec4 with components 0.0-1.0).
    Returns: packed_data (list of uint32), world_positions (list of float)
    """
    n = len(gaussians)
    packed_data = [0] * (n * 8)
    world_positions = [0.0] * (n * 3)

    for i in range(n):
        g = gaussians[i]
        x, y, z = g['pos']
        sx, sy, sz = g['scale']
        qw, qx, qy, qz = g['rot']
        r, g_val, b, a = g['color']

        # 1. Store raw positions
        world_positions[3*i + 0] = x
        world_positions[3*i + 1] = y
        world_positions[3*i + 2] = z

        # 2. Bit-cast positions to uint32
        ux = float_to_uint32(x)
        uy = float_to_uint32(y)
        uz = float_to_uint32(z)

        packed_data[8*i + 0] = ux
        packed_data[8*i + 1] = uy
        packed_data[8*i + 2] = uz
        packed_data[8*i + 3] = 0  # Selection flag

        # 3. Compute Covariance
        # Convert quaternion to rotation matrix R (3x3)
        R = quat_to_mat3(qw, qx, qy, qz)
        
        # Scale matrix S (diagonal)
        # M = R * S
        M00 = R[0][0] * sx
        M01 = R[0][1] * sy
        M02 = R[0][2] * sz
        M10 = R[1][0] * sx
        M11 = R[1][1] * sy
        M12 = R[1][2] * sz
        M20 = R[2][0] * sx
        M21 = R[2][1] * sy
        M22 = R[2][2] * sz

        # Sigma elements (Sigma = M * M^T)
        sigma_xx = M00*M00 + M10*M10 + M20*M20
        sigma_xy = M00*M01 + M10*M11 + M20*M21
        sigma_xz = M00*M02 + M10*M12 + M20*M22
        sigma_yy = M01*M01 + M11*M11 + M21*M21
        sigma_yz = M01*M02 + M11*M12 + M21*M22
        sigma_zz = M02*M02 + M12*M12 + M22*M22

        # Pack covariance into half-precision uint32 pairs
        packed_data[8*i + 4] = pack_half2x16(sigma_xx, sigma_xy)
        packed_data[8*i + 5] = pack_half2x16(sigma_xz, sigma_yy)
        packed_data[8*i + 6] = pack_half2x16(sigma_yz, sigma_zz)

        # 4. Pack color
        r_int = int(round(r * 255)) & 0xFF
        g_int = int(round(g_val * 255)) & 0xFF
        b_int = int(round(b * 255)) & 0xFF
        a_int = int(round(a * 255)) & 0xFF
        
        packed_color = r_int | (g_int << 8) | (b_int << 16) | (a_int << 24)
        packed_data[8*i + 7] = packed_color

    return packed_data, world_positions

def float_to_uint32(f):
    """Reinterpret bits of float32 as uint32."""
    import struct
    return struct.unpack('I', struct.pack('f', f))[0]

def pack_half2x16(v1, v2):
    """Pack two float32 values into a single uint32 using half-precision."""
    import struct
    # Convert to half-precision (2 bytes each)
    h1 = struct.pack('e', v1) # 'e' is half precision in struct
    h2 = struct.pack('e', v2)
    # Combine into uint32 (little-endian assumed by GLSL packHalf2x16 behavior usually)
    # GLSL packHalf2x16 packs the first float into the lower 16 bits and second into upper 16 bits
    return struct.unpack('I', h1 + h2)[0]
```

**Parameters and Edge Cases:**
*   **Precision:** Position uses full float32 precision (via bit-cast). Covariance uses half-precision (float16), which may introduce quantization error for very large or small scales. Color uses 8-bit integer per channel.
*   **Boundary Conditions:** The method assumes valid input quaternions (normalized). If `rot` is not normalized, the resulting rotation matrix will be incorrect.
*   **Color Range:** Color components are expected in $[0, 1]$. Values outside this range will clamp or wrap incorrectly when multiplied by 255 and cast to uint8.

## Subtopics

### [Gaussian Splat Texture Packing](3283-gaussian-splat-texture-packing.md)
Stores packed data into a 2D texture array (`GL_RGBA32UI`) for GPU rendering. Uses fixed-width 2048-pixel texture with row-major layout.

### [SPZ Packed Format](3926-spz-packed-format.md)
Binary storage format for low-precision, non-interleaved data with ZSTD compression. Designed for web/browser loading.

### [SPZ Packed Gaussian Unpacking](3927-spz-packed-gaussian-unpacking.md)
Deserializes SPZ format back to raw floats. Inverts quantization and compression.

### [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md)
Lossless compression using 8/16-bit PNG images. Leverages spatial locality and Morton code sorting.

### [GLTF/GLB Buffer & Accessor Packing](20588-gltf-glb-buffer-accessor-packing.md)
Packs attributes into GLTF buffer views with 4-byte alignment.

## Operating characteristics
*   **Time Complexity:** $O(n)$, where $n$ is the number of Gaussians. Each Gaussian requires constant-time matrix operations and packing.
*   **Memory Complexity:** $O(n)$. `packedData` uses $32n$ bytes. `worldPositions` uses $12n$ bytes. Total $44n$ bytes.
*   **Performance:** The method is CPU-bound during packing. The use of half-precision for covariance reduces memory bandwidth compared to full float32, but requires conversion. Bit-casting positions avoids precision loss during transfer to GPU shaders that expect integer position formats.
*   **Strengths:** Compact representation for GPU consumption. Direct mapping to GLSL `uvec4` and `packHalf2x16`.
*   **Limitations:** Covariance precision is limited to 16-bit. Color is limited to 8-bit per channel. Not suitable for high-precision scientific visualization where float32 covariance is required.

## Background
Originated in the implementation of 3D Gaussian Splatting rendering pipelines, specifically in the `gsplat-opengl` repository [S1, unofficial]. It addresses the need for efficient data transfer between CPU-side Gaussian representations and GPU-side rendering shaders.

## Sources
[S1] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
