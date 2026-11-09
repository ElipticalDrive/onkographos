# Voctopus Sparse Voxel Octree

Also called: Voctopus

## Overview
Voctopus is a sparse voxel octree (SVO) implementation designed for direct shader rendering in WebGL via ray tracing, eliminating the need for meshification [S1, unofficial]. It distinguishes itself by using a flat, contiguous memory buffer managed by a low-level `asm.js` kernel to achieve high throughput (up to 8 million read/write operations per second) while maintaining a small memory footprint [S1, unofficial].

## How it works
**Inputs and Outputs:**
The primary input is the `depth` parameter, defining the maximum resolution of the octree ($2^{depth}$). The output is a `Voctopus` instance containing a memory buffer, a kernel object for low-level operations, and high-level API methods for voxel access.

**Data Structures:**
1.  **Buffer:** A contiguous `ArrayBuffer` storing voxel data. The size is determined dynamically, starting at `startSize` and expanding up to `MAX_BUFFER` (512 GB) [S1, unofficial].
2.  **Voxel (32-bit):** Each voxel is a single 32-bit integer with the following bit layout [S1, unofficial]:
    *   Bits 0–23: RGB color (24 bits).
    *   Bits 24–27: Alpha channel (4 bits).
    *   Bits 28–30: Reserved for rendering instructions (3 bits).
    *   Bit 31: Type flag. If set, the entry is a pointer to a child octant; if clear, it is voxel data.
3.  **Octant:** A single voxel node at a specific depth.
4.  **Octet:** An array of 8 voxels (children of a parent node). In the kernel's internal representation, an octet occupies 8 bytes (one 32-bit word per voxel) [S1, unofficial].
5.  **Kernel:** An `asm.js` module handling memory management, traversal, and bit-packing. It exposes methods like `getOctant`, `setOctant`, `prepareLookup`, and `initOctet` [S1, unofficial].

**Parameters:**
*   `depth`: Maximum tree depth. Minimum useful depth is 2 [S1, unofficial].
*   `MAX_BUFFER`: Constant, $1024 \times 1024 \times 1024 \times 512$ bytes (512 GB) [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Validate `depth`.
    *   Calculate `octantSize` = 8 bytes.
    *   Calculate `octetSize` = `octantSize` * 8.
    *   Calculate `firstOffset` from `VK_FO` (kernel-specific constant) [S1, unofficial].
    *   Calculate `maxSize` using `npot(firstOffset + sump8(depth))` [S1, unofficial].
    *   Calculate `dimensions` = $2^{depth}$.
    *   Calculate `startSize` = `npot(max(0x10000, min(maxSize/8, MAX_BUFFER)))` [S1, unofficial].
    *   Allocate `buffer` of size `startSize`.
    *   Initialize `kernel` with `buffer` and `depth-1` [S1, unofficial].

2.  **Buffer Expansion (Triggered when memory is exhausted):**
    *   Calculate `maxAllowedSize` = `min(MAX_BUFFER, this.maxSize)`.
    *   Calculate `newSize` = `buffer.byteLength` * 2.
    *   If `newSize` > `maxAllowedSize`, return `false` [S1, unofficial].
    *   Create `tmp` buffer of size `newSize`.
    *   Transfer contents from `buffer` to `tmp`.
    *   Re-initialize `kernel` with `tmp`.
    *   Update `buffer` and view references.
    *   Return `true` [S1, unofficial].
    *   *Note:* Buffer expansion is a very expensive operation and should be avoided until necessary [S1, unofficial].

3.  **Traversal and Access:**
    *   **Prepare Lookup:** `kernel.prepareLookup(x, y, z, depth)` calculates the path from the root to the target coordinate [S1, unofficial].
    *   **Walk Tree:** Initialize a stack with the start pointer (`kernel.getFirstOffset()`). Loop `i` from 0 to `depth-1`: call `step()` to get the next child pointer. If a child exists, push to stack and update pointer; else return stack [S1, unofficial].
    *   **Get Voxel:** Call `kernel.getOctant(p)` to get raw data. Extract R, G, B, A using `kernel.rFrom()`, `gFrom()`, `bFrom()`, `aFrom()` [S1, unofficial].
    *   **Set Voxel:** Pack RGBA into a 32-bit value using `kernel.valFromRGBA(r, g, b, a)`. Write to `kernel.setOctant(p, rgba)` [S1, unofficial].
    *   **Initialize Voxel:** Call `kernel.prepareLookup()` then `kernel.initOctet()` to ensure the path exists and return the index [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Voctopus:
    def __init__(self, depth):
        # 1. Validate depth
        if depth < 2:
            raise ValueError("Minimum useful tree depth is 2")
        
        self.depth = depth
        self.octantSize = 8
        self.octetSize = self.octantSize * 8
        
        # 2. Calculate sizes (npot = next power of two)
        # VK_FO and sump8 are kernel-specific constants/functions
        firstOffset = get_vk_fo() 
        maxSize = npot(firstOffset + sump8(depth))
        self.dimensions = 2 ** depth
        
        startSize = npot(max(0x10000, min(maxSize // 8, MAX_BUFFER)))
        
        # 3. Allocate buffer
        self.buffer = ArrayBuffer(startSize)
        
        # 4. Initialize kernel
        self.kernel = Kernel(self.buffer, depth - 1)
        
    def expand(self):
        maxAllowedSize = min(MAX_BUFFER, self.maxSize)
        newSize = self.buffer.byteLength * 2
        
        if newSize > maxAllowedSize:
            return False
            
        tmp = ArrayBuffer(newSize)
        # Transfer buffer contents to tmp
        copy_buffer_contents(self.buffer, tmp)
        
        # Re-initialize kernel with tmp
        self.kernel.reinit(tmp)
        
        # Update buffer reference
        self.buffer = tmp
        return True

    def get_voxel(self, index, out):
        raw = self.kernel.get_octant(index)
        out.r = self.kernel.r_from(raw)
        out.g = self.kernel.g_from(raw)
        out.b = self.kernel.b_from(raw)
        out.a = self.kernel.a_from(raw)
        return out

    def set_voxel(self, index, r, g, b, a):
        rgba = self.kernel.val_from_rgba(r, g, b, a)
        self.kernel.set_octant(index, rgba)
        return index

    def init_voxel(self, v):
        # v is [x, y, z]
        self.kernel.prepare_lookup(v[0], v[1], v[2], self.depth)
        index = self.kernel.init_octet()
        return index

    def walk_tree(self, v):
        stack = [self.kernel.get_first_offset()]
        self.kernel.prepare_lookup(v[0], v[1], v[2], self.depth)
        
        for i in range(self.depth - 1):
            c = self.kernel.step()
            if c:
                stack.append(c)
                # Update current pointer logic handled by kernel state or explicit p=c
                # Note: The source implies p is updated internally or via return
        return stack

    def get_voxel_at_coord(self, v, out):
        self.kernel.prepare_lookup(v[0], v[1], v[2], self.depth)
        ptr = self.kernel.seek()
        return self.get_voxel(ptr, out)

    def set_voxel_at_coord(self, v, props):
        self.kernel.prepare_lookup(v[0], v[1], v[2], self.depth)
        index = self.kernel.init_octet()
        return self.set_voxel(index, props.r, props.g, props.b, props.a)

    def set_octet(self, v, data):
        # data is array of 8 voxel objects
        p = self.init_voxel(v)
        for i in range(8):
            self.set_voxel(p + i, data[i].r, data[i].g, data[i].b, data[i].a)

    def get_octet(self, v, out):
        # out is array of 8 voxel objects
        p = self.seek(v)
        for i in range(8):
            self.get_voxel(p + i, out[i])
        return out

    def ray_cast_accumulator(self, ray):
        # f(t, p) => bool
        # t: distance traveled, p: pointer to voxel hit
        # Returns true if at least one voxel was hit
        hit = False
        # Implementation depends on shader/ray marching logic
        # This is a conceptual accumulator function
        return hit
```

**Edge Cases and Pitfalls:**
*   **Buffer Expansion:** Expanding the buffer is extremely expensive. It should only be triggered when necessary [S1, unofficial].
*   **Invalid Index:** Calling `get()` with an invalid index returns garbage data [S1, unofficial].
*   **Corruption:** Calling `set()` with an incorrect index will corrupt the octree structure [S1, unofficial].
*   **Precision:** The voxel format uses 24-bit RGB and 4-bit alpha. Precision is limited by this fixed-point representation [S1, unofficial].

## Operating characteristics
*   **Performance:** Over one million read/write operations per second on commodity hardware, with benchmarks showing up to 8 million operations per second [S1, unofficial].
*   **Memory:** Small memory footprint due to sparse representation and contiguous buffer layout. Maximum buffer size is capped at 512 GB [S1, unofficial].
*   **Strengths:** High throughput for direct shader rendering; no meshification required; efficient memory usage for sparse data.
*   **Limitations:** Buffer expansion is costly; fixed 32-bit voxel format limits color/alpha precision; still considered a work in progress with unstable API [S1, unofficial].
*   **When to use:** WebGL applications requiring real-time ray-traced volume rendering without preprocessing to meshes.
*   **When not to use:** Applications requiring high-precision floating-point voxel data or dynamic frequent buffer resizing.

## Background
Voctopus is a JavaScript implementation of a sparse voxel octree created by nphyx [S1, unofficial]. It was designed to enable direct shader rendering in WebGL using ray tracing techniques, including cone tracing, path tracing, and depth marching [S1, unofficial]. The project consists of a low-level `asm.js` kernel for performance and a high-level JavaScript API [S1, unofficial]. As of the source date, it was a work in progress with an unstable API and data structure [S1, unofficial].

## Sources
[S1] nphyx/voctopus, https://github.com/nphyx/voctopus, fetched 2026-10-26 (unofficial)
