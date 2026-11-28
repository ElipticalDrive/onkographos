# Swizzling
Also called: Bit-interleaved addressing with hole-skipping, Bit-swizzling, Bulk Processing, Software Swizzling, Subtract-and-mask swizzling, Swizzled Texture Addressing, Swizzled Texture Addressing with Bitwise Carry Propagation

## Overview
Swizzling is a memory layout optimization technique where the linear memory address bits are rearranged (swizzled) to improve spatial locality and cache performance [S1, unofficial]. It effectively forms the linear address then swaps some of the memory address bits around [S1, unofficial]. Unlike simple tiling, which chops images into subrects, swizzling is a broader category involving bit rearrangement, including nested tiling schemes [S1, unofficial]. It is particularly effective for 2D data traversed in both row and column directions, such as textures in GPU rendering or matrices in BLAS libraries [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: Source data in standard linear (row-major) order; swizzle pattern parameters (tile sizes, bit interleaving rules).
- **Output**: Data stored in swizzled memory layout; destination pointers updated using optimized arithmetic.

### Data Structures
The method relies on bit masks and offset variables.
- `offs_x`, `offs_y`: 32-bit unsigned integers representing the byte offset within the swizzled memory space for the current x and y coordinates.
- `x_mask`, `y_mask`: 32-bit unsigned integers where bits are set to 1 at positions corresponding to the x and y coordinate bits in the swizzled address, respectively. Bits corresponding to "holes" (reserved for other coordinates or tile indices) are 0.
- `incr_y`: 32-bit unsigned integer representing the byte increment required to move from one tile row to the next in non-power-of-2 layouts.

### Algorithm Steps
1. **Initialization**:
   - Compute initial offsets for the starting tile $(x_0, y_0)$ using swizzle functions:
     $$ offs\_x0 = \text{swizzle\_x}(x_0) $$
     $$ offs\_y = \text{swizzle\_y}(y_0) $$
   - Compute masks by swizzling all 1s (bitwise NOT of 0, i.e., `~0u`):
     $$ x\_mask = \text{swizzle\_x}(\sim 0u) $$
     $$ y\_mask = \text{swizzle\_y}(\sim 0u) $$
   - For non-power-of-2 textures, compute the y-increment for tile rows:
     $$ incr\_y = \text{swizzle\_x}(\text{padded\_width}) $$
   - Adjust the initial x-offset to account for the starting tile row:
     $$ offs\_x0 \leftarrow offs\_x0 + incr\_y \times (y_0 / \text{tile\_height}) $$

2. **Outer Loop (Y-axis)**:
   - Iterate $y$ from $y_0$ to $y_1$.
   - Reset source pointer to the start of the current row.
   - Set destination line base: $dest\_line = dest + offs\_y$.
   - Reset inner x-offset: $offs\_x = offs\_x0$.

3. **Inner Loop (X-axis)**:
   - Iterate $x$ from $x_0$ to $x_1$.
   - Read pixel from source.
   - Write pixel to destination: $*(dest\_line + offs\_x) = \text{pixel}$.
   - Update x-offset using the subtract-and-mask formula:
     $$ offs\_x \leftarrow (offs\_x - x\_mask) \land x\_mask $$
     This increments the x-bits while carrying through 'holes' (bits reserved for y or tile index) [S1, unofficial].

4. **End of Row Processing**:
   - Update y-offset:
     $$ offs\_y \leftarrow (offs\_y - y\_mask) \land y\_mask $$
   - Check for wrap-around to the next tile row (specifically for non-power-of-2 textures where $offs\_y$ wraps to zero):
     $$ \text{if } (!offs\_y) \text{ then } offs\_x0 \leftarrow offs\_x0 + incr\_y $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def swizzle_texture(src, dest, x0, y0, x1, y1, padded_width, tile_height, swizzle_x_func, swizzle_y_func):
    # Initialization
    offs_x0 = swizzle_x_func(x0)
    offs_y = swizzle_y_func(y0)
    x_mask = swizzle_x_func(~0)  # Swizzle all 1s
    y_mask = swizzle_y_func(~0)  # Swizzle all 1s
    
    # Non-power-of-2 handling
    incr_y = swizzle_x_func(padded_width)
    offs_x0 += incr_y * (y0 // tile_height)

    for y in range(y0, y1):
        src_row = src + y * src_pitch
        dest_line = dest + offs_y
        offs_x = offs_x0

        for x in range(x0, x1):
            pixel = src_row[x]
            dest_line[offs_x] = pixel
            # Core update rule: offs' = (offs - mask) & mask
            offs_x = (offs_x - x_mask) & x_mask

        # Update y-offset
        offs_y = (offs_y - y_mask) & y_mask
        
        # Handle wrap-around for non-power-of-2 textures
        if offs_y == 0:
            offs_x0 += incr_y
```

### Parameters
- **Inner loop tile size**: Depends on cache line size and texture format. For Morton order, 32bpp, and 64-byte cache lines, the size is 4x4 pixels [S1, unofficial]. For 128-byte cache lines, it is 8x4 pixels [S1, unofficial].
- **swizzle_x**: Maps an x-coordinate to its bit-interleaved position. Must handle both the swizzled x-part and the tile index for non-power-of-2 textures [S1, unofficial].
- **swizzle_y**: Maps a y-coordinate to its bit-interleaved position. Must ignore the tile index bits [S1, unofficial].

### Edge Cases and Pitfalls
- **Non-power-of-2 textures**: The y-mask cannot overlap with the x-mask because the y-increment involves adding `tilesPerRow` (not a power of 2), which breaks the simple carry trick [S1, unofficial]. The solution is to detect the wrap-around ($offs\_y == 0$) and manually adjust $offs\_x0$ [S1, unofficial].
- **Non-square power-of-2 textures**: Straight Morton order swizzling only works well with square dimensions. Non-square power-of-2 textures require padding or a nested tiling approach [S1, unofficial].

## Subtopics

### Nested Tiling
Nested Tiling is a swizzle construction method where different tile sizes are nested (e.g., 8x8 tiles inside 32x32 tiles) [S1, unofficial]. Morton order is viewed as 2x2 tiles inside 4x4 tiles inside 8x8 tiles, etc. [S1, unofficial].
- **When to choose**: Use for non-square power-of-2 textures (W > H) by using tiles of HxH pixels with Morton-order swizzling inside them, avoiding padding to square size [S1, unofficial]. For non-power-of-2 textures, define a top-level tile size (e.g., 32x32), pad textures to multiples of this size, and use regular 2D row-major order for areas above this tile size [S1, unofficial].
- **Property**: Nested tiling schemes consume bits from x and y coordinates in order, never swapping the relative order of bits inside a coordinate [S1, unofficial].

### Morton Order Swizzling
Morton Order Swizzling is a swizzle pattern that interleaves the x and y coordinate bits, also called Z-order because the storage order follows a recursive Z-shaped pattern [S1, unofficial].
- **Layout**: For a 256x256 32bpp texture, the linear address bits (y7..y0, x7..x0, c1..c0) are rearranged to: y7, x7, y6, x6, y5, x5, y4, x4, y3, x3, y2, x2, y1, x1, y0, x0, c1, c0 [S1, unofficial].
- **Pitfall**: Straight Morton order swizzling only works well with square dimensions. Non-square power-of-2 textures require padding or a nested tiling approach [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The inner loop update is $O(1)$ per pixel, using only addition, subtraction, and bitwise AND operations [S1, unofficial]. This replaces the more expensive bitwise OR addressing computation `dest + (offs_y | offs_x)` with a sum `(dest + offs_y) + offs_x` [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space for masks and offsets [S1, unofficial].
- **Performance**: On PowerPC, the inner loop can be compiled into four instructions per pixel (lwzu, subf, stwx, and) [S1, unofficial]. Writing full aligned cache lines allows skipping around in memory without incurring performance penalties due to write combining [S1, unofficial].
- **Strengths**: High cache locality for 2D access patterns; efficient hardware implementation (no logic required for bit swapping) [S1, unofficial].
- **Limitations**: Complex software implementation for arbitrary swizzle patterns; requires careful handling of non-power-of-2 and non-square dimensions [S1, unofficial].
- **When to use**: When accessing 2D data in both row and column directions, such as GPU textures or tiled matrices in BLAS [S1, unofficial].
- **When not to use**: When data access is primarily 1D or random, as the overhead of bit manipulation may not be justified [S1, unofficial].

## Background
Swizzling was described by RasterGarden (Felix K.) in a blog post titled "Texture tiling and swizzling" in 2011 [S1, unofficial]. The technique is used in Vivante GPUs for the 'supertiling' layout and is referenced in Intel Developer Documentation as a recommended optimization for accessing shared GPU/CPU resources [S1, unofficial]. The "subtract-and-mask" method for incrementing offsets is a generalization of techniques used for Morton codes, applicable to arbitrary bit-skip patterns [S1, unofficial].

## Sources
[S1] Texture tiling and swizzling | The ryg blog, https://fgiesen.wordpress.com/2011/01/17/texture-tiling-and-swizzling/, fetched 2026-10-26 (unofficial)
