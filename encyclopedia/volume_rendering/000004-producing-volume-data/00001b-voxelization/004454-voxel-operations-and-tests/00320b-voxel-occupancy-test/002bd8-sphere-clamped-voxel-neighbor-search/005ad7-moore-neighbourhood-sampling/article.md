# Moore Neighbourhood Sampling

Also called: 3x3x3 Voxel Neighborhood Read, 27-Voxel Block Access

## Overview
Moore Neighbourhood Sampling reads the values of all 27 voxels in a 3x3x3 block centered on a target voxel, returning both the data and a bitmask indicating which neighbors are non-empty. It serves as the foundational data access primitive for local voxel operations, such as the Sphere-Clamped Voxel Neighbor Search. It is distinguished from its sibling [PBF Displacement Calculation with Grid-Based Neighbor Search](9026-pbf-displacement-calculation-with-grid-based-neighbor-search.md) by its fixed, exhaustive 27-neighbor scope rather than a physics-based radius search.

## How it works
**Inputs:**
- `voxels`: A flat array of voxel data, indexed linearly.
- `center`: The 3D integer coordinates $(v_x, v_y, v_z)$ of the central voxel.
- `VOXEL_EMPTY`: The sentinel value representing an empty voxel.

**Outputs:**
- `result`: An array of 27 voxel values.
- `mask`: A 32-bit unsigned integer bitmask.

**Data Structures:**
The output is encapsulated in a `MooreNeighbourhood` struct:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct MooreNeighbourhood:
    neighbours: array<u32, 27>  # Values of the 27 voxels
    mask: u32                   # Bitmask: bit i is 1 if neighbours[i] != VOXEL_EMPTY
```

**Linear Indexing Logic:**
The 27 neighbors are indexed linearly from $i = 0$ to $26$. The mapping between linear index $i$ and 3D offset $(dx, dy, dz)$ is defined as follows:

1.  **Linear to Offset:**
    Given linear index $i$:
    $$ z = \lfloor i / 9 \rfloor $$
    $$ y = \lfloor (i \pmod 9) / 3 \rfloor $$
    $$ x = i \pmod 3 $$
    The offset vector is $(x-1, y-1, z-1)$.

2.  **Offset to Linear:**
    Given offset vector $v = (v_x, v_y, v_z)$ where $v_x, v_y, v_z \in \{-1, 0, 1\}$:
    $$ \text{linear\_index} = u32(v_x + 1) + 3 \cdot u32(v_y + 1) + 9 \cdot u32(v_z + 1) $$
    Here, $u32(\cdot)$ denotes casting to unsigned 32-bit integer to handle potential negative values in intermediate calculations if implemented in languages with signed arithmetic, though in Python-like pseudocode, standard integer arithmetic suffices.

**Algorithm Steps:**
1.  Initialize `neighbours` array of size 27 and `mask` to 0.
2.  For each $i$ from 0 to 26:
    a. Calculate offset $(dx, dy, dz)$ using the Linear to Offset formula.
    b. Calculate neighbor position $u = (v_x + dx, v_y + dy, v_z + dz)$.
    c. Calculate linear index $idx$ for $u$ using the Offset to Linear formula.
    d. Read `value = voxels[idx]`.
    e. Set `neighbours[i] = value`.
    f. If `value != VOXEL_EMPTY`, set bit $i$ in `mask` (i.e., `mask |= (1 << i)`).
3.  Return `MooreNeighbourhood(neighbours, mask)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def moore_neighbourhood_sampling(voxels, center, voxel_empty):
    neighbours = [0] * 27
    mask = 0
    
    for i in range(27):
        # Linear index i to 3D offset
        z = i // 9
        y = (i % 9) // 3
        x = i % 3
        
        # Offset vector relative to center
        dx = x - 1
        dy = y - 1
        dz = z - 1
        
        # Neighbor position
        ux = center[0] + dx
        uy = center[1] + dy
        uz = center[2] + dz
        
        # Neighbor position to linear index
        # Assuming voxels are stored in Z-major order with stride 9 for Z, 3 for Y, 1 for X
        # The formula provided in sources: u32(v.x + 1) + 3 * u32(v.y + 1) + 9 * u32(v.z + 1)
        # This maps offsets -1,0,1 to indices 0,1,2 within the 3x3x3 block
        linear_idx = (ux - center[0] + 1) + 3 * (uy - center[1] + 1) + 9 * (uz - center[2] + 1)
        
        # Read value
        # Note: In a real implementation, one must ensure ux, uy, uz are within bounds of the voxels array
        # before accessing voxels[global_linear_index(ux, uy, uz)]. 
        # The source implies a local lookup within the 3x3x3 block structure.
        # We assume a helper function global_index exists or that voxels is padded/bounded.
        # Based on source [S1, unofficial], the linear index calculation is for the 3x3x3 block mapping.
        # The actual array access would be:
        # global_idx = linear_index_3d_to_1d(ux, uy, uz, dimensions)
        # value = voxels[global_idx]
        
        # For the purpose of this method definition as per source [S1, unofficial]:
        # The source describes the mapping logic. We assume valid bounds checking is handled externally or voxels are padded.
        value = voxels[linear_idx] # Placeholder for actual global indexing logic if not local block
        
        neighbours[i] = value
        
        if value != voxel_empty:
            mask |= (1 << i)
            
    return MooreNeighbourhood(neighbours, mask)
```
*[G] The source [S1, unofficial] provides the mapping logic for the 3x3x3 block but does not explicitly detail the global array bounds checking or the global-to-linear conversion for the `voxels` array itself. An implementer must add boundary checks for `ux, uy, uz` against the volume dimensions before accessing the global `voxels` array.*

## Subtopics

### Von Neumann Neighbourhood Extraction
Extracts the 6 face-adjacent voxels from a Moore neighbourhood result. It returns the values at specific indices corresponding to the $\pm x, \pm y, \pm z$ directions. The indices in the 27-element Moore array are `[12, 14, 10, 16, 4, 22]`, corresponding to directions `-x, x, -y, y, -z, z` respectively [S1, unofficial]. A precomputed bitmask `VON_NEUMANN_MASK` (value `4281360u`) has bits 12, 14, 10, 16, 4, and 22 set, allowing for efficient filtering of the Moore mask to identify which face neighbors are non-empty [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(1)$, as it always accesses exactly 27 voxels.
**Memory Complexity:** $O(1)$, requiring storage for 27 values and one 32-bit integer.
**Strengths:** Provides complete local context for voxel-based algorithms; the bitmask allows for efficient conditional processing of only occupied neighbors.
**Limitations:** Accesses empty voxels unnecessarily if the neighborhood is sparse; requires boundary handling for voxels at the edge of the volume.

## Background
Originating from implementations in GPU-based voxel terrain rendering, this method is documented in the `wgpu-voxel-terrain` repository [S1, unofficial]. It is used as a low-level primitive for spatial queries in voxel grids, enabling higher-level operations like neighbor searches for physics or rendering.

## Sources
[S1] statusfailed/wgpu-voxel-terrain, https://github.com/statusfailed/wgpu-voxel-terrain, fetched 2026-10-27 (unofficial)
