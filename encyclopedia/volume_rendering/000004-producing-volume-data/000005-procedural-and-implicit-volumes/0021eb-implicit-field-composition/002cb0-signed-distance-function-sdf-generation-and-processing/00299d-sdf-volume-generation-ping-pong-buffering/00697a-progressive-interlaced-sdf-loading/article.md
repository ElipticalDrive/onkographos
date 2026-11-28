# Progressive Interlaced SDF Loading

Also called: Interlaced Loading, Multi-pass Progressive SDF

## Overview
Progressive Interlaced SDF Loading is a technique to progressively fill a 3D Signed Distance Field (SDF) texture by iterating through voxels in multiple passes with decreasing resolution (increasing density), allowing for interactive preview updates before the full resolution is computed [S1, unofficial]. It differs from standard sequential loading by decoupling the visual feedback loop from the full-resolution computation, enabling the renderer to display a coarse approximation of the SDF immediately while the fine details are calculated in subsequent passes [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs**:
    *   `bounding_box`: An array of two `Vector3<f32>` representing the minimum and maximum corners of the SDF domain [S1, unofficial].
    *   `max_voxels_side`: The maximum dimension size for the voxel grid, used to scale other dimensions to maintain aspect ratio [S1, unofficial].
    *   `loading_passes`: The number of passes for the interlaced loading. Default is 3 if resetting [S1, unofficial].
    *   `max_delta_time`: The maximum time allowed for the update loop to run, ensuring interactive framerate [S1, unofficial].
    *   `sdf_surface`: An object implementing the `SDFSurface` trait, providing the distance function and a `changed()` method returning an `Option` of bounding box [S1, unofficial].
*   **Outputs**:
    *   `CpuTexture3D`: A 3D texture containing the SDF data, updated incrementally.
    *   `update_count`: The number of voxels sampled/updated in the current frame [S1, unofficial].

### Data Structures

**LoadingManager**
Manages the iteration state across passes.
*   `step_size`: `u32`. The stride for voxel traversal. Initialized to $2^{passes-1}$.
*   `next_index`: `Vector3<u32>`. The current voxel index $(x, y, z)$ being visited.
*   `iterations`: `u32`. The number of iterations completed in the current pass.
*   `total_iterations`: `u32`. The total number of iterations completed across all passes.
*   `limits`: `Vector3<u32>`. The dimensions of the voxel grid $(width, height, depth)$.

**CpuTexture3D**
*   `name`: `String`.
*   `data`: `Vec<[f32; 4]>`. A 1D vector of RGBA floats. Size is $width \times height \times depth$. Initialized with `AIR_DIST` [S1, unofficial].
*   `width`, `height`, `depth`: `u32`.
*   `min_filter`, `mag_filter`: `Interpolation`. Set to `Nearest` for blocky visualization [S1, unofficial].
*   `wrap_s`, `wrap_t`, `wrap_r`: `Wrapping`. Set to `MirroredRepeat` [S1, unofficial].

**SDFViewer**
*   `tex0`: `CpuTexture3D`. The primary texture being written to.
*   `tex1`: `CpuTexture3D`. A secondary texture (ping-pong buffer), though the primary update logic writes to `tex0` [S1, unofficial].
*   `volume`: `Rc<RefCell<Gm<Mesh, SDFViewerMaterial>>>`. The mesh and material for rendering.
*   `loading_mgr`: `LoadingManager`.
*   `bounding_box`: `[Vector3<f32>; 2]`.
*   `changed_box`: `Option<[Vector3<f32>; 2]>`.
*   `changed_box_while_loading`: `bool`.
*   `ctx`: `three_d::Context`.

### Parameters
*   `AIR_DIST`: `f32`. Default value for uncomputed SDF values. Set to $0.1 + 0.001234 = 0.101234$. This value is small to avoid graphical artifacts but not too small to slow down rendering [S1, unofficial].
*   `max_voxels_side`: `usize`. Maximum dimension size for the voxel grid.
*   `loading_passes`: `usize`. Number of passes. Default 3 [S1, unofficial].
*   `max_delta_time`: `f64`. Time limit per update loop [S1, unofficial].

### Algorithm Steps

1.  **Initialization**:
    *   Calculate voxel dimensions (`limits`) from `bounding_box` and `max_voxels_side`. The dimension with the largest size in the bounding box is set to `max_voxels_side`, and others are scaled proportionally [S1, unofficial].
    *   Initialize `CpuTexture3D` with `AIR_DIST` for all voxels.
    *   Initialize `LoadingManager` with `limits` and `loading_passes`. Call `reset(passes)`.

2.  **LoadingManager Reset**:
    *   Set `step_size` to $2^{passes-1}$. If `passes` is 0 or negative, use `max(1, passes)` to ensure at least one pass [S1, unofficial].
    *   Set `next_index` to $(0, 0, 0)$.
    *   Set `iterations` to 0.
    *   Set `total_iterations` to 0.

3.  **Update Loop (Per Frame)**:
    *   Check if `sdf_surface` has changed via `changed()`. If changed, merge bounding boxes and reset `LoadingManager` with 3 passes [S1, unofficial].
    *   Record start time using `instant::Instant` [S1, unofficial].
    *   While `time_elapsed < max_delta_time` AND `LoadingManager` is not idle:
        *   Get next voxel index `v` from `LoadingManager.next()`.
        *   If `v` is `None`, break loop.
        *   Calculate flat index:
            $$ \text{flat\_index} = (v.z \times \text{height} + v.y) \times \text{width} + v.x $$
            where `width`, `height`, `depth` are the texture dimensions [S1, unofficial].
        *   Compute SDF value at `v` using `sdf_surface`.
        *   Update `tex0.data[flat_index]` with the computed value (stored as RGBA, likely with alpha 1.0 or similar, though specific channel mapping is not detailed beyond `RgbaF32` panic check) [S1, unofficial].
        *   Increment `update_count`.
        *   Check time limit.

4.  **LoadingManager Iterator (`next`)**:
    *   If `step_size` is 0, return `None` [S1, unofficial].
    *   Increment `iterations` and `total_iterations`.
    *   Return `next_index`.
    *   Advance `next_index.x` by `step_size`.
    *   If `next_index.x >= limits.x`:
        *   Reset `next_index.x` to 0.
        *   Increment `next_index.y` by `step_size`.
        *   If `next_index.y >= limits.y`:
            *   Reset `next_index.y` to 0.
            *   Increment `next_index.z` by `step_size`.
            *   If `next_index.z >= limits.z`:
                *   Decrement `step_size` to the previous power of 2: `step_size = prev_power_of_2(step_size - 1)` [S1, unofficial].
                *   If `step_size` becomes 0, the next call to `next()` will return `None` [S1, unofficial].
                *   Reset `next_index` to $(0, 0, 0)$.
                *   Reset `iterations` to 0.

5.  **Helper: `prev_power_of_2`**:
    *   Returns the largest power of 2 less than or equal to the input. For input 0, it returns 0 [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time
from dataclasses import dataclass
from typing import Optional, List, Tuple

# Assuming Vector3 and SDFSurface are defined elsewhere or standard
# Vector3: x, y, z fields
# SDFSurface: method distance(v: Vector3) -> float, method changed() -> Optional[Vector3, Vector3]

def prev_power_of_2(n: int) -> int:
    """Returns the largest power of 2 less than or equal to n. Returns 0 if n <= 0."""
    if n <= 0:
        return 0
    # Bitwise operation to find highest set bit
    p = 1
    while p * 2 <= n:
        p *= 2
    return p

@dataclass
class LoadingManager:
    limits: Tuple[int, int, int]  # (width, height, depth)
    step_size: int
    next_index: Tuple[int, int, int]
    iterations: int
    total_iterations: int
    passes: int

    def reset(self, passes: int):
        # Ensure at least one pass
        effective_passes = max(1, passes)
        self.passes = effective_passes
        # step_size = 2^(passes-1)
        self.step_size = 1 << (effective_passes - 1)
        self.next_index = (0, 0, 0)
        self.iterations = 0
        self.total_iterations = 0

    def next(self) -> Optional[Tuple[int, int, int]]:
        if self.step_size == 0:
            return None
        
        self.iterations += 1
        self.total_iterations += 1
        
        x, y, z = self.next_index
        w, h, d = self.limits
        
        # Advance x
        x += self.step_size
        if x >= w:
            x = 0
            y += self.step_size
            if y >= h:
                y = 0
                z += self.step_size
                if z >= d:
                    # End of current pass, decrease resolution
                    self.step_size = prev_power_of_2(self.step_size - 1)
                    if self.step_size == 0:
                        return None
                    x = 0
                    y = 0
                    z = 0
                    self.iterations = 0
        
        self.next_index = (x, y, z)
        return (x, y, z)

    def passes_left(self) -> int:
        # log2 of step_size + 1
        if self.step_size == 0:
            return 0
        # Assuming step_size is power of 2
        import math
        return int(math.log2(self.step_size)) + 1

    def len(self) -> int:
        # Remaining iterations calculation
        # Sum of steps for each pass
        # This is a simplified approximation based on source description
        # Exact implementation requires summing steps_per_dim for each step_size
        return 0 # Placeholder for complex calculation

class SDFViewer:
    def __init__(self, bounding_box: List[Vector3], max_voxels_side: int, loading_passes: int = 3):
        self.bounding_box = bounding_box
        self.max_voxels_side = max_voxels_side
        self.loading_passes = loading_passes
        
        # Calculate limits
        self.limits = self._calculate_limits(bounding_box, max_voxels_side)
        
        # Initialize Texture
        self.tex0 = CpuTexture3D(self.limits)
        
        # Initialize Loading Manager
        self.loading_mgr = LoadingManager(self.limits, 0, (0,0,0), 0, 0, loading_passes)
        self.loading_mgr.reset(loading_passes)
        
        self.ctx = None # Context reference
        self.sdf_surface = None # SDFSurface reference

    def _calculate_limits(self, bb: List[Vector3], max_side: int) -> Tuple[int, int, int]:
        # Calculate dimensions based on bounding box size and max_voxels_side
        # The dimension with the largest size in the bounding box is set to max_voxels_side
        # Others are scaled proportionally
        min_v = bb[0]
        max_v = bb[1]
        size = Vector3(max_v.x - min_v.x, max_v.y - min_v.y, max_v.z - min_v.z)
        
        max_dim = max(size.x, size.y, size.z)
        
        if max_dim == 0:
            return (1, 1, 1)
            
        scale = max_side / max_dim
        w = int(size.x * scale)
        h = int(size.y * scale)
        d = int(size.z * scale)
        
        # Ensure at least 1 voxel per dimension
        w = max(1, w)
        h = max(1, h)
        d = max(1, d)
        
        return (w, h, d)

    def update(self, sdf_surface: 'SDFSurface', max_delta_time: float) -> int:
        self.sdf_surface = sdf_surface
        
        # Check if SDF has changed
        changed_box = sdf_surface.changed()
        if changed_box is not None:
            # Merge bounding boxes and reset
            # For simplicity, assume we reset with 3 passes if changed
            self.loading_mgr.reset(3)
            
        if self.loading_mgr.step_size == 0:
            return 0
            
        start_time = time.time()
        updates = 0
        
        while True:
            current_time = time.time()
            if current_time - start_time > max_delta_time:
                break
                
            voxel = self.loading_mgr.next()
            if voxel is None:
                break
                
            x, y, z = voxel
            w, h, d = self.limits
            
            # Calculate flat index
            flat_index = (z * h + y) * w + x
            
            # Compute SDF value
            # Note: voxel coordinates are in grid space, need to map to world space if sdf_surface expects world coords
            # Assuming sdf_surface.distance takes grid coords or voxel is already in correct space
            # The source notes imply direct usage of voxel index for texture update
            # We assume sdf_surface can handle the coordinate system or voxel is transformed
            # For implementation completeness, we assume a helper to get world pos exists or sdf_surface is grid-aware
            # Since source doesn't specify coordinate transformation, we assume direct distance call
            try:
                dist = sdf_surface.distance(Vector3(x, y, z))
            except Exception:
                # Fallback if coordinate system mismatch, but source implies direct usage
                dist = 0.0 
            
            # Update texture data
            # Texture data is Vec<[f32; 4]>, RGBA
            # Source notes panic if not RgbaF32
            if len(self.tex0.data) <= flat_index:
                break # Should not happen if limits are correct
                
            # Store distance in R channel, others as 1.0 or 0.0 depending on convention
            # Source doesn't specify channel mapping, assuming R=distance, GBA=1.0
            self.tex0.data[flat_index] = [dist, 1.0, 1.0, 1.0]
            
            updates += 1
            
        return updates

class CpuTexture3D:
    def __init__(self, limits: Tuple[int, int, int]):
        self.width, self.height, self.depth = limits
        self.data = [0.101234] * (self.width * self.height * self.depth) # AIR_DIST
        self.min_filter = "Nearest"
        self.mag_filter = "Nearest"
        self.wrap_s = "MirroredRepeat"
        self.wrap_t = "MirroredRepeat"
        self.wrap_r = "MirroredRepeat"
```

### Edge Cases and Pitfalls
*   **Texture Format**: The texture data must be `RgbaF32`. If not, the code panics with 'developer error: expected RgbaF32 texture data' [S1, unofficial].
*   **Step Size Zero**: If `step_size` is 1, `step_size - 1` is 0. `prev_power_of_2(0)` returns 0, causing the next pass to terminate immediately. This is handled by the check `if step_size == 0: return None` in `next()` [S1, unofficial].
*   **Bounding Box Consistency**: The code assumes the SDF surface is consistent and the `changed()` method returns accurate bounding boxes [S1, unofficial].
*   **SDF Surface Modification**: The code assumes the SDF surface is not modified during the update process [S1, unofficial].
*   **Time Limit**: The update loop ensures at least one iteration is performed if needed, even if the time limit is reached [S1, unofficial].

## Operating characteristics

*   **Time Complexity**: The number of iterations in the loading manager is proportional to the total number of voxels divided by the step size, summed over all passes. Specifically, for $P$ passes, the total number of voxels visited is $\sum_{i=0}^{P-1} \frac{N}{2^i}$, where $N$ is the total number of voxels. This results in $O(N)$ total work over all passes, but $O(N/2^k)$ work in pass $k$ [S1, unofficial].
*   **Memory Complexity**: $O(N)$ for the 3D texture data, where $N = width \times height \times depth$. Each voxel stores 4 floats [S1, unofficial].
*   **Measured Performance**: The method is designed for interactive performance, using `max_delta_time` to limit the work per frame. Test cases include cube sizes $2\times2\times2$, $8\times8\times8$, $64\times64\times64$, non-cube $11\times11\times11$, and non-cube $8\times11\times17$ [S1, unofficial].
*   **Strengths**: Provides immediate visual feedback with a coarse SDF, improving user experience during loading. Reduces initial latency.
*   **Limitations**: Requires multiple passes to achieve full resolution. The ping-pong buffer (`tex1`) is present but not fully utilized in the primary update logic described, potentially leading to confusion about its purpose [S1, unofficial].
*   **When to Use**: Interactive SDF viewers where immediate preview is valuable.
*   **When Not to Use**: Batch processing where full resolution is required immediately, or when memory bandwidth is severely constrained and multiple texture copies are prohibitive.

## Background

*   **Origin**: This method is implemented in the `yeicor/sdf-viewer` project, an unofficial tool for visualizing Signed Distance Fields [S1, unofficial].
*   **History**: The technique leverages the concept of progressive rendering, adapted for 3D voxel grids. By starting with a sparse grid (large step size) and refining it, the viewer can display a recognizable shape quickly.
*   **Usage**: Used in the `SDFViewer` application to manage the loading of SDF textures, allowing users to interact with the scene while the full resolution SDF is being computed in the background [S1, unofficial].

## Sources

[S1] yeicor/sdf-viewer, https://github.com/yeicor/sdf-viewer, fetched 2026-10-27 (unofficial)
