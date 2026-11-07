# Grid Topology Generation (Edge Building)

Also called: Edge Indexing, Topology Construction

## Overview
This method generates the edge connectivity for the extracted mesh in a 2D Dual Contouring implementation by determining which grid vertices are connected based on cell intersection counts and tile boundaries [S1, unofficial]. It operates on a tiled grid structure, using shared memory reductions and parallel prefix sums to efficiently compute global edge indices for a CUDA kernel [S1, unofficial]. It differs from its sibling [Dual Contouring with NDC Constraint](4410-dual-contouring-with-ndc-constraint.md) by focusing specifically on the low-level parallel topology construction for standard 2D grid cells without additional non-directional constraint handling [S1, unofficial].

## How it works
**Inputs:**
- `ibuffer`: A device memory array storing vertex indices. Layout: `int` array of size `(n - 1) * TILE_HEIGHT * 2`, initialized to `-1` [S1, unofficial]. It stores indices for the current, left, and down vertices relative to the processing tile [S1, unofficial].
- `n`: Number of mesh grid vertices along one axis [S1, unofficial].
- `TILE_WIDTH`, `TILE_HEIGHT`: Compile-time constants defining CUDA block dimensions [S1, unofficial].
- `ecnt`: Device integer pointer for global edge count, initialized to 0 [S1, unofficial].
- `edges`: Device integer array for output edges, capacity `ecap` [S1, unofficial].

**Outputs:**
- `edges`: Array of integers storing edge connections. Each edge is defined by two vertex indices. The array is filled sequentially based on computed global offsets [S1, unofficial].
- `ecnt`: Updated global count of generated edges [S1, unofficial].

**Data Structures:**
- `s[BLOCK_SIZE]`: Shared memory array used for parallel reduction and prefix sum. Indexed by `ti = threadIdx.y * TILE_WIDTH + threadIdx.x` [S1, unofficial]. `BLOCK_SIZE` equals `TILE_WIDTH * TILE_HEIGHT`.

**Steps:**
1. **Fetch Indices:** Each thread fetches the vertex indices for the current cell (`self`), the left neighbor (`left`), and the down neighbor (`down`) from `ibuffer`. If a neighbor is outside the grid or tile boundary, the index is `-1` [S1, unofficial].
2. **Count Connections:** Compute the number of valid connections for the current cell:
   $$ count = (self \neq -1) \times ((left \neq -1) + (down \neq -1)) $$
   If `self == -1`, `count` is 0 [S1, unofficial].
3. **Shared Memory Reduction:** Perform a warp-level or block-level reduction in shared memory to count how many threads have `count == 1` and `count == 2`:
   $$ total1 = \text{__syncthreads\_count}(count == 1) $$
   $$ total2 = \text{__syncthreads\_count}(count == 2) $$
   The total edges contributed by this tile is:
   $$ total = total1 + total2 \times 2 $$
   [S1, unofficial].
4. **Global Offset:** Compute the global starting index for this tile's edges using an atomic add:
   $$ base = \text{atomicAdd}(ecnt, total) $$
   [S1, unofficial].
5. **Prefix Sum:** Execute a parallel prefix sum (scan) on the `count` values stored in `s` to determine local edge indices. The local edge index for a thread is:
   $$ ei = base + s[ti] - count $$
   [S1, unofficial].
6. **Write Edges:** Write the edges to the `edges` array based on `ei`:
   - If `left != -1`, write edge `(self, left)` at `edges[ei]` and `edges[ei + (left != -1 ? 1 : 0)]`? No, the source specifies: "first edge connects `self` to `left` (or `down`), second edge connects `self` to `down`". Specifically:
     - If `left != -1`, write `edges[ei] = self`, `edges[ei + 1] = left`. Increment `ei` by 1 if only one edge, or handle sequentially. The source says: "Write edges to `edges` array: first edge connects `self` to `left` (or `down`), second edge connects `self` to `down`." This implies a specific order. If `count == 1`, write one edge. If `count == 2`, write two edges. The index `ei` is the start position.
     - Implementation detail: If `left != -1`, write `(self, left)`. If `down != -1`, write `(self, down)`. The `ei` calculated is the base index for the *first* edge of the cell. If `count=1`, `ei` is the index for that single edge. If `count=2`, `ei` is the index for the first edge, and `ei+1` for the second [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inside CUDA kernel KBuildTopology
ti = threadIdx.y * TILE_WIDTH + threadIdx.x
# Fetch indices from ibuffer based on tileY and thread position
# self, left, down are integers, -1 if out of bounds
count = 0
if self != -1:
    if left != -1:
        count += 1
    if down != -1:
        count += 1

# Store count in shared memory for reduction
s[ti] = count
__syncthreads()

# Reduction to count total1 (count==1) and total2 (count==2)
# Note: __syncthreads_count is a hypothetical helper for reduction
# In practice, this requires a full reduction kernel or warp shuffle logic
# Assuming a simplified reduction for the article:
total1 = 0
total2 = 0
for i in range(BLOCK_SIZE):
    if s[i] == 1:
        total1 += 1
    elif s[i] == 2:
        total2 += 1

# Compute global edge offset
base = atomicAdd(ecnt, total1 + total2 * 2)

# Parallel prefix sum on s to get local offsets
# This requires a scan operation. Assuming s is modified in-place or a separate array is used.
# For simplicity, we assume a standard inclusive scan on the count values
# Let local_offset be the result of the scan for this thread's count
# The source formula: ei = base + s[ti] - count
# This implies s[ti] holds the prefix sum value *before* subtraction or is the rank.
# Re-reading source: "Execute parallel prefix sum (scan) on s array to determine local edge indices ei = base + s[ti] - count"
# This suggests s[ti] is the prefix sum of counts up to ti.
# If s[ti] is the inclusive prefix sum, then the number of edges before this thread is s[ti] - count.
# So ei = base + (s[ti] - count)

# Write edges
idx = base + s[ti] - count
if left != -1:
    edges[idx] = self
    edges[idx + 1] = left
    if down != -1:
        idx += 1
if down != -1:
    edges[idx] = self
    edges[idx + 1] = down
```
*Note: The pseudocode above simplifies the CUDA-specific reduction and scan. A real implementation would use warp shuffles or a dedicated scan kernel. The formula `ei = base + s[ti] - count` is taken directly from [S1, unofficial].*

**Parameters:**
- `TILE_WIDTH`, `TILE_HEIGHT`: Typical values depend on GPU architecture, often powers of 2 (e.g., 16x16, 32x32) [S1, unofficial].

**Edge Cases:**
- Boundary vertices: Indices are `-1` and must be handled to avoid writing invalid edges [S1, unofficial].
- Degenerate cells: If `self` is `-1`, `count` is 0, and no edges are written [S1, unofficial].

## Subtopics

### DCx Mesh Generation Pipeline
DCx (Dual Contouring over Expanded Cubes) extends this topology generation to 3D UDFs. It applies a thinning algorithm to refine active cubes predicted by generative models like TRELLIS.2 [S2]. It constructs mesh topology by connecting dual vertices via a V2M (Voxel-to-Mesh) look-up table over expanded cubes [S2]. It includes a post-processing step to eliminate residual topological noise [S2]. Unlike Faithful Contouring (FC), which uses standard QEF and suffers from artifacts, DCx uses an optimization-based projection strategy and the V2M lookup table, yielding cleaner topology with fewer spurious non-manifold artifacts and topological holes [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of grid cells, $O(N)$, due to the parallel nature of the reduction and prefix sum [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the `edges` array and $O(BLOCK\_SIZE)$ for shared memory `s` [S1, unofficial].
- **Performance:** Efficient on CUDA hardware due to coalesced memory access and shared memory usage [S1, unofficial].
- **Strengths:** High parallelism, handles tile boundaries gracefully, avoids race conditions via atomic operations [S1, unofficial].
- **Limitations:** Requires sufficient grid resolution to resolve details; lower resolutions may lead to topological ambiguities [S2].

## Background
This method originates from the `cudc` project, an unofficial CUDA implementation of Dual Contouring for 2D grids [S1, unofficial]. It is part of the pipeline for extracting polygonal meshes from scalar fields defined on 2D grids [S1, unofficial].

## Sources
[S1] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
[S2] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
