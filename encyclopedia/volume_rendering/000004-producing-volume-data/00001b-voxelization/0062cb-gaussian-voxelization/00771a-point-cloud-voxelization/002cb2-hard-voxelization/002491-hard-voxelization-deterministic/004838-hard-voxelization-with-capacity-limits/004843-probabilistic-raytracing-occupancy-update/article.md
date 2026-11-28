# Probabilistic Raytracing Occupancy Update
Also called: Occupancy Log-Odds Update, Raytraced Probability Update

## Overview
This method updates voxel occupancy probabilities by combining direct point insertions with raytracing hits and misses, incrementing log-odds on hits and decrementing on misses, bounded by configurable thresholds [S1, unofficial]. It extends deterministic hard voxelization by introducing probabilistic confidence measures for voxel occupancy, allowing smoother integration of sparse point cloud data into dense voxel grids [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A list of 3D points, a voxel grid with chunks and cells, configuration parameters (`cfg_`), and an origin point for raytracing [S1, unofficial].
- **Output:** Updated voxel grid where cell occupancy probabilities (`prob_log`) are adjusted based on point insertions and raytracing results [S1, unofficial].

**Data Structures:**
- **Chunk:** Contains an accessor for cell values and a state flag (`MODIFIED`) [S1, unofficial].
- **Cell:** Contains `points` (array of 3D points), `num_points` (count), `raytrace_id` (integer identifier for the current raytracing pass), and `prob_log` (log-odds value representing occupancy probability) [S1, unofficial].
- **Configuration (`cfg_`):** Contains `do_raytracing` (boolean), `prob_hit` (log-odds increment per hit), `prob_miss` (log-odds increment per miss), `occupancy_max_threshold`, `occupancy_min_threshold`, and `max_points_per_voxel` [S1, unofficial].

**Numbered Steps:**
1. **Point Insertion:** For each point, check if all coordinates are finite; skip if not [S1, unofficial].
2. Convert the point to voxel coordinates [S1, unofficial].
3. If `cfg_.do_raytracing` is true, store the voxel coordinate in `raytrace_hits_` [S1, unofficial].
4. Determine the chunk index for the voxel coordinate [S1, unofficial].
5. If the chunk is not in the neighbor window, skip the point [S1, unofficial].
6. If the cell exists and `num_points < max_points_per_voxel`, add the point to the cell; otherwise, create a new cell with the point [S1, unofficial].
7. Mark the chunk as `MODIFIED` and increment insertion counters [S1, unofficial].
8. **Raytracing Pass:** If `cfg_.do_raytracing` is true, define a `clear` callback for the ray iterator: for each coordinate along the ray, if the chunk is in the neighbor window and the cell exists with `raytrace_id != raytrace_iter_id_`, set `raytrace_id = raytrace_iter_id_`, update `prob_log = max(prob_log + prob_miss, occupancy_min_threshold)`, and mark the chunk `MODIFIED`; if the cell does not exist, skip it to avoid memory bloat [S1, unofficial].
9. Iterate through `raytrace_hits_`: for each target coordinate `to`, raytrace from the origin to `to` using a `RayIterator`, invoking the `clear` callback for every cell along the path (misses) [S1, unofficial].
10. After the raytrace, update the target cell at `to` directly: if it exists and `raytrace_id != raytrace_iter_id_`, set `raytrace_id = raytrace_iter_id_`, update `prob_log = min(prob_log + prob_hit, occupancy_max_threshold)`, and mark the chunk `MODIFIED` [S1, unofficial].
11. Clear `raytrace_hits_` and increment `raytrace_iter_id_` after processing all hits [S1, unofficial].

**Formulas:**
- For hits: $prob\_log_{new} = \min(prob\_log_{old} + prob\_hit, occupancy\_max\_threshold)$ [S1, unofficial].
- For misses: $prob\_log_{new} = \max(prob\_log_{old} + prob\_miss, occupancy\_min\_threshold)$ [S1, unofficial].

**Parameters:**
- `prob_miss`: Log-odds increment per ray miss (decreases probability); value not specified in the code [S1, unofficial].
- `prob_hit`: Log-odds increment per ray hit; value not specified in the code [S1, unofficial].
- `occupancy_max_threshold`: Maximum `prob_log` value; value not specified in the code [S1, unofficial].
- `occupancy_min_threshold`: Minimum `prob_log` value; value not specified in the code [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Redundant Updates:** `raytrace_iter_id_` prevents redundant updates within the same raytracing pass; cells must be checked against it before updating [S1, unofficial].
- **Missing Cells:** The RayIterator callback must handle missing cells gracefully; the code logs an error but does not create cells to avoid memory bloat [S1, unofficial].
- **Precision:** Log-odds values can become very large or small; thresholds should be set appropriately to prevent overflow or underflow [G].
- **Boundary Conditions:** Points outside the neighbor window are ignored; the window size must be sufficient to capture relevant data [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from dataclasses import dataclass
from typing import List, Tuple, Dict, Any

@dataclass
class Coord:
    x: int
    y: int
    z: int

@dataclass
class Cell:
    points: List[Tuple[float, float, float]]
    num_points: int
    raytrace_id: int
    prob_log: float

@dataclass
class Chunk:
    class State:
        MODIFIED = 1
    state: int
    accessor: Dict[Coord, Cell]

def update_occupancy(points: List[Tuple[float, float, float]], origin: Tuple[float, float, float],
                     cfg: Dict[str, Any], chunk_map: Dict[Coord, Chunk],
                     current_source_chunk_idx: Coord, raytrace_hits: List[Coord],
                     raytrace_iter_id: List[int], cell_size: float = 1.0, chunk_size: int = 16) -> Tuple[int, int, int]:
    insert_count = 0
    out_of_range_count = 0
    rejected_count = 0

    for p in points:
        if not (math.isfinite(p[0]) and math.isfinite(p[1]) and math.isfinite(p[2])):
            continue

        voxel_coord = Coord(int(p[0] / cell_size), int(p[1] / cell_size), int(p[2] / cell_size))
        if cfg["do_raytracing"]:
            raytrace_hits.append(voxel_coord)

        point_chunk_idx = Coord(voxel_coord.x // chunk_size, voxel_coord.y // chunk_size, voxel_coord.z // chunk_size)
        if not (abs(point_chunk_idx.x - current_source_chunk_idx.x) <= 1 and
                abs(point_chunk_idx.y - current_source_chunk_idx.y) <= 1 and
                abs(point_chunk_idx.z - current_source_chunk_idx.z) <= 1):
            out_of_range_count += 1
            continue

        if point_chunk_idx not in chunk_map:
            chunk_map[point_chunk_idx] = Chunk(state=0, accessor={})
        point_chunk = chunk_map[point_chunk_idx]

        if voxel_coord in point_chunk.accessor:
            val = point_chunk.accessor[voxel_coord]
            if val.num_points < cfg["max_points_per_voxel"]:
                val.points.append(p)
                val.num_points += 1
            else:
                rejected_count += 1
        else:
            point_chunk.accessor[voxel_coord] = Cell(points=[p], num_points=1, raytrace_id=0, prob_log=0.0)

        point_chunk.state = Chunk.State.MODIFIED
        insert_count += 1

    if cfg["do_raytracing"]:
        def clear(coord: Coord) -> bool:
            chunk_idx = Coord(coord.x // chunk_size, coord.y // chunk_size, coord.z // chunk_size)
            if abs(chunk_idx.x - current_source_chunk_idx.x) <= 1 and \
               abs(chunk_idx.y - current_source_chunk_idx.y) <= 1 and \
               abs(chunk_idx.z - current_source_chunk_idx.z) <= 1:
                if chunk_idx in chunk_map:
                    chunk = chunk_map[chunk_idx]
                    if coord in chunk.accessor:
                        val = chunk.accessor[coord]
                        if val.raytrace_id != raytrace_iter_id[0]:
                            val.raytrace_id = raytrace_iter_id[0]
                            val.prob_log = max(val.prob_log + cfg["prob_miss"], cfg["occupancy_min_threshold"])
                            chunk.state = Chunk.State.MODIFIED
            return True

        origin_coord = Coord(int(origin[0] / cell_size), int(origin[1] / cell_size), int(origin[2] / cell_size))
        for to in raytrace_hits:
            chunk_idx = Coord(to.x // chunk_size, to.y // chunk_size, to.z // chunk_size)
            if abs(chunk_idx.x - current_source_chunk_idx.x) <= 1 and \
               abs(chunk_idx.y - current_source_chunk_idx.y) <= 1 and \
               abs(chunk_idx.z - current_source_chunk_idx.z) <= 1:
                if chunk_idx in chunk_map:
                    chunk = chunk_map[chunk_idx]
                    # RayIterator from origin_coord to to, calling clear for each cell along the path [S1, unofficial]
                    # Standard DDA/Bresenham step omitted; implement per source requirement

                    if to in chunk.accessor:
                        val = chunk.accessor[to]
                        if val.raytrace_id != raytrace_iter_id[0]:
                            val.raytrace_id = raytrace_iter_id[0]
                            val.prob_log = min(val.prob_log + cfg["prob_hit"], cfg["occupancy_max_threshold"])
                            chunk.state = Chunk.State.MODIFIED

        raytrace_hits.clear()
        raytrace_iter_id[0] += 1

    return insert_count, out_of_range_count, rejected_count
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of points and raytracing hits. Raytracing adds overhead proportional to the number of rays and their lengths [S1, unofficial].
- **Memory Complexity:** Linear with respect to the number of voxels and points. The `raytrace_hits_` list grows with the number of points [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of points and the size of the neighbor window. Parallelization is noted as a work-in-progress [S1, unofficial].
- **Strengths:** Provides probabilistic occupancy information, which can be useful for applications requiring confidence measures [S1, unofficial].
- **Limitations:** Not intended for real-time performance [S1, unofficial]. Memory usage can increase if cells are created for missing data (though the current implementation avoids this) [S1, unofficial].
- **When to Use:** When probabilistic occupancy information is needed and real-time performance is not a critical requirement [S1, unofficial].
- **When Not to Use:** When real-time performance is required or when memory usage is a significant constraint [S1, unofficial].

## Background
This method originates from an unofficial implementation of voxel-based localization and mapping [S1, unofficial]. It builds on the concept of hard voxelization with capacity limits, adding probabilistic updates via raytracing [S1, unofficial].

## Sources
[S1] goguma426/voxlvoxl, https://github.com/goguma426/voxlvoxl, fetched 2026-10-28 (unofficial)
