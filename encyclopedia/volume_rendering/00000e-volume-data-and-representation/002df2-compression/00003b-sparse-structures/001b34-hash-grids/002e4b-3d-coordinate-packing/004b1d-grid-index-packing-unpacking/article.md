# Grid Index Packing & Unpacking

Also called: 3D coordinate to integer mapping, Bit-packing 3D coordinates into a 32-bit integer, Linearization of 3D grid indices, Voxel ID via bit shifting

## Overview
This method converts three 3D voxel coordinates $(x, y, z)$ into a unique single integer ID and reverses the operation using bit shifting and addition [S1, unofficial]. It is designed for sparse voxel grids where dimensions fit within specific bit limits, providing a compact key for hash map indexing [S1, S2, unofficial]. Unlike general space-filling curves (e.g., Z-order), this method relies on fixed bit-width allocations, making it faster but dimensionally constrained [S1, unofficial].

## How it works
**Inputs:** Three integer coordinates $(ix, iy, iz)$.
**Outputs:** A packed integer `packed` (encoding) or a tuple $(ix, iy, iz)$ (decoding).

**Data Structures:**
The packed value is stored in a 32-bit unsigned integer (`uint32_t`) or signed integer (`int32_t`), depending on coordinate sign handling [S1, S2, unofficial].

**Parameters:**
Two common bit-allocation schemes exist in the sources:
1.  **Scheme A (Symmetric):** 10 bits for $x$, 10 bits for $y$, 10 bits for $z$. This uses 30 bits, leaving 2 bits unused in a 32-bit integer [S1, unofficial].
    *   Max world size: $x \in [0, 2^{10}-1]$, $y \in [0, 2^{10}-1]$, $z \in [0, 2^{10}-1]$ [S1, unofficial].
    *   Source [S1, unofficial] claims a max world size of $4.295 \times 10^9$ for $x$, which contradicts the 10-bit allocation ($2^{10}=1024$). The claim $4.295 \times 10^9$ corresponds to $2^{32}/1024$, implying the *remaining* bits after shifting, but the shift amounts (10, 20) in the code snippet only support 10-bit coordinates. We follow the code snippet's shift logic [S1, unofficial].
2.  **Scheme B (Asymmetric):** 10 bits for $ix$, 11 bits for $iy$, 11 bits for $iz$. Total 32 bits [S2, unofficial].
    *   Max values: $ix < 1024$, $iy < 2048$, $iz < 2048$ [S2, unofficial].

**Steps:**

1.  **Encode:** Shift coordinates to their allocated bit positions and combine them using bitwise OR (or addition, as OR is safe when bit ranges do not overlap) [S1, S2, unofficial].
2.  **Decode:** Shift the packed integer right by the allocated bits and apply a mask to extract each coordinate [S1, S2, unofficial].

**Formulas:**

*Scheme A (10-bit each, from [S1, unofficial]):*
$$ \text{packed} = x + (y \ll 10) + (z \ll 20) $$
$$ x = \text{packed} \& 0x3FF $$
$$ y = (\text{packed} \gg 10) \& 0x3FF $$
$$ z = (\text{packed} \gg 20) \& 0x3FF $$

*Scheme B (10/11/11-bit, from [S2, unofficial]):*
$$ \text{packed} = (ix \ll 22) \ | \ (iy \ll 11) \ | \ iz $$
$$ ix = \text{packed} \gg 22 $$
$$ iy = (\text{packed} \gg 11) \ \& \ 0x7FF $$
$$ iz = \text{packed} \ \& \ 0x7FF $$

**Pseudocode:**
The following implementation uses Scheme B (10/11/11-bit) as it fully utilizes the 32-bit integer and includes explicit masking for safety [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_index(ix: int, iy: int, iz: int) -> int:
    """
    Packs 3D coordinates into a 32-bit integer.
    ix: 10 bits (max 1023)
    iy: 11 bits (max 2047)
    iz: 11 bits (max 2047)
    """
    # Ensure inputs are non-negative for unsigned interpretation
    # If signed inputs are required, see Pitfalls
    return (ix << 22) | (iy << 11) | iz

def unpack_index(packed: int) -> tuple:
    """
    Unpacks a 32-bit integer into 3D coordinates.
    """
    ix = packed >> 22
    iy = (packed >> 11) & 0x7FF  # 0x7FF is 2047 (11 ones)
    iz = packed & 0x7FF
    return (ix, iy, iz)
```

**Edge Cases and Pitfalls:**
*   **Negative Coordinates:** The method assumes non-negative coordinates. If negative numbers are used, the behavior depends on the integer type (signed vs. unsigned) and the language's shift operator (arithmetic vs. logical shift) [S1, unofficial]. In C/C++, right-shifting a negative signed integer is implementation-defined or arithmetic, which may corrupt the sign bit during unpacking if not handled carefully [S1, unofficial].
*   **Overflow:** Coordinates exceeding their bit limits will overflow into adjacent fields, causing collisions and incorrect unpacking [S1, unofficial].
*   **Bit Width:** Scheme A leaves 2 bits unused. Scheme B uses all 32 bits but restricts $x$ to 10 bits [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for both packing and unpacking. It involves only bitwise shifts and OR/AND operations [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per voxel key. Stores 3 coordinates in 4 bytes (32 bits) [S1, S2, unofficial].
*   **Performance:** Described as "the fastest" hash/unhash method due to minimal CPU instructions [S1, unofficial].
*   **Strengths:** Extremely fast, low memory overhead, simple implementation [S1, unofficial].
*   **Limitations:** Fixed dimension limits; not suitable for arbitrary large grids without changing integer size (e.g., to 64-bit) [S1, unofficial]. Does not preserve spatial locality (unlike Z-order curves), so adjacent voxels may have distant hash keys [S1, unofficial].

## Background
*   **Origin:** The method is a standard technique in computer graphics for sparse voxel grid indexing. Source [S1, unofficial] presents it as a Stack Overflow answer from 2013, noting its speed advantage [S1, unofficial]. Source [S2, unofficial] uses it in a volumetric display project [S2, unofficial].
*   **History:** It is a variant of linear indexing for multi-dimensional arrays, adapted for sparse storage where hash maps are used instead of dense arrays.
*   **Usage:** Used in sparse voxel octrees, hash-based voxel grids, and real-time volumetric rendering where fast coordinate-to-key conversion is critical [S1, S2, unofficial].

## Sources
[S1] Voxel unique ID in 3D space, https://stackoverflow.com/questions/20286179/voxel-unique-id-in-3d-space, fetched 2026-10-26 (unofficial)
[S2] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
