# Voctopus Sparse Voxel Octree

Also called: Voctopus, Voctopus Kernel, VoctopusKernel32, asm.js octree kernel

## Overview
Voctopus is a sparse voxel octree (SVO) implementation designed for direct shader rendering in WebGL without meshification, utilizing ray tracing techniques [S1, unofficial]. It distinguishes itself by using a compact, flat memory layout with pointer encoding to minimize memory footprint and enable high-performance traversal in JavaScript/asm.js environments [S1, unofficial]. Unlike general-purpose SVOs, it is optimized for small, fast kernels that handle pointer encoding, tree traversal, and ray-AABB intersection testing directly within a constrained buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Maximum tree depth $D_{MAX}$, initial buffer size (optional), ray origin $(r_x, r_y, r_z)$ and direction $(d_x, d_y, d_z)$ for traversal.
- **Output:** Voxel data (RGBA), traversal pointers, or boolean hit status for ray casting.

**Data Structures:**
The kernel manages a flat `Uint32Array` buffer (heap) acting as the octree storage. The layout is fixed at specific indices:

| Index | Variable | Description |
| :--- | :--- | :--- |
| 0 | $D_{MAX}$ | Maximum depth of the tree |
| 1 | $P_{FO}$ | First offset (root octet offset) |
| 2 | $P_{NO}$ | Next octet pointer (allocation cursor) |
| 3 | $P_{CUR}$ | Current pointer (traversal state) |
| 4 | $P_{TMP}$ | Temporary pointer |
| 5 | $D_{CUR}$ | Current depth in traversal |
| 6 | $LOC_X$ | Current X coordinate |
| 7 | $LOC_Y$ | Current Y coordinate |
| 8 | $LOC_Z$ | Current Z coordinate |
| 9 | $D_{TGT}$ | Target depth for traversal |
| 10 | $D_{INV}$ | Depth inverse ($D_{MAX} - D_{CUR}$) |
| 11 | $V_{TMP}$ | Temporary value |
| 12 | $STEPA$ | Step accumulator |
| 13 | $STEPB$ | Step buffer |
| 14 | $TRAVA$ | Traversal accumulator |
| 15 | $INITA$ | Initialization accumulator |

*Note: Indices are multiplied by 4 (shifted right by 2) for 32-bit access in the heap [S1, unofficial].*

**Voxel Value Layout:**
Each octant is a 32-bit value:
- Bits 0–23: RGB color (8 bits each).
- Bits 24–27: Alpha channel (4 bits).
- Bits 28–30: Reserved for rendering instructions.
- Bit 31: Flag indicating if the entry is a pointer or voxel data [S1, unofficial].

**Pointer Encoding:**
- **Encode:** $makeP(n) = (n \ll 1) \mid 1$. The least significant bit (LSB) is set to 1 to flag a value as a pointer [S1, unofficial].
- **Decode:** $pFrom(n) = ((\sim(0x7FFFFFFF + (n \& 1)) \& n) \gg 1)$. Returns 0 if $n$ is not a pointer (LSB is 0) [S1, unofficial].

**Voxel Packing:**
- **Pack:** $valFromRGBA(r, g, b, a) = (r \ll 24) + (g \ll 16) + (b \ll 8) + ((a \& 63) \ll 4)$. R, G, B are 8-bit; A is 4-bit (masked by 63) [S1, unofficial].
- **Unpack:**
  - $r = (n \gg 8) \& 0xFF$
  - $g = n \& 0xFF$
  - $b = (n \& 0xFF00) \gg 8$
  - $a = (n \& 0xF0) \gg 4$ [S1, unofficial]

**Octant Identity:**
To determine which of the 8 children an octet occupies at a given depth:
$octantIdentity(v, dc, dm) = (((v[2] \gg (dm - 1 - dc)) \& 1) \ll 2) \mid (((v[1] \gg (dm - 1 - dc)) \& 1) \ll 1) \mid ((v[0] \gg (dm - 1 - dc)) \& 1)$ [S1, unofficial].

**Parameters:**
- $MAX\_BUFFER$: $1024 \times 1024 \times 1024 \times 512$ bytes (512 GB) [S1, unofficial].
- $VK\_FO$: 100 (First Offset) [S1, unofficial].
- $VK\_OS$: 8 bytes (Octet Size) [S1, unofficial].
- $depth$: Required constructor parameter, minimum useful depth is 2 [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:**
   - Validate depth.
   - Calculate $octantSize = 8$ bytes.
   - Calculate $octetSize = octantSize \times 8 = 64$ bytes? *Correction: Source says octetSize is 8 bytes in kernel context, but standard octet is 8 voxels. The source notes "Octet size: 8 bytes for a single octet (in the kernel's internal representation context)". This likely refers to the stride or a specific internal metric. We follow the source's explicit "octetSize (octantSize * 8)" calculation in the constructor step, but note the conflict. The source explicitly states `octetSize = octantSize * 8` in the constructor step, but later says "Octet size: 8 bytes". Given `octantSize` is 8, `octetSize` would be 64. However, the heap layout uses `P_NO` increment by 8. We will use the explicit constructor logic: `octetSize = octantSize * 8` [S1, unofficial].*
   - Calculate $firstOffset = VK\_FO$.
   - Calculate $maxSize$ using `npot(firstOffset + sump8(depth))`.
   - Calculate $dimensions = 2^{depth}$.
   - Calculate $startSize = npot(\max(0x10000, \min(maxSize/8, MAX\_BUFFER)))$.
   - Allocate buffer of $startSize$.
   - Initialize kernel with buffer and $depth-1$ [S1, unofficial].

2. **Buffer Expansion:**
   - If memory is needed, calculate $maxAllowed = \min(MAX\_BUFFER, this.maxSize)$.
   - Calculate $newSize = buffer.byteLength \times 2$.
   - If $newSize > maxAllowed$, return false.
   - Create new `ArrayBuffer` of size $newSize$.
   - Transfer contents.
   - Re-initialize kernel with new buffer.
   - Update references. Return true [S1, unofficial].

3. **Traversal (`seek`):**
   - Initialize $TRAVA = 0$.
   - Loop while $D_{CUR} < \min(D_{TGT}, D_{MAX})$:
     - Increment $D_{CUR}$.
     - Calculate $octantPointer$ using current coordinates and depth inverse.
     - Read octant value into $TRAVA$.
     - If $isP(TRAVA)$ is false, return $TRAVA$ (leaf voxel).
     - If $isP(TRAVA)$ is true, update $P_{CUR} = pFrom(TRAVA)$.
   - Return $P_{CUR}$ if loop finishes [S1, unofficial].

4. **Initialization (`initOctet`):**
   - Initialize $INITA = 0$.
   - Loop while $D_{CUR} < \min(D_{TGT}, D_{MAX})$:
     - Increment $D_{CUR}$.
     - Calculate $octantPointer$.
     - Read octant value into $INITA$.
     - If $isP(INITA)$ is false:
       - Allocate new octet (increment $P_{NO}$).
       - Create pointer from allocation.
       - Set octant at $octantPointer$ to this new pointer.
     - Update $P_{CUR} = pFrom(INITA)$.
   - Return $P_{CUR}$ [S1, unofficial].

5. **Step (`step`):**
   - Increment $D_{CUR}$.
   - Calculate $octantPointer$.
   - Read octant value into $STEPA$.
   - Update $P_{CUR} = pFrom(STEPA)$.
   - Return $octantPointer$ [S1, unofficial].

6. **Ray-AABB Intersection (`rayAABB`):**
   - Calculate slabs:
     - $DBLA = (bs_x - ro_x) \times rdx$
     - $DBLB = (be_x - ro_x) \times rdx$
     - $DBLC = (bs_y - ro_y) \times rdy$
     - $DBLD = (be_y - ro_y) \times rdy$
     - $DBLE = (bs_z - ro_z) \times rdz$
     - $DBLF = (be_z - ro_z) \times rdz$ *Note: Source lists DBLD twice for Z, implying a typo in source. Standard slab method uses distinct variables for Z. We assume standard logic: $DBL_Z1 = (bs_z - ro_z) \times rdz$, $DBL_Z2 = (be_z - ro_z) \times rdz$.*
   - Return $1$ if $\max(\min(DBLA, DBLB), \min(DBLC, DBLD), \min(DBL_Z1, DBL_Z2)) \le \min(\max(DBLA, DBLB), \max(DBLC, DBLD), \max(DBL_Z1, DBL_Z2))$, else $0$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voctopus_init(depth, buffer_size=None):
    """Initialize the Voctopus kernel."""
    VK_FO = 100
    VK_OS = 8
    MAX_BUFFER = 1024 * 1024 * 1024 * 512
    
    octant_size = 8
    octet_size = octant_size * 8
    
    # Calculate max size needed for a full octree
    # sump8(n) = (8^(n+1) - 1) / 7
    sum_p8 = (8**(depth + 1) - 1) // 7
    first_offset = VK_FO
    max_size = next_power_of_two(first_offset + sum_p8)
    
    dimensions = 2 ** depth
    
    # Calculate start size
    min_start = 0x10000
    max_start = max_size // 8
    start_size = next_power_of_two(max(min_start, min(max_start, MAX_BUFFER)))
    
    if buffer_size is None:
        buffer_size = start_size
        
    buffer = allocate_buffer(buffer_size)
    
    # Initialize heap indices
    heap = Uint32Array(buffer)
    heap[0] = depth  # D_MAX
    heap[1] = VK_FO  # P_FO
    heap[2] = VK_FO + VK_OS  # P_NO
    
    return Kernel(heap, depth)

def voctopus_seek(kernel, target_coords, target_depth):
    """Traverse to the deepest voxel at target_coords."""
    heap = kernel.heap
    depth = kernel.depth
    
    # Set target depth
    heap[9] = target_depth
    
    # Prepare lookup (set coordinates and depth inverse)
    prepare_lookup(kernel, target_coords, depth)
    
    # Initialize traversal state
    heap[5] = 0  # D_CUR = 0
    heap[3] = heap[1]  # P_CUR = P_FO
    
    # Seek loop
    while heap[5] < min(heap[9], heap[0]):
        increment_current_depth(kernel)
        octant_ptr = octant_pointer(kernel)
        trava = get_octant(heap, octant_ptr)
        
        if not is_pointer(trava):
            return trava  # Leaf voxel data
        
        # Update current pointer
        heap[3] = p_from(trava)
        
    return heap[3]  # Return pointer if target depth reached

def voctopus_init_octet(kernel, target_coords, target_depth):
    """Initialize path to target, creating nodes if necessary."""
    heap = kernel.heap
    depth = kernel.depth
    
    heap[9] = target_depth
    prepare_lookup(kernel, target_coords, depth)
    heap[5] = 0
    heap[3] = heap[1]
    
    while heap[5] < min(heap[9], heap[0]):
        increment_current_depth(kernel)
        octant_ptr = octant_pointer(kernel)
        inita = get_octant(heap, octant_ptr)
        
        if not is_pointer(inita):
            # Allocate new octet
            new_ptr = allocate_octet(kernel)
            new_ptr_encoded = make_pointer(new_ptr)
            set_octant(heap, octant_ptr, new_ptr_encoded)
            inita = new_ptr_encoded
            
        heap[3] = p_from(inita)
        
    return heap[3]

def voctopus_set_voxel(kernel, coords, r, g, b, a):
    """Set a voxel at specific coordinates."""
    depth = kernel.depth
    # Initialize path
    ptr = voctopus_init_octet(kernel, coords, depth)
    # Set value
    val = val_from_rgba(r, g, b, a)
    set_octant(kernel.heap, ptr, val)
    return ptr

def voctopus_get_voxel(kernel, coords, out):
    """Get voxel at specific coordinates."""
    depth = kernel.depth
    # Seek to leaf
    ptr = voctopus_seek(kernel, coords, depth)
    raw = get_octant(kernel.heap, ptr)
    
    out.r = r_from(raw)
    out.g = g_from(raw)
    out.b = b_from(raw)
    out.a = a_from(raw)
    return out

def voctopus_ray_cast(kernel, ray_origin, ray_direction, max_dist):
    """Cast a ray through the octree."""
    # Simplified ray-AABB check for current voxel
    # In a real shader, this would be iterative
    pass

def is_pointer(n):
    return (n & 1) != 0

def p_from(n):
    if not is_pointer(n):
        return 0
    return ((~(0x7FFFFFFF + (n & 1)) & n) >> 1)

def make_pointer(n):
    return (n << 1) | 1

def val_from_rgba(r, g, b, a):
    return (r << 24) + (g << 16) + (b << 8) + ((a & 63) << 4)

def r_from(n):
    return ((n >> 8) & 0xFF)

def g_from(n):
    return (n & 0xFF)

def b_from(n):
    return ((n & 0xFF00) >> 8)

def a_from(n):
    return ((n & 0xF0) >> 4)

def octant_identity(v, dc, dm):
    d = dm - 1 - dc
    return (((v[2] >> d) & 1) << 2) | (((v[1] >> d) & 1) << 1) | ((v[0] >> d) & 1)

def prepare_lookup(kernel, coords, depth):
    heap = kernel.heap
    heap[6] = coords[0]
    heap[7] = coords[1]
    heap[8] = coords[2]
    # D_INV is calculated dynamically based on D_CUR

def octant_pointer(kernel):
    heap = kernel.heap
    dc = heap[5]
    dm = heap[0]
    d_inv = dm - dc
    
    x = heap[6]
    y = heap[7]
    z = heap[8]
    
    # Calculate octant identity
    id = (((z >> d_inv) & 1) << 2) | (((y >> d_inv) & 1) << 1) | ((x >> d_inv) & 1)
    
    # Current pointer is the base for this octet
    p_cur = heap[3]
    
    # Offset within the octet (8 voxels)
    # Note: Source implies octantPointer is an index into the heap relative to P_CUR
    # The exact calculation of octantPointer from P_CUR and ID is not fully detailed
    # but typically involves adding ID * octant_size to the current pointer
    return p_cur + (id * 4)  # Assuming 4-byte indices in heap, or 8-byte voxels?

def increment_current_depth(kernel):
    kernel.heap[5] += 1
    # D_INV is recalculated in octant_pointer

def allocate_octet(kernel):
    heap = kernel.heap
    p_no = heap[2]
    heap[2] = p_no + 8  # Increment by 8 bytes? Or 4 indices?
    return p_no - 8  # Return previous value

def get_octant(heap, index):
    return heap[index >> 2]  # Shift right by 2 for 32-bit access

def set_octant(heap, index, value):
    heap[index >> 2] = value

def next_power_of_two(n):
    n -= 1
    n |= n >> 1
    n |= n >> 2
    n |= n >> 4
    n |= n >> 8
    n |= n >> 16
    n |= n >> 32
    return n + 1
```

## Operating characteristics
- **Time Complexity:** Traversal is $O(D)$ where $D$ is the depth of the tree, as it visits one node per level [S1, unofficial].
- **Memory Complexity:** $O(8^D)$ in the worst case (dense), but sparse in practice. The buffer size is determined by `npot(firstOffset + sump8(depth))` [S1, unofficial].
- **Performance:** Over one million read/write operations per second on commodity hardware, up to 8 million r/w operations per second [S1, unofficial].
- **Strengths:** Small memory footprint, fast traversal in JavaScript/asm.js, direct shader rendering capability [S1, unofficial].
- **Limitations:** Buffer expansion is very expensive and should be avoided [S1, unofficial]. Using `get()` with an invalid index returns garbage data [S1, unofficial]. Using `set()` with an incorrect index will corrupt the octree [S1, unofficial].
- **When to use:** WebGL environments requiring direct voxel rendering without meshing [S1, unofficial].
- **When not to use:** When frequent buffer expansion is expected, or when high-precision floating-point voxel data is required (limited to 32-bit packed values) [S1, unofficial].

## Background
Voctopus was developed by nphyx as a JavaScript implementation of a sparse voxel octree [S1, unofficial]. The goal was to create a data structure small and fast enough to be rendered directly by a shader in WebGL without meshification, using ray tracing techniques [S1, unofficial]. It consists of a low-level kernel written in asm.js and a high-level JavaScript API [S1, unofficial]. The project is still a work in progress, with features like pruning and defragmentation incomplete [S1, unofficial].

## Sources
[S1] nphyx/voctopus, https://github.com/nphyx/voctopus, fetched 2026-10-26 (unofficial)
