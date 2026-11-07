# Parallel Ray Casting with Segment Lists


## Overview
Parallel Ray Casting with Segment Lists is a volume rendering acceleration method that combines spatial decomposition of the image plane with ray-space culling via segment lists to skip empty space [S1, unofficial]. It distributes rendering tasks across multiple processors (MPI ranks) and threads (OpenMP), calculating sampling points based on precomputed segment boundaries rather than iterating every voxel [S1, unofficial]. This approach differs from [Parallel Image Compositing with Overlapping](3554-parallel-image-compositing-with-overlapping.md) by handling visibility culling at the ray level before compositing, and from [Ray Compaction](4039-ray-compaction.md) by retaining the full ray structure but skipping empty intervals explicitly via list iteration rather than compacting active rays into a dense array [S1, unofficial].

## How it works
**Inputs:**
- Image dimensions: `width`, `height`.
- Camera parameters: `cam_pos` (position), `ray_dir` (direction vector), `aspect` (aspect ratio).
- Volume data: `dims` (volume dimensions vector), `buf_base` (base buffer pointer), block structure with null checks.
- Configuration: `sampling_rate` (default 1.0), `comm_size` (number of MPI ranks), `thread_nums` (number of OpenMP threads, default 4).

**Outputs:**
- Rendered image pixels (RGB values).

**Data Structures:**
- `raytracer` struct: Contains `m_start_x`, `m_start_y`, `m_end_x`, `m_end_y`, `m_width`, `m_height`, `m_depth`, `m_writer`, `m_threadId` [S2, unofficial].
- `seg_list`: A list of segments, where each segment is defined by start and end points `(s_beg, s_end)` along the ray [S1, unofficial].
- `point_list`: A list of sampling points along the ray [S1, unofficial].

**Step Length Calculation:**
The step length for sampling is calculated as:
$$ \text{step\_len} = \frac{1.0}{\text{sampling\_rate} \times \|\text{dims}\|} $$
where $\|\text{dims}\|$ is the Euclidean norm of the volume dimensions vector [S1, unofficial].

**Precomputation:**
Block offsets for Memory Management Unit (MMU) access are precomputed by iterating over brick dimensions $(k, j, i)$. For each block:
- If the block is null, the offset is $-1$.
- Otherwise, the offset is $\text{block} \to \text{data} - \text{buf\_base}$ [S1, unofficial].

**Parallel Execution Steps:**
1.  **Job Distribution:**
    - Calculate total line numbers: $\text{line\_nums} = \lceil \text{height} / \text{comm\_size} \rceil$.
    - Calculate thread line numbers: $\text{t\_line\_nums} = \lceil \text{line\_nums} / \text{thread\_nums} \rceil$ [S1, unofficial].
    - If multi-threading is enabled, job count is $\text{config::core\_count} \times \text{config::core\_thread\_count}$; otherwise, it is 1 [S2, unofficial].

2.  **Thread Loop:**
    - For each thread with ID `t_id` in the parallel region:
        - Iterate $i$ from $0$ to $\text{t\_line\_nums} - 1$.
        - Calculate row index: $\text{row} = \text{my\_rank} + (\text{t\_id} \times \text{t\_line\_nums} + i) \times \text{comm\_size}$.
        - If $\text{row} \ge \text{height}$, break [S1, unofficial].

3.  **Ray Casting per Row:**
    - For each column `col` in the assigned row:
        - Calculate pixel coordinates:
          $$ u = \left(-0.5 + \frac{\text{col} + 0.5}{\text{width}}\right) \times \text{aspect} $$
          $$ v = -0.5 + \frac{\text{row} + 0.5}{\text{height}} $$
        - Calculate `ray_dir` using camera orientation functions (e.g., `cal_ray_dir`), then normalize it [S1, unofficial].
        - Get ray distance to volume bounding box: $\text{seg} = \text{volume} \to \text{getRayDistance}(\text{cam\_pos}, \text{ray\_dir})$ [S1, unofficial].
        - Get segment list for this ray: $\text{volume} \to \text{getSegList}(*\text{seg\_list}, \text{cam\_pos}, \text{ray\_dir})$ [S1, unofficial].
        - Clear `point_list` [S1, unofficial].

4.  **Sampling Point Generation:**
    - For each segment $(s\_beg, s\_end)$ in `seg_list`:
        - Align $s\_beg$ to the step grid:
          $$ s\_beg = \text{seg}.x + \left\lceil \frac{s\_beg - \text{seg}.x}{\text{step\_len}} \right\rceil \times \text{step\_len} $$
        - While $s\_beg \le s\_end$:
            - Push $s\_beg$ to `point_list`.
            - $s\_beg \leftarrow s\_beg + \text{step\_len}$ [S1, unofficial].

5.  **Compositing:**
    - Iterate over `point_list` to sample volume data and composite the final color for the pixel (implementation details for compositing formula not provided in sources) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def parallel_ray_casting_segment_lists(width, height, cam_pos, ray_dir_func, volume, config):
    comm_size = config.comm_size
    thread_nums = config.thread_nums
    sampling_rate = config.sampling_rate
    dims = volume.dims
    
    # Precompute step length
    dims_norm = math.sqrt(dims.x**2 + dims.y**2 + dims.z**2)
    step_len = 1.0 / (sampling_rate * dims_norm)
    
    # Precompute block offsets
    block_offsets = {}
    for k in range(volume.bricks.k):
        for j in range(volume.bricks.j):
            for i in range(volume.bricks.i):
                block = volume.get_block(k, j, i)
                if block is None:
                    block_offsets[(k,j,i)] = -1
                else:
                    block_offsets[(k,j,i)] = block.data - volume.buf_base

    # Calculate distribution
    line_nums = math.ceil(height / comm_size)
    t_line_nums = math.ceil(line_nums / thread_nums)
    
    # Parallel region (conceptual)
    for t_id in range(thread_nums):
        for i in range(t_line_nums):
            row = config.my_rank + (t_id * t_line_nums + i) * comm_size
            if row >= height:
                break
            
            for col in range(width):
                # Calculate ray
                u = (-0.5 + (col + 0.5) / width) * config.aspect
                v = -0.5 + (row + 0.5) / height
                ray_dir = ray_dir_func(u, v)
                ray_dir = ray_dir / math.linalg.norm(ray_dir)
                
                # Get segment info
                seg = volume.get_ray_distance(cam_pos, ray_dir)
                seg_list = volume.get_seg_list(cam_pos, ray_dir)
                point_list = []
                
                # Generate sampling points
                for s_beg, s_end in seg_list:
                    # Align s_beg to step grid
                    # Note: seg.x is assumed to be the ray origin or start of segment
                    aligned_beg = seg.x + math.ceil((s_beg - seg.x) / step_len) * step_len
                    
                    current = aligned_beg
                    while current <= s_end:
                        point_list.append(current)
                        current += step_len
                
                # Composite pixel (implementation not in source)
                # color = composite(volume, cam_pos, ray_dir, point_list)
                # image.set_pixel(row, col, color)
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point alignment of $s\_beg$ may introduce small errors; ensure `step_len` calculation is stable [S1, unofficial].
- **Boundaries:** The row calculation must strictly check `row >= height` to avoid out-of-bounds access [S1, unofficial].
- **Degenerate Input:** If `ray_dir` is zero or `dims.norm()` is zero, the method fails; guards are required [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the average number of segments per ray. Skipping empty space reduces $S$ significantly compared to full voxel traversal.
- **Memory Complexity:** $O(B)$ for block offsets and $O(R)$ for segment lists, where $B$ is the number of blocks and $R$ is the number of rays.
- **Performance:** Depends heavily on the sparsity of the volume. Dense volumes offer less benefit from segment lists. Parallelism scales with `comm_size` and `thread_nums` [S1, unofficial].
- **Strengths:** Efficiently skips empty space; simple parallelization via image rows.
- **Limitations:** Load balancing may be uneven if some rays have significantly more segments than others. Not given in the sources.

## Background
Originated in the context of parallel volume rendering implementations using OpenMP and MPI [S1, unofficial]. The method is part of a broader family of image-order rendering techniques that prioritize ray-space culling.

## Sources
[S1] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S2] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
