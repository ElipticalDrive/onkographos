# Early Ray Termination with Binary Mask

Also called: Alpha-blending with binary skip

## Overview
An acceleration technique for alpha-blending that uses a binary volume to skip fully transparent cells and terminates the front-to-back evaluation when accumulated transparency falls below a threshold [S1, unofficial]. It sets itself apart from standard early ray termination by explicitly encoding the presence of opaque voxels into a bit-mask structure, allowing the renderer to jump directly to non-empty regions rather than sampling every voxel along the ray [S1, unofficial].

## How it works
**Inputs:**
- Volume data: A 3D array of voxels, where each voxel contains a color and an opacity value.
- Binary mask: A 3D array of 32-bit integers, `mask[i][y][x]`, where `i` represents a block of 32 depth slices, and `y, x` are pixel coordinates. Each bit in the integer corresponds to a voxel in that block; a set bit indicates the voxel is opaque (or non-transparent), and a cleared bit indicates it is transparent.
- Threshold: A pre-defined double value for accumulated transparency below which the ray terminates.

**Outputs:**
- A single RGB color value for the pixel at `(x, y)`.

**Data Structures:**
- `mask`: Layout is `[Z/32][Height][Width]` (integer). Each integer stores 32 bits. Bit 0 corresponds to the first slice in the block, bit 31 to the last.
- `volume`: Layout is `[Z][Height][Width]` (voxel).
- `trans`: Accumulated transparency, initialized to 1.0.
- `color`: Accumulated color, initialized to BLACK.

**Algorithm Steps:**
1. Calculate the number of 32-slice blocks: $i_{max} = \lceil Z / 32 \rceil$.
2. Initialize `trans = 1.0` and `color = BLACK`.
3. Loop `i` from 0 to $i_{max} - 1$:
    a. Retrieve the segment integer: `segment = mask[i][y][x]`.
    b. While `segment` is non-zero:
        i. Find the position of the least significant set bit (or the next relevant bit depending on implementation convention, but the source implies finding the next non-zero bit to process). Let this bit offset be `z`. The source code uses `Depth(segment)` to find this.
        ii. Calculate the absolute depth index: $z_{abs} = 32 \times i + z$.
        iii. Sample the voxel: $v = \text{volume}[z_{abs}][y][x]$.
        iv. Update accumulated transparency: $\text{trans} = \text{trans} \times (1.0 - v.\text{opacity})$.
        v. Check termination: If $\text{trans} < \text{threshold}$, return `color`.
        vi. Update accumulated color: $\text{color} = \text{color} + v.\text{color} \times v.\text{opacity} \times \text{trans}$.
        vii. Clear the processed bit in the segment: `segment = segment & ~(0x80000000 >> z)`. Note: The source code uses `0x80000000 >> z`. This implies bit 31 is the most significant bit. If `z` is the bit index (0-31), `0x80000000` is $2^{31}$. Shifting it right by `z` creates a mask with a 1 at bit position $31-z$. This suggests the source might be treating bit 0 as the MSB or using a specific bit ordering. However, standard bit manipulation usually uses `1 << z`. The source explicitly writes `segment &= ~(0x80000000 >> z)`. We must follow the source's explicit formula.
        viii. Call `Depth(segment)` again to find the next non-zero bit.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alpha_blending_binary_mask(x, y, volume, mask, threshold, Z):
    """
    Renders a pixel using early ray termination with a binary mask.
    
    Args:
        x, y: Pixel coordinates.
        volume: 3D array of voxels [Z][Y][X], each with .color and .opacity.
        mask: 3D array of integers [Z//32][Y][X].
        threshold: Transparency threshold for early termination.
        Z: Total depth of the volume.
        
    Returns:
        RGB color.
    """
    color = (0.0, 0.0, 0.0)  # BLACK
    trans = 1.0
    i_max = (Z + 31) // 32
    
    for i in range(i_max):
        segment = mask[i][y][x]
        
        while segment != 0:
            # Find the position of the next non-zero bit.
            # The source uses a routine Depth(segment).
            # We assume Depth returns the bit index z (0-31) of the next set bit.
            # Note: The exact bit ordering (LSB vs MSB) depends on the implementation
            # of Depth and the mask construction. The source code uses 0x80000000 >> z,
            # implying z=0 corresponds to the MSB (bit 31) if we interpret the shift
            # as creating a mask for the MSB. However, standard "find first set bit"
            # usually returns the LSB index.
            # To remain faithful to the source's arithmetic:
            # segment &= ~(0x80000000 >> z)
            # If z=0, mask is ~0x80000000 (clears bit 31).
            # If z=31, mask is ~0x00000001 (clears bit 0).
            # This implies z=0 is the MSB (depth 31 in block) and z=31 is LSB (depth 0 in block).
            # Or, more likely, the source's Depth function returns the index such that
            # 0x80000000 >> z targets the correct bit.
            # Let's assume Depth returns an index z in [0, 31].
            
            z = depth_of_next_set_bit(segment)
            
            # Calculate absolute z-coordinate
            # The source states: z-coordinate of the voxel is calculated as 32 * i + z
            z_abs = 32 * i + z
            
            # Sample voxel
            v = volume[z_abs][y][x]
            
            # Update transparency
            trans = trans * (1.0 - v.opacity)
            
            # Early termination check
            if trans < threshold:
                return color
            
            # Update color
            # color += v.color * v.opacity * trans
            color = (
                color[0] + v.color[0] * v.opacity * trans,
                color[1] + v.color[1] * v.opacity * trans,
                color[2] + v.color[2] * v.opacity * trans
            )
            
            # Clear the processed bit
            # Source: segment &= ~(0x80000000 >> z)
            # 0x80000000 is 2^31.
            mask_bit = 0x80000000 >> z
            segment = segment & (~mask_bit)
            
    return color

def depth_of_next_set_bit(segment):
    """
    Finds the bit index z such that 0x80000000 >> z targets the next set bit.
    This is a helper to implement the source's specific bit manipulation logic.
    If the source implies standard LSB-first processing, this function would
    need to map the LSB index to the MSB-shifted mask.
    However, without a definition of Depth, we note this as a required helper.
    [G] Standard bit scanning instructions (e.g., __builtin_ctz) return LSB index.
    If Depth returns LSB index k, then z = 31 - k to match the source's mask shift.
    """
    # Implementation detail: The source code is the primary reference.
    # We assume Depth is available.
    raise NotImplementedError("Depth routine must be implemented to find bit position")
```

**Parameters:**
- `threshold`: A pre-defined value for accumulated transparency. If `trans < threshold`, the ray evaluation terminates [S1, unofficial]. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
- **Bit Ordering:** The source code uses `0x80000000 >> z`. This is non-standard for LSB-first bit scanning. The implementer must ensure the `Depth` function and the mask construction are consistent with this shift direction. If `Depth` returns the index of the least significant bit (0), the mask `0x80000000 >> 0` is `0x80000000` (bit 31). This suggests the source treats bit 0 of the integer as the MSB or the `Depth` function returns an index relative to the MSB.
- **Precision:** Floating-point multiplication for `trans` and `color` accumulation can suffer from precision loss over long rays.
- **Empty Rays:** If the mask is 0 for a pixel, the loop is skipped, and BLACK is returned.

## Operating characteristics
- **Time Complexity:** Proportional to the number of opaque voxels along the ray, rather than the total depth $Z$. In sparse volumes, this is significantly faster than full ray casting.
- **Memory Complexity:** Requires an additional binary mask array of size $Z/32 \times Y \times X$ integers.
- **Strengths:** Efficiently skips large transparent regions. Simple to implement alongside standard alpha blending.
- **Limitations:** Requires pre-computation of the binary mask. Performance degrades if the volume is dense (many opaque voxels), as the mask provides little skipping benefit.
- **When to use:** Sparse volumes where large transparent regions exist.
- **When not to use:** Dense volumes or when memory bandwidth is the primary bottleneck (due to mask access).

## Background
This technique is described in the context of lookup-table based volume rendering for medical data visualization [S1, unofficial]. It is noted as a well-known acceleration technique in volume rendering literature, specifically referred to as early ray termination [S1, unofficial]. The specific use of a binary mask to skip cells is attributed to the work presented in the source [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
