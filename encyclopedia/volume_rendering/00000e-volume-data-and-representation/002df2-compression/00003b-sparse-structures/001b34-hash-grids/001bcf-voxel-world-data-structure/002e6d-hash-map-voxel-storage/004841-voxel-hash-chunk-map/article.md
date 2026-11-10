# Voxel Hash Chunk Map

Also called: voxlvoxl

## Overview
The Voxel Hash Chunk Map (voxlvoxl) is a sparse 3D point cloud representation that partitions space into independent, hash-addressed cubic chunks to mitigate memory overhead for large-scale maps [S1, unofficial]. Unlike sibling structures like Rolling Bonxai that enforce a fixed 3x3x3 active window, voxlvoxl uses a hash function for chunk indexing, allowing unlimited degrees of freedom for dynamic summoning and dismissing of regions [S1, unofficial].

## How it works
**Inputs:** 3D points ($V3D$), voxel resolution ($res$), chunk size ($n$, must be power of two).
**Outputs:** A hash map of chunks, where each chunk contains a local voxel grid storing point data and occupancy probabilities.

**Data Structures:**
*   **Chunk:** An independent voxel hash map. Contains a mutex (`chunk_mtx_`) for thread safety, a `ready` flag, and an accessor to the underlying voxel data.
*   **CellT:** Stored in each voxel.
    *   `uint32_t raytrace_id`: Default 0.
    *   `int32_t prob_log`: Default `unknown_prob` (log-odds of 0.5).
    *   `std::array<P3D, 20> points`: Fixed-size array of 3D points.
    *   `int32_t num_points`: Default 0.
*   **Config:**
    *   `occupancy_min_threshold`: Default `logods(0.12)`.
    *   `occupancy_max_threshold`: Default `logods(0.97)`.
    *   `occupancy_threshold`: Default `logods(0.5)`.
    *   `prob_hit`: Default `logods(0.7)`.
    *   `prob_miss`: Default `logods(0.4)`.

**Formulas:**
Conversion between probability $p \in [0,1]$ and fixed-point log-odds:
$$ \text{logods}(p) = \lfloor 10^6 \cdot \ln\left(\frac{p}{1 - p}\right) \rfloor $$
$$ p(\text{logods}) = 1 - \frac{1}{1 + \exp(\text{logods} \cdot 10^{-6})} $$

**Steps:**

1.  **Voxel Coordinate Conversion:** Convert a 3D point $p$ to voxel coordinates. To optimize performance, use SIMD-accelerated floor operations rather than standard `std::floor` [S1, unofficial].
    $$ \text{coord} = \lfloor p \cdot \text{inv\_resolution} \rfloor $$
2.  **Chunk Index Calculation:** Calculate the chunk index from voxel coordinates using bit-shifts. This requires `chunk_size` to be a power of two ($2^k$).
    $$ \text{chunk\_idx} = \left( \frac{\text{coord}.x}{2^k}, \frac{\text{coord}.y}{2^k}, \frac{\text{coord}.z}{2^k} \right) $$
3.  **Ray Iteration:** Iterate through voxels from `key_origin` to `key_end` using a 3D DDA-like algorithm.
    *   Initialize `error = {0,0,0}`, `coord = key_origin`, `delta = key_end - coord`.
    *   Determine `step` signs based on `delta` signs.
    *   Let `max` be the maximum absolute component of `delta`.
    *   Loop $i$ from 0 to `max`:
        *   `error += delta`
        *   If `(error.x << 1) >= max`: `coord.x += step.x`, `error.x -= max`
        *   If `(error.y << 1) >= max`: `coord.y += step.y`, `error.y -= max`
        *   If `(error.z << 1) >= max`: `coord.z += step.z`, `error.z -= max`
        *   Call `func(coord)` [S1, unofficial].
4.  **Radius Search (Single Point):**
    *   Lock `chunk_mtx_`.
    *   Convert point to `voxel_coord` and `chunk_idx`.
    *   If chunk not in map, return empty.
    *   Wait for `chunk->ready` with a 10ms timeout. If timeout, skip chunk [S1, unofficial].
    *   Calculate `search_range = (radius * inv_resolution_) + 1`.
    *   Loop $x, y, z$ from `-search_range` to `search_range`.
    *   Get `val = chunk->accessor->value(query_voxel_coord + {x,y,z})`.
    *   If `val` exists, loop $k$ from 0 to `val->num_points`.
    *   Calculate distance between `val->points[k]` and query point. If distance < radius, add to results.
5.  **Radius Search (Batch Points):**
    *   Lock `chunk_mtx_`.
    *   Create bins mapping `chunk_idx` to list of `(point_index, point)`.
    *   For each point, if not finite, skip. Calculate `voxel_coord` and `chunk_idx`. Add to bins.
    *   Convert bins to `bin_vec`.
    *   Calculate `search_range`.
    *   Parallel loop over `bin_vec`. For each chunk, wait for ready (10ms timeout).
    *   For each point in chunk, loop $x,y,z$ in search range. Check distance. Add to `results[idx]` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

# Constants from Config
UNKNOWN_PROB = int(1e6 * math.log(0.5 / (1.0 - 0.5)))
OCCUPANCY_MIN_THRESHOLD = int(1e6 * math.log(0.12 / (1.0 - 0.12)))
OCCUPANCY_MAX_THRESHOLD = int(1e6 * math.log(0.97 / (1.0 - 0.97)))
OCCUPANCY_THRESHOLD = int(1e6 * math.log(0.5 / (1.0 - 0.5)))
PROB_HIT = int(1e6 * math.log(0.7 / (1.0 - 0.7)))
PROB_MISS = int(1e6 * math.log(0.4 / (1.0 - 0.4)))

def logods(prob: float) -> int:
    """Convert probability to fixed-point log-odds."""
    return int(1e6 * math.log(prob / (1.0 - prob)))

def prob(logods_fixed: int) -> float:
    """Convert fixed-point log-odds back to probability."""
    logods = logods_fixed * 1e-6
    return 1.0 - 1.0 / (1.0 + math.exp(logods))

class CellT:
    def __init__(self):
        self.raytrace_id = 0
        self.prob_log = UNKNOWN_PROB
        self.points = [None] * 20
        self.num_points = 0

class VoxelHashChunkMap:
    def __init__(self, resolution: float, chunk_size: int):
        self.resolution = resolution
        self.inv_resolution = 1.0 / resolution
        self.chunk_size = chunk_size
        # chunk_size must be power of two
        self.chunk_bits = int(math.log2(chunk_size))
        self.chunk_map = {} # Hash map: ChunkIdx -> Chunk object
        self.chunk_mtx = None # Mutex placeholder for thread safety

    def v3d_to_voxel_coord(self, p: np.ndarray) -> np.ndarray:
        """Convert 3D point to voxel coordinates using SIMD-friendly floor."""
        # Using numpy array operations to simulate Eigen::array().floor()
        coord = (p * self.inv_resolution)
        # floor and cast to int32
        return np.floor(coord).astype(np.int32)

    def get_corresponding_chunk_idx(self, coord: np.ndarray) -> tuple:
        """Calculate chunk index using bit-shifts."""
        return (coord[0] >> self.chunk_bits,
                coord[1] >> self.chunk_bits,
                coord[2] >> self.chunk_bits)

    def ray_iterator(self, key_origin: np.ndarray, key_end: np.ndarray, func):
        """Iterate through voxels from key_origin to key_end."""
        if np.array_equal(key_origin, key_end):
            return
        
        if not func(key_origin):
            return

        error = np.array([0, 0, 0], dtype=np.int32)
        coord = key_origin.copy()
        delta = key_end - coord
        step = np.sign(delta).astype(np.int32)
        max_val = np.max(np.abs(delta))

        for _ in range(int(max_val)):
            error += delta
            if (error[0] << 1) >= max_val:
                coord[0] += step[0]
                error[0] -= max_val
            if (error[1] << 1) >= max_val:
                coord[1] += step[1]
                error[1] -= max_val
            if (error[2] << 1) >= max_val:
                coord[2] += step[2]
                error[2] -= max_val
            if not func(coord):
                return

    def radius_search_single(self, point: np.ndarray, radius: float) -> list:
        """Radius search for a single point."""
        # Lock chunk_mtx_ [G]
        
        voxel_coord = self.v3d_to_voxel_coord(point)
        chunk_idx = self.get_corresponding_chunk_idx(voxel_coord)
        
        if chunk_idx not in self.chunk_map:
            return []
        
        chunk = self.chunk_map[chunk_idx]
        # Wait for chunk->ready with 10ms timeout [G]
        if not chunk.ready:
            return [] # Simplified timeout handling

        search_range = int((radius * self.inv_resolution) + 1)
        results = []
        
        # Accessor is not thread-safe [G]
        for x in range(-search_range, search_range + 1):
            for y in range(-search_range, search_range + 1):
                for z in range(-search_range, search_range + 1):
                    query_coord = voxel_coord + np.array([x, y, z])
                    val = chunk.accessor.value(query_coord)
                    if val is not None:
                        for k in range(val.num_points):
                            p = val.points[k]
                            if np.linalg.norm(p - point) < radius:
                                results.append(p)
        return results

    def radius_search_batch(self, points: list, radius: float) -> list:
        """Radius search for a batch of points."""
        # Lock chunk_mtx_ [G]
        
        results = [[] for _ in points]
        bins = {} # chunk_idx -> list of (point_index, point)
        
        for i, p in enumerate(points):
            if not np.isfinite(p).all():
                continue
            voxel_coord = self.v3d_to_voxel_coord(p)
            chunk_idx = self.get_corresponding_chunk_idx(voxel_coord)
            if chunk_idx not in bins:
                bins[chunk_idx] = []
            bins[chunk_idx].append((i, p))
        
        bin_vec = list(bins.items())
        search_range = int((radius * self.inv_resolution) + 1)
        
        # Parallel loop over bin_vec [G]
        for chunk_idx, points_in_chunk in bin_vec:
            if chunk_idx not in self.chunk_map:
                continue
            
            chunk = self.chunk_map[chunk_idx]
            # Wait for chunk->ready with 10ms timeout [G]
            if not chunk.ready:
                continue

            for idx, point in points_in_chunk:
                for x in range(-search_range, search_range + 1):
                    for y in range(-search_range, search_range + 1):
                        for z in range(-search_range, search_range + 1):
                            query_coord = self.v3d_to_voxel_coord(point) + np.array([x, y, z])
                            val = chunk.accessor.value(query_coord)
                            if val is not None:
                                for k in range(val.num_points):
                                    p = val.points[k]
                                    if np.linalg.norm(p - point) < radius:
                                        results[idx].append(p)
        return results
```

## Operating characteristics
*   **Memory:** Mitigates memory usage for large point cloud maps by allowing separate disk I/O and dynamic loading/unloading of regions [S1, unofficial].
*   **Performance:** Voxel index calculation is accelerated by a factor of 3 or more using SIMD operations (`_mm_floor_ps`) via `Eigen::Vector3d::array().floor()` [S1, unofficial]. Chunk index calculation uses bit-shifts, which is efficient but requires chunk size to be a power of two [S1, unofficial].
*   **Concurrency:** Chunk loading is asynchronous. `radiusSearch` waits for `chunk->ready` with a 10ms timeout; if the timeout occurs, the chunk is skipped [S1, unofficial]. The `accessor->value` method is not thread-safe [S1, unofficial].
*   **Strengths:** Independent chunks allow flexible summon/dismiss policies. No coordinate conversion overhead within chunks.
*   **Limitations:** Point lookup/insertion requires calculating both chunk and voxel indices, which adds overhead for millions of points. `std::floor()` is computationally expensive without SIMD mitigation [S1, unofficial].

## Background
The Voxel Hash Chunk Map (voxlvoxl) was developed as an alternative to Rolling Bonxai, which hard-fixes a 3x3x3 window of 27 chunks based on position [S1, unofficial]. voxlvoxl indexes chunks using a hash function (borrowed from niessner2013hashing.pdf) and stores active chunks in a `std::unordered_map` of `Bonxai::CoordT` integer 3-dimensional coordinates [S1, unofficial]. This allows unlimited degrees of freedom for chunk management, including implementing Rolling Bonxai style behavior if desired [S1, unofficial].

## Sources
[S1] goguma426/voxlvoxl, https://github.com/goguma426/voxlvoxl, fetched 2026-10-28 (unofficial)
