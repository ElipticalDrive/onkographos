# Cubic Normal Encoding

Also called: AttributeEncoder, Cube Face Normal Encoding, Cube Normal Encoding, EncodeNormal, Normal Encoding into 32-bit Integer, Normal Vector Encoding

## Overview
Cubic Normal Encoding maps a 3D unit normal vector to a single 32-bit integer by projecting the vector onto the face of a cube with the largest absolute component [S1, unofficial]. It preserves higher precision for normals aligned with the axes compared to spherical mappings, making it suitable for sparse voxel octree storage where memory efficiency is critical [S1, unofficial]. Unlike Octahedral Normal Encoding, which wraps the sphere onto an octahedron, this method uses a cubic projection, resulting in a deterministic tie-breaking rule when components are equal [S1, unofficial].

## How it works
**Inputs:** A 3D normal vector $\mathbf{n} = (x, y, z)$ where $\|\mathbf{n}\| \approx 1$.
**Outputs:** A 32-bit unsigned integer `encoded`.

**Data Structure:**
The 32-bit integer is laid out as follows:
- Bit 22: Sign of the dominant axis (0 for negative, 1 for positive).
- Bits 21-20: Face index (0 for X, 1 for Y, 2 for Z).
- Bits 19-10: 10-bit unsigned normalized coordinate for the second largest component (1024 levels).
- Bits 9-0: 10-bit unsigned normalized coordinate for the third largest component (1024 levels).

**Steps:**
1. Compute absolute values: $a_x = |x|, a_y = |y|, a_z = |z|$.
2. Find the maximum absolute component: $m = \max(a_x, a_y, a_z)$.
3. Normalize the vector relative to the maximum component: $\mathbf{c} = \mathbf{n} / m$. This results in one component being $\pm 1$ and the others being in $[-1, 1]$.
4. Convert $\mathbf{c}$ to unsigned normalized range $[0, 1]$: $\mathbf{c}_{unorm} = \mathbf{c} \cdot 0.5 + 0.5$.
5. Initialize `encoded = 0`.
6. Determine the dominant axis and encode:
   - If $|x| == m$:
     - Set bit 22 to 1 if $x > 0$, else 0.
     - Set bits 21-20 to 0 (Face X).
     - Set bits 19-10 to $\lfloor 1023 \cdot c_{unorm, y} \rfloor$.
     - Set bits 9-0 to $\lfloor 1023 \cdot c_{unorm, z} \rfloor$.
   - Else if $|y| == m$:
     - Set bit 22 to 1 if $y > 0$, else 0.
     - Set bits 21-20 to 1 (Face Y).
     - Set bits 19-10 to $\lfloor 1023 \cdot c_{unorm, x} \rfloor$.
     - Set bits 9-0 to $\lfloor 1023 \cdot c_{unorm, z} \rfloor$.
   - Else if $|z| == m$:
     - Set bit 22 to 1 if $z > 0$, else 0.
     - Set bits 21-20 to 2 (Face Z).
     - Set bits 19-10 to $\lfloor 1023 \cdot c_{unorm, x} \rfloor$.
     - Set bits 9-0 to $\lfloor 1023 \cdot c_{unorm, y} \rfloor$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_cubic_normal(normal: tuple[float, float, float]) -> int:
    x, y, z = normal
    ax, ay, az = abs(x), abs(y), abs(z)
    m = max(ax, ay, az)
    
    # Avoid division by zero if normal is zero vector [G]
    if m == 0:
        return 0
        
    # Normalize relative to max component
    cx, cy, cz = x / m, y / m, z / m
    
    # Convert to [0, 1] range
    cx_unorm = cx * 0.5 + 0.5
    cy_unorm = cy * 0.5 + 0.5
    cz_unorm = cz * 0.5 + 0.5
    
    encoded = 0
    
    if ax == m:
        # Dominant axis is X
        sign_bit = 1 if x > 0 else 0
        encoded |= (sign_bit << 22)
        # Face index 0 is implicit (bits 21-20 are 0)
        encoded |= (int(cy_unorm * 1023) << 10)
        encoded |= int(cz_unorm * 1023)
    elif ay == m:
        # Dominant axis is Y
        sign_bit = 1 if y > 0 else 0
        encoded |= (sign_bit << 22)
        # Face index 1
        encoded |= (1 << 20)
        encoded |= (int(cx_unorm * 1023) << 10)
        encoded |= int(cz_unorm * 1023)
    elif az == m:
        # Dominant axis is Z
        sign_bit = 1 if z > 0 else 0
        encoded |= (sign_bit << 22)
        # Face index 2
        encoded |= (2 << 20)
        encoded |= (int(cx_unorm * 1023) << 10)
        encoded |= int(cy_unorm * 1023)
        
    return encoded
```

**Decoding:**
To reconstruct the normal from the encoded integer:
1. Extract `sign_bit = (encoded >> 22) & 1`.
2. Extract `axis = (encoded >> 20) & 3`.
3. Extract `comp2 = (encoded >> 10) & 0x3FF`.
4. Extract `comp1 = encoded & 0x3FF`.
5. Reconstruct components based on axis:
   - If `axis == 0` (X): $n_x = 2 \cdot \text{sign\_bit} - 1$, $n_y = \frac{\text{comp2}}{1023} \cdot 2 - 1$, $n_z = \frac{\text{comp1}}{1023} \cdot 2 - 1$.
   - If `axis == 1` (Y): $n_x = \frac{\text{comp2}}{1023} \cdot 2 - 1$, $n_y = 2 \cdot \text{sign\_bit} - 1$, $n_z = \frac{\text{comp1}}{1023} \cdot 2 - 1$.
   - If `axis == 2` (Z): $n_x = \frac{\text{comp2}}{1023} \cdot 2 - 1$, $n_y = \frac{\text{comp1}}{1023} \cdot 2 - 1$, $n_z = 2 \cdot \text{sign\_bit} - 1$.
6. Normalize the resulting vector $\mathbf{n}$ to unit length [S1, unofficial].

**Parameters:**
- Precision: 10 bits per coordinate component (1024 levels).
- Sign: 1 bit.
- Face ID: 2 bits (values 0, 1, 2 used; 3 is invalid).
- Total: 32 bits.

**Edge Cases and Pitfalls:**
- If multiple components have the same absolute maximum value, the order of checks (X, then Y, then Z) determines the face index. This is a deterministic but arbitrary tie-breaking rule [S1, unofficial].
- The decoding step requires renormalization because the cubic projection distorts the vector length [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for both encoding and decoding.
- **Memory Complexity:** 32 bits per normal vector.
- **Strengths:** Higher precision for axis-aligned normals compared to octahedral encoding. Simple bitwise operations for fast encoding/decoding.
- **Limitations:** Discontinuities at the boundaries between cube faces. Requires renormalization upon decoding.
- **When to use:** When storing normals in memory-constrained environments like sparse voxel octrees and axis-aligned normals are common.
- **When not to use:** When smooth interpolation across faces is required without renormalization artifacts, or when spherical symmetry is preferred.

## Background
Cubic Normal Encoding is used in the context of Sparse Voxel Octrees (SVO) for volume rendering, specifically in implementations like the one by xandergos [S1, unofficial]. It serves as a method to compress normal attributes within the octree nodes. The method is part of a family of vector normalization and encoding techniques designed to reduce the storage footprint of geometric data in 3D graphics applications.

## Sources
[S1] xandergos/unity-sparse-voxel-octrees, https://github.com/xandergos/unity-sparse-voxel-octrees, fetched 2026-10-26 (unofficial)
