# Evaluation Cache

Also called: CachedEval, EvalCache

## Overview
Evaluation Cache is a memory optimization technique that eliminates redundant computation by storing previously computed field values and their derivatives at shared grid locations. It differs from its siblings, which focus on spatial data layout (e.g., [Bricking](914-bricking.md)) or hardware texture sampling (e.g., [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md)), by operating at the algorithmic level to cache the *results* of expensive evaluations rather than optimizing the *access* to the source data.

## How it works
The method maps unique geometric entities (typically grid corners or edges) to a cache entry containing the computed scalar value and optional gradient. By using a quantized key representing the global grid position, the cache ensures that shared vertices between adjacent cells are evaluated only once.

**Inputs and Outputs**
- Input: Grid origin, cell size, and the evaluation function $f(\mathbf{x})$ returning a scalar and optionally a gradient vector.
- Output: A cache structure mapping grid positions to computed values.

**Data Structures**
The implementation uses two primary structures [S1, unofficial]:

1.  **GridPos**: A quantized integer key representing a unique corner in the global grid.
2.  **CachedEval**:
    - `value`: `f64` (64-bit float) storing the scalar field value.
    - `gradient`: `Option<[f64; 3]>` storing the 3D gradient vector. `None` if the gradient is not required or computed.
3.  **EvalCache**:
    - `entries`: A hash map (`HashMap<GridPos, CachedEval>`) where the key is the quantized grid position.

**Algorithm Steps**

1.  **Quantize Grid Position**: For a given cell corner, compute a unique integer key.
    Let $\mathbf{c} = (c_x, c_y, c_z)$ be the integer coordinates of the cell containing the corner.
    Let $corner\_idx \in \{0, 1, 2, 3, 4, 5, 6, 7\}$ be the local corner index within the cell (e.g., 0 for 0,0,0; 1 for 1,0,0, etc.).
    The quantized grid position $P$ is computed as:
    $$ P = c_x + 2c_y + 4c_z + 8 \cdot corner\_idx $$
    *Note: The source [S1, unofficial] specifies adding corner_idx bits (1 for X, 2 for Y, 4 for Z) to the cell's integer coordinates. This implies a specific bit-packing scheme. A robust interpretation for a unique global key is:*
    $$ P = c_x + 2c_y + 4c_z + 8 \cdot corner\_idx $$
    *However, strictly following [S1, unofficial]'s "add corner_idx bits... to cell's integer coordinates" suggests:*
    $$ P = c_x + 2c_y + 4c_z + corner\_idx $$
    *This is ambiguous if cell coordinates are large. The standard interpretation for "quantized grid position" in this context (Manifold Dual Contouring) is a unique global vertex ID. We assume the key is a unique integer identifier for the vertex.*
    [G] To ensure uniqueness for large grids, the key should typically be a 64-bit integer combining cell coordinates and corner index, e.g., $P = (c_x \ll 32) | (c_y \ll 16) | (c_z \ll 8) | corner\_idx$. The source [S1, unofficial] implies a simpler additive scheme likely valid for small or local grids. We implement the additive scheme as described but note the potential for collision in large grids.

2.  **Check Cache**: Look up $P$ in `EvalCache.entries`.
    - If present, return `entry.value` and `entry.gradient`.
    - If absent, proceed to step 3.

3.  **Evaluate**: Compute the world-space coordinate $\mathbf{x}$ for the corner.
    $$ \mathbf{x} = \mathbf{origin} + \mathbf{P}_{grid} \cdot \text{cell\_size} $$
    Where $\mathbf{P}_{grid}$ is the continuous grid position corresponding to the corner (derived from cell coordinates and corner offset).
    Compute $v = f(\mathbf{x})$ and $\mathbf{g} = \nabla f(\mathbf{x})$.

4.  **Store**: Insert $(P, \text{CachedEval}(v, \mathbf{g}))$ into `EvalCache.entries`.

5.  **Return**: Return $v$ and $\mathbf{g}$.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import Tuple, Optional, Dict

class CachedEval:
    def __init__(self, value: float, gradient: Optional[Tuple[float, float, float]]):
        self.value = value
        self.gradient = gradient

class EvalCache:
    def __init__(self):
        self.entries: Dict[int, CachedEval] = {}

    def quantize_grid_pos(self, cell_coords: Tuple[int, int, int], corner_idx: int) -> int:
        """
        Computes a unique integer key for a grid corner.
        Based on [S1, unofficial]: add corner_idx bits (1 for X, 2 for Y, 4 for Z) to cell's integer coordinates.
        Note: This additive scheme assumes small cell coordinates or local grids.
        """
        cx, cy, cz = cell_coords
        # Using the additive scheme from [S1, unofficial]
        return cx + 2 * cy + 4 * cz + corner_idx

    def get_or_eval(self, cache: EvalCache, origin: Tuple[float, float, float], 
                    cell_size: float, cell_coords: Tuple[int, int, int], 
                    corner_idx: int, 
                    eval_func: callable) -> Tuple[float, Optional[Tuple[float, float, float]]]:
        """
        Retrieves cached value or evaluates and caches it.
        """
        key = cache.quantize_grid_pos(cell_coords, corner_idx)
        
        if key in cache.entries:
            entry = cache.entries[key]
            return entry.value, entry.gradient
        
        # Compute world-space coordinate
        # [S1, unofficial]: add the product of the grid position component and cell_size to the origin component.
        # The grid position component for the corner is derived from cell_coords and corner_idx.
        # Assuming corner_idx 0-7 maps to offsets (0,0,0) to (1,1,1).
        # Bit decomposition of corner_idx:
        x_bit = (corner_idx >> 0) & 1
        y_bit = (corner_idx >> 1) & 1
        z_bit = (corner_idx >> 2) & 1
        
        gx = cell_coords[0] + x_bit
        gy = cell_coords[1] + y_bit
        gz = cell_coords[2] + z_bit
        
        world_x = origin[0] + gx * cell_size
        world_y = origin[1] + gy * cell_size
        world_z = origin[2] + gz * cell_size
        
        # Evaluate function
        value, gradient = eval_func(world_x, world_y, world_z)
        
        # Store in cache
        cache.entries[key] = CachedEval(value, gradient)
        
        return value, gradient
```

**Parameters**
- `cell_size`: The physical size of a single grid cell. Typical values depend on the dataset resolution and domain size.
- `origin`: The world-space coordinate of the grid's (0,0,0) corner.

**Edge Cases and Pitfalls**
- **Key Collisions**: The additive quantization scheme in [S1, unofficial] may produce collisions if cell coordinates are large. A bit-shifted key is safer for general use [G].
- **Memory Usage**: The cache grows with the number of unique corners. For large volumes, this may exceed available memory.
- **Gradient Storage**: Storing gradients doubles the memory footprint per entry. If gradients are not needed (e.g., simple scalar field evaluation), `gradient` should be `None`.

## Subtopics

### Volume Access Caching
A variant where the cache stores the result of texture sampling operations. In iterative volume rendering (e.g., pre-integration), the back sample of one iteration is cached and reused as the front sample of the next. This ensures the number of texture accesses remains $N$ regardless of the integration order, reducing computational cost by avoiding redundant fetches [S2].

## Operating characteristics
- **Time Complexity**: $O(1)$ average case for cache hits; $O(1)$ for cache misses (evaluation). Total time is $O(V)$ where $V$ is the number of unique vertices.
- **Memory Complexity**: $O(V)$, where $V$ is the number of unique grid corners. Each entry stores a scalar and optionally a 3D vector.
- **Strengths**: Eliminates redundant mathematical evaluations (e.g., trilinear interpolation, gradient computation). Particularly effective in algorithms like Marching Cubes or Dual Contouring where corners are shared by multiple cells.
- **Limitations**: Requires additional memory for the hash map. Hash map overhead can be significant for very large grids.
- **When to use**: When the evaluation function is expensive and corners are shared. Not suitable if memory is extremely constrained or if the evaluation is trivial (e.g., direct array lookup).

## Background
Evaluation Cache is used in isosurface extraction algorithms, specifically Manifold Dual Contouring, to optimize the computation of corner values and gradients [S1, unofficial]. It is also applied in volume rendering to cache texture samples [S2].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
