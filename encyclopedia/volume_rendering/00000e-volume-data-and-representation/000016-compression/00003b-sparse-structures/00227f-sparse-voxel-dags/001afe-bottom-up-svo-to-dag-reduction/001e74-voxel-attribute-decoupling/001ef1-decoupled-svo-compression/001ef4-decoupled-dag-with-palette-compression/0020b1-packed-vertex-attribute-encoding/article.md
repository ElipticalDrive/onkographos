# Packed Vertex Attribute Encoding

Also called: 32-bit vertex packing, PackedVertex, Vertex compression

## Overview
Packed Vertex Attribute Encoding is a compression scheme for vertex attributes in hardware-accelerated volume splatting that reduces memory usage by packing volume texture coordinates and Gaussian texture coordinates into 32-bit integers per vertex [S1]. It sets itself apart from sibling methods by specifically targeting the attribute storage of proxy geometry in EWA splatting, decoupling coordinate components into fixed-width bit fields to achieve high compression ratios while allowing fast decompression via lookup tables [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume texture coordinates ($tx, ty$) and Gaussian texture coordinate components ($fx, fy$) for a single vertex [S1].
*   **Output:** A single 32-bit unsigned integer (`uint32_t`) containing the packed attributes [S1].

**Data Structures:**
The packed data is stored in a 32-bit unsigned integer [S1]. The bit layout consists of six fields with fixed widths totaling 32 bits: $mx$ (7 bits), $ix$ (8 bits), $my$ (7 bits), $iy$ (8 bits), $fx$ (1 bit), and $fy$ (1 bit) [S1]. The exact bit ordering (LSB to MSB) is not specified in the sources [S1].

**Formulas:**
The volume texture coordinates $tx$ and $ty$ are decomposed into integer components:
$$ tx = mx \times 256 + ix $$
$$ ty = my \times 256 + iy $$
Where:
*   $mx = \lfloor tx / 256 \rfloor$
*   $ix = tx \pmod{256}$
*   $my = \lfloor ty / 256 \rfloor$
*   $iy = ty \pmod{256}$ [S1]

The Gaussian components $fx$ and $fy$ are binary values (0 or 1) [S1].

**Parameters:**
*   $mx, my$: Stored using 7 bits each, allowing volume slices up to $2^{15} \times 2^{15}$ voxels [S1].
*   $ix, iy$: Stored using 8 bits each [S1].
*   $fx, fy$: Stored using 1 bit each [S1].

**Steps:**
1.  **Decompose Coordinates:** For each vertex, calculate $mx, ix, my, iy$ from the integer volume texture coordinates $tx, ty$ using the formulas above [S1].
2.  **Validate Gaussian Components:** Ensure $fx$ and $fy$ are binary (0 or 1) [S1].
3.  **Pack Bits:** Construct the 32-bit integer by shifting and masking the components into the defined bit fields [S1].
4.  **Store:** Write the resulting 32-bit integer to the vertex buffer [S1].

**Decoding:**
1.  **Extract:** Retrieve the 32-bit integer from the vertex buffer [S1].
2.  **Unpack:** Extract $ix, mx, iy, my, fx, fy$ using bitwise operations and masks corresponding to their bit widths [S1].
3.  **Reconstruct:** Calculate $tx = mx \times 256 + ix$ and $ty = my \times 256 + iy$ [S1].
4.  **Lookup:** Fast decompression can be implemented using a small lookup table, as used in Direct3D implementations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_vertex_attributes(tx: int, ty: int, fx: int, fy: int) -> int:
    """
    Packs volume texture coordinates (tx, ty) and Gaussian coordinates (fx, fy)
    into a single 32-bit integer according to the specified bit widths.
    Note: Exact bit ordering is not specified in the source [S1]. This implementation
    packs fields contiguously from LSB to MSB: ix, mx, iy, my, fx, fy.
    """
    # Decompose tx and ty
    mx = tx // 256
    ix = tx % 256
    my = ty // 256
    iy = ty % 256
    
    # Validate inputs
    assert 0 <= mx < 128, "mx must fit in 7 bits"
    assert 0 <= ix < 256, "ix must fit in 8 bits"
    assert 0 <= my < 128, "my must fit in 7 bits"
    assert 0 <= iy < 256, "iy must fit in 8 bits"
    assert fx in (0, 1), "fx must be 0 or 1"
    assert fy in (0, 1), "fy must be 0 or 1"
    
    # Pack into 32-bit integer (LSB-first ordering)
    packed = 0
    packed |= (ix & 0xFF) << 0
    packed |= (mx & 0x7F) << 8
    packed |= (iy & 0xFF) << 15
    packed |= (my & 0x7F) << 23
    packed |= (fx & 0x1) << 30
    packed |= (fy & 0x1) << 31
    
    return packed

def unpack_vertex_attributes(packed: int) -> tuple:
    """
    Unpacks a 32-bit integer into volume texture coordinates (tx, ty) 
    and Gaussian coordinates (fx, fy).
    """
    ix = (packed >> 0) & 0xFF
    mx = (packed >> 8) & 0x7F
    iy = (packed >> 15) & 0xFF
    my = (packed >> 23) & 0x7F
    fx = (packed >> 30) & 0x1
    fy = (packed >> 31) & 0x1
    
    tx = mx * 256 + ix
    ty = my * 256 + iy
    
    return (tx, ty, fx, fy)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The decomposition assumes integer coordinates. If floating-point coordinates are used, they must be scaled and rounded to integers before packing [S1].
*   **Boundaries:** $mx$ and $my$ are limited to 7 bits, restricting volume slices to $2^{15} \times 2^{15}$ voxels [S1].
*   **Degenerate Input:** $fx$ and $fy$ must be binary. Non-binary values will corrupt the packing [S1].

## Operating characteristics
*   **Time Complexity:** Encoding is $O(1)$ per vertex. Decoding is $O(1)$ per vertex, potentially optimized with a lookup table [S1].
*   **Memory Complexity:** Reduces vertex memory from 16 bytes (4 floats) to 4 bytes (1 uint32) per vertex [S1].
*   **Measured Performance:** For a $256^3$ volume dataset, vertex memory is reduced from 2048M bytes to 12M bytes, a compression ratio of 171:1 [S1].
*   **Strengths:** High compression ratio, fast decompression, suitable for hardware-accelerated rendering [S1].
*   **Limitations:** Limited volume size ($2^{15} \times 2^{15}$), fixed Gaussian coordinate resolution (binary) [S1].
*   **When to Use:** Hardware-accelerated volume splatting with proxy geometry where memory bandwidth is a bottleneck [S1].
*   **When Not to Use:** Volumes larger than $2^{15} \times 2^{15}$ slices, or when higher precision for Gaussian coordinates is required [S1].

## Background
Packed Vertex Attribute Encoding was introduced in the context of hardware-accelerated adaptive EWA volume splatting [S1]. It was developed to address the memory bottleneck of storing vertex attributes for proxy geometry in volume rendering [S1]. The method allows the volume texture data and packed proxy geometry information to be pre-loaded in the local memory of graphics hardware for interactive rendering [S1].

## Sources
[S1] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28
