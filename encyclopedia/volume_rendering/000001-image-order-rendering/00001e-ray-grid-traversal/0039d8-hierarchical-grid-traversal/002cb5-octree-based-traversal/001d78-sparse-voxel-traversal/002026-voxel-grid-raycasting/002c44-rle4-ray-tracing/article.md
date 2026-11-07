# RLE4 Ray Tracing

Also called: Map4 Traversal

## Overview
RLE4 Ray Tracing is a ray-marching and voxel-intersection algorithm that traverses an RLE4 (Run-Length Encoded 4D) compressed volume structure, also known as Map4, to locate the first occupied voxel along a ray or to colorize a voxel based on its position [S1, unofficial]. It distinguishes itself from standard voxel grid traversal by using slab-based RLE encoding to skip empty space efficiently [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A target voxel coordinate $(x, y, z)$ within the volume bounds, the volume dimensions $(v_x, v_y, v_z)$, and the RLE4 compressed volume data structure (`map4`) [S1, unofficial].
- **Output:** The color value of the voxel at $(x, y, z)$ if occupied, or termination if empty [S1, unofficial].

**Data Structures:**
The `map4` structure contains:
- `sx`: The width of the volume in voxels (X dimension) [S1, unofficial].
- `slabs`: A pointer to an array of `ushort` (16-bit unsigned integers) representing the slab map, indexed by `x + z * sx` [S1, unofficial].
- Each entry in the `slabs` array for a specific $(x, z)$ coordinate consists of:
  1. `len1`: The number of slab segments in the first run [S1, unofficial].
  2. `len2`: The number of slab segments in the second run [S1, unofficial].
  3. `len1` slab entries, each encoded as a `ushort` [S1, unofficial].

**Slab Entry Encoding:**
Each slab entry `slab` is a `ushort` packed with Y-range information [S1, unofficial]:
- Lower 10 bits (`slab & 1023`): The starting Y coordinate offset for the slab segment [S1, unofficial].
- Upper 6 bits (`slab >> 10`): The length of the slab segment in Y [S1, unofficial].
- The ending Y coordinate is $y_{end} = y_{start} + \text{length}$ [S1, unofficial].

**Numbered Steps:**
1. **Coordinate Conversion:** Calculate integer voxel coordinates $x, y, z$ from the ray's float position, scaled by the volume dimensions $v_x, v_y, v_z$ [S1, unofficial].
2. **Bounds Check:** Verify that $0 \le x < v_x$, $0 \le y < v_y$, and $0 \le z < v_z$. If any coordinate is out of bounds, terminate the search [S1, unofficial].
3. **Slab Pointer Access:** Compute the index into the slab map: $idx = x + z \cdot v_x$. Access the slab pointer: $p = \text{map4.slabs} + idx$ [S1, unofficial].
4. **Read Lengths:** Read the first two `ushort` values from $p$:
   - $len1 = *p$
   - $p \leftarrow p + 1$
   - $len2 = *p$
   - $p \leftarrow p + 1$ [S1, unofficial].
5. **Initialize Variables:**
   - Set $y_1 = 0$, $y_2 = 0$ [S1, unofficial].
   - Set texture pointer $pt = p + len1$ [S1, unofficial].
   - Initialize `texture = 0` [S1, unofficial].
6. **Iterate Slabs:** Loop $s$ from $0$ to $len1 - 1$:
   a. Read slab entry: $slab = *p$; $p \leftarrow p + 1$ [S1, unofficial].
   b. Decode Y-range:
      - $y_1 \leftarrow y_1 + (slab \ \& \ 1023)$
      - $y_2 \leftarrow y_1 + (slab \gg 10)$ [S1, unofficial].
   c. Check if target $y$ is in $[y_1, y_2)$:
      - If $y_2 > y$ and $y_1 \le y$:
        - The voxel is found [S1, unofficial].
        - Retrieve the color value `col` from the texture data at `pt[texture + y - y1]` [S1, unofficial].
        - Decode color components (5 bits per channel, range 0-31):
          - $r = \text{clamp}((col \ \& \ 0x1F) \cdot 15 + 16, 0, 31)$
          - $g = \text{clamp}(((col \gg 5) \ \& \ 0x1F) \cdot 15 + 16, 0, 31)$
          - $b = \text{clamp}(((col \gg 10) \ \& \ 0x1F) \cdot 15 + 16, 0, 31)$ [S1, unofficial].
        - Store packed RGB back into the texture array:
          - $pt[\text{texture} + y - y_1] = (r \ | \ (g \ll 5) \ | \ (b \ll 10))$ [S1, unofficial].
        - Return.
   d. If not found, advance:
      - $\text{texture} \leftarrow \text{texture} + (y_2 - y_1)$
      - $y_1 \leftarrow y_2$ [S1, unofficial].
7. **Termination:** If the loop completes without finding the voxel, the voxel is empty [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rle4_traverse(map4, x, y, z, vx, vy, vz):
    # 1. Bounds Check
    if x < 0 or x >= vx or y < 0 or y >= vy or z < 0 or z >= vz:
        return None

    # 2. Access Slab Map
    idx = x + z * vx
    p = map4.slabs[idx]
    
    # 3. Read Lengths
    len1 = p[0]
    len2 = p[1]
    p = p[2:]
    
    # 4. Initialize
    y1 = 0
    y2 = 0
    pt = p + len1  # Pointer arithmetic pseudocode
    texture_offset = 0
    
    # 5. Iterate Slabs
    for s in range(len1):
        slab = p[0]
        p = p[1:]
        
        # Decode Y-range
        y1 = y1 + (slab & 1023)
        y2 = y1 + (slab >> 10)
        
        # Check if y is in [y1, y2)
        if y2 > y and y1 <= y:
            # Voxel found
            col = pt[texture_offset + y - y1]
            
            # Decode 5-bit RGB
            r_raw = col & 0x1F
            g_raw = (col >> 5) & 0x1F
            b_raw = (col >> 10) & 0x1F
            
            # Quantize to 0-31 range
            r = clamp(r_raw * 15 + 16, 0, 31)
            g = clamp(g_raw * 15 + 16, 0, 31)
            b = clamp(b_raw * 15 + 16, 0, 31)
            
            # Store packed RGB back
            new_col = r | (g << 5) | (b << 10)
            pt[texture_offset + y - y1] = new_col
            
            return new_col
        
        # Advance
        texture_offset += (y2 - y1)
        y1 = y2
        
    return None

def clamp(val, min_val, max_val):
    if val < min_val: return min_val
    if val > max_val: return max_val
    return val
```

**Parameters:**
- **Color Quantization:** 5 bits per channel (RGB), resulting in a 31-level intensity range (0-31) [S1, unofficial].
- **Slab Encoding:** 10 bits for Y-start, 6 bits for Y-length [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Bounds:** The algorithm assumes the target Y is within volume bounds, checked at the start. If the ray passes through multiple slabs, it iterates until the Y range is found [S1, unofficial].
- **Precision:** The Y-coordinate decoding uses integer arithmetic. The source uses `int` for $y_1, y_2$, so overflow is unlikely for typical volume sizes [S1, unofficial].
- **Empty Space:** The RLE4 structure efficiently skips empty space by only storing occupied slab segments. If a voxel is not found in the loop, it is considered empty [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] sp4cerat/OpenGL-Voxel-Splatting, https://github.com/sp4cerat/OpenGL-Voxel-Splatting, fetched 2026-10-27 (unofficial)
