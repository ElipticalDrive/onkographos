# Ray Packet BVH Traversal

Also called: occluded, closest (packet), shadow ray packet traversal, primary ray packet traversal, Ray Packet Traversal (Closest Hit)

## Overview
Ray Packet BVH Traversal traverses a Wald-ordered Bounding Volume Hierarchy (BVH) for a packet of rays to find the closest intersection for each active ray in the packet, or to determine which rays are occluded by triangles [S1, unofficial]. It differs from single-ray traversal by processing multiple rays simultaneously, leveraging SIMD-like parallelism within the traversal stack and leaf intersection tests. It builds on [Bounding Volume Hierarchy (BVH) Traversal](18487-bounding-volume-hierarchy-bvh-traversal.md) by extending the node visitation logic to handle a range of ray indices (`first` to `first + packet_size`) rather than a single ray.

## How it works
**Inputs:**
- `bvhtree`: A Wald-ordered BVH structure.
- `p`: Ray packet data containing origin, direction, and flags.
- `hit`: Output structure for intersection results.
- `flags`: Bitwise combination of `raypacket::INTERVALARITH`, `raypacket::SHAREDORG`, and others determining optimization paths [S1, unofficial].

**Outputs:**
- `hit`: Intersection data for the closest hit for each ray in the packet.
- `occnum`: Count of occluded rays (in shadow ray variants).

**Data Structures:**
- `stack`: Array of `pair<intersector::node*, u32>` with fixed size 64. Each element stores a BVH node pointer and the starting ray index (`first`) for that node in the packet [S1, unofficial].
- `active`: Array of integers (or bitmasks) indicating which rays in the current packet are still active (have not yet hit an object). Size is `MAXRAYNUM` [S1, unofficial].
- `signs`: Array of 3 integers (0 or 1) indicating the sign of each ray direction component: `signs[i] = 1` if `dir[i] >= 0`, else `0` [S1, unofficial].

**Parameters:**
- `MAXRAYNUM`: Maximum number of rays in a packet. Not given in the sources.
- `INTERVALARITH`: Flag enabling interval arithmetic optimizations.
- `SHAREDORG`: Flag indicating all rays in the packet share the same origin.

**Steps:**

1. **Initialize Traversal State:**
   - Compute sign bits for ray direction components:
     ```python
     signs[0] = 1 if p.dir().x >= 0 else 0
     signs[1] = 1 if p.dir().y >= 0 else 0
     signs[2] = 1 if p.dir().z >= 0 else 0
     ```
   - Initialize stack with root node and starting ray index 0:
     ```python
     stack[0] = (bvhtree.root, 0)
     stacksz = 1
     occnum = 0  # For shadow ray variants
     ```

2. **Traversal Loop:**
   - While `stacksz > 0`:
     - Pop element from stack: `elem = stack[--stacksz]`
     - Extract `node = elem.first` and `first = elem.second`

3. **Interval Arithmetic Check (if `INTERVALARITH` flag set):**
   - If `SHAREDORG` is set:
     - Call `slaboneco(node, p)`. If it returns true, jump to `processnode`.
     - Call `culliaco(node, p)`. If it returns true, break loop (no rays intersect this node).
   - Else (not `SHAREDORG`):
     - Call `slabone(node, p)`. If it returns true, jump to `processnode`.
     - Call `cullia(node, p)`. If it returns true, break loop.

4. **First Slab Intersection Check:**
   - If `SHAREDORG` is set:
     - Call `slabfirstco(node, p)`. If it returns false, break loop (no rays intersect this node).
   - Else:
     - Call `slabfirst(node, p)`. If it returns false, break loop.

5. **Process Node (`processnode` label):**
   - Get node flag: `flag = node.getflag()`
   - If `flag == NONLEAF`:
     - Get split axis and offset:
       ```python
       axis = node.getaxis()
       offset = node.getoffset()
       farindex = signs[axis]
       nearindex = farindex ^ 1
       ```
     - Push far child to stack:
       ```python
       stack[stacksz] = (node + offset + farindex, first)
       stacksz += 1
       ```
     - Move to near child:
       ```python
       node = node + offset + nearindex
       ```
     - Continue loop (process near child)
   - If `flag == TRILEAF`:
     - Get triangle pointer: `tris = node.getptr<waldtriangle>()`
     - Get triangle count: `n = tris.num`
     - Initialize active array: `active[first] = 1` (and other rays as needed, typically all 1 initially)
     - Filter active rays within node's box:
       - If `SHAREDORG`: `slabfilterco(active, first, node, p)`
       - Else: `slabfilter(active, first, node, p)`
     - Loop over triangles `i` from 0 to `n-1`:
       - Call `closest<flags>(tris[i], p, active, first, hit)`
       - If shadow ray variant: `occnum += result`
       - If `occnum == p.raynum`: return immediately (all rays occluded)
     - Break loop (leaf processed)
   - If `flag` is neither `NONLEAF` nor `TRILEAF` (sub-BVH leaf):
     - Get pointer to child intersector: `node = node.getptr<intersector>().root`
     - Jump to `processnode`

6. **Dispatch to Specialized Implementation:**
   - Initialize extra data: `extra = initextra(p, hit)`
   - Switch on flags, calling `closest<X>(bvhtree, p, extra, hit)` for `X` in `{0, 4, 8, 12}` [S1, unofficial].

**Edge Cases and Pitfalls:**
- Unreachable code path if flags do not match 0, 4, 8, or 12. Handled by assert [S1, unofficial].
- Precision issues in slab intersection tests may cause rays to be incorrectly culled. Interval arithmetic flags help mitigate this.
- Stack overflow if BVH depth exceeds 64 levels. Not given in the sources.

## Operating characteristics
- **Time Complexity:** $O(N \cdot \log M)$ on average, where $N$ is the number of rays in the packet and $M$ is the number of nodes in the BVH. The packet size $N$ provides parallelism, but traversal depth remains logarithmic.
- **Memory Complexity:** $O(D)$ for stack depth $D$, where $D$ is the BVH depth. Fixed stack size of 64 elements [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with packet size up to SIMD width (e.g., 4, 8, 16 rays). Beyond that, diminishing returns due to divergence in ray paths.
- **Strengths:** Reduced traversal overhead per ray due to packet processing; efficient culling using slab tests on the entire packet.
- **Limitations:** Divergence in ray paths within a packet can lead to wasted work; stack size limit may require dynamic stack for deep BVHs.
- **When to Use:** When rendering with many rays (e.g., primary rays, shadow rays) and hardware supports SIMD-like parallelism.
- **When Not to Use:** For very small packets (single ray) or highly divergent ray paths where overhead of packet management outweighs benefits.

## Background
Origin: Implementation details from the mini.q engine [S1, unofficial]. History and development: Part of the mini.q ray tracer's BVH traversal system, optimized for Wald-ordered BVHs. Where it is used: In the mini.q engine for rendering scenes with BVH-accelerated ray tracing [S1, unofficial].

## Sources
[S1] bsegovia/mini.q, https://github.com/bsegovia/mini.q, fetched 2026-10-26 (unofficial)
