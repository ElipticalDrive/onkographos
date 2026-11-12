# Dynamic Gaussian Splatting Scene Management

Also called: Spirula Scene Management, ROSplat Dynamic Buffer Management

## Overview
Dynamic Gaussian Splatting Scene Management handles the runtime lifecycle of 3D Gaussian splats within a rendering engine, specifically addressing the need for continuous, batched updates in dynamic scenes [S1, unofficial]. Unlike static loading methods that require full re-initialization, this method supports incremental `append` operations and dynamic buffer resizing, allowing the scene to grow or refresh without restarting the renderer [S1, unofficial]. It differs from sibling methods like [Splat Data Buffer Layout](5568-splat-data-buffer-layout.md) by managing the host-to-device transfer and capacity growth logic, rather than just the internal memory layout.

## How it works
The method operates on a Structure-of-Arrays (SoA) paradigm, maintaining device-resident buffers for splat attributes and metadata [S1, unofficial]. It manages the transition from host memory (where incoming batches arrive) to device memory (where the renderer reads data) through a series of validated steps.

### Inputs and Outputs
*   **Inputs:**
    *   `sh_degree`: Integer in $[0, 4]$ determining spherical harmonic complexity [S1, unofficial].
    *   `initial_capacity`: Integer $\ge 0$ for the initial buffer size [S1, unofficial].
    *   `batch_count`: Number of splats to append [S1, unofficial].
    *   `means_xyz`: $N \times 3$ float array of positions [S1, unofficial].
    *   `quaternions_wxyz`: $N \times 4$ float array of rotations (scalar-last) [S1, unofficial].
    *   `log_scales_xyz`: $N \times 3$ float array of log-scales [S1, unofficial].
    *   `opacity_logits`: $N$ float array of opacity logits [S1, unofficial].
    *   `features_dc_rgb`: $N \times 3$ float array of DC spherical harmonic coefficients [S1, unofficial].
    *   `features_sh_rgb`: $N \times D \times 3$ float array of higher-order SH coefficients, where $D = (sh\_degree + 1)^2 - 1$ [S1, unofficial].
    *   `camera_params`: Width, height, view matrix ($4 \times 4$), and intrinsics ($fx, fy, cx, cy$) [S1, unofficial].
    *   `scale_modifier`: Positive finite float for global scale adjustment [S1, unofficial].
*   **Outputs:**
    *   Rendered image buffers (RGBA8, Depth, or Opacity) [S1, unofficial].
    *   Updated scene state (count, capacity, device buffers) [S1, unofficial].

### Data Structures
The core state is held in `DeviceScene` and `HostRender` structures [S1, unofficial].

**DeviceScene** (Device Memory):
*   `means`: `float3*` pointer, size $capacity \times 3$ [S1, unofficial].
*   `quats`: `float4*` pointer, size $capacity \times 4$ [S1, unofficial].
*   `scales`: `float3*` pointer, size $capacity \times 3$ [S1, unofficial].
*   `opacities`: `float*` pointer, size $capacity$ [S1, unofficial].
*   `features_dc`: `float3*` pointer, size $capacity \times 3$ [S1, unofficial].
*   `features_sh`: `float3*` pointer, size $capacity \times num\_sh$ [S1, unofficial].
*   `count`: `int64_t`, current number of splats [S1, unofficial].
*   `capacity`: `int64_t`, current buffer capacity [S1, unofficial].
*   `sh_degree`: `int`, current SH degree [S1, unofficial].
*   `num_sh`: `int`, number of SH coefficients per splat [S1, unofficial].

**HostRender** (Pinned Host Memory):
*   `rgb`: `float3*` pointer, size $rgb\_pixels$ [S1, unofficial].
*   `scalar`: `float*` pointer, size $scalar\_pixels$ [S1, unofficial].
*   `rgb_pixels`: `size_t` [S1, unofficial].
*   `scalar_pixels`: `size_t` [S1, unofficial].

### Steps

1.  **Initialize Renderer**:
    *   Validate `sh_degree` in $[0, 4]$ and `initial_capacity` $\ge 0$ [S1, unofficial].
    *   Reset engine state and release existing scene data [S1, unofficial].
    *   Set `scene.sh_degree` to `sh_degree` [S1, unofficial].
    *   Calculate `scene.num_sh` as $(sh\_degree + 1)^2 - 1$ [S1, unofficial].
    *   Reserve memory for `initial_capacity` [S1, unofficial].
    *   Bind scene to engine [S1, unofficial].

2.  **Reserve Scene Capacity**:
    *   If `requested_capacity` > `current_capacity`:
        *   Calculate `next_capacity` as $\max(requested, \max(4096, current\_capacity \times 2))$ [S1, unofficial].
        *   Grow all device buffers (`means`, `quats`, `scales`, `opacities`, `features_dc`) to `next_count` [S1, unofficial].
        *   If `num_sh` > 0, grow `features_sh` to `next_count * num_sh` [S1, unofficial].
        *   Update `scene.capacity` and bind scene [S1, unofficial].

3.  **Bind Scene to Engine**:
    *   Set `state.cur_num_splats` to `scene.count` [S1, unofficial].
    *   Set `state.max_num_splats` to `scene.capacity` [S1, unofficial].
    *   Set `state.num_sh` to `scene.num_sh` and `state.sh_degree` to `scene.sh_degree` [S1, unofficial].
    *   Create `DeviceVector` views for `means`, `quats`, `scales`, `opacities`, `features_dc` with size `capacity` [S1, unofficial].
    *   If `num_sh` > 0, create `DeviceTensor2D` view for `features_sh` with dimensions `(capacity, num_sh, 3)`; else empty tensor [S1, unofficial].
    *   Set `state.world.initialized` to `true` if `capacity` > 0 [S1, unofficial].

4.  **Append Splats**:
    *   Validate `batch_count` $\ge 0$ and input arrays are non-null [S1, unofficial].
    *   Check for integer overflow: if `scene.count` > `numeric_limits<int64_t>::max() - batch_count`, throw `std::overflow_error` [S1, unofficial].
    *   Calculate `offset` = `scene.count` [S1, unofficial].
    *   Reserve space for `offset + batch_count` (triggering capacity growth if needed) [S1, unofficial].
    *   Copy `means`, `quats`, `scales`, `opacities`, `features_dc` from host to device at `offset` [S1, unofficial].
    *   If `num_sh` > 0, copy `features_sh` from host to device at `offset * num_sh` [S1, unofficial].
    *   Increment `scene.count` and bind scene [S1, unofficial].

5.  **Set Camera**:
    *   Validate `width`/`height` > 0 and `view_matrix`/`intrinsics` non-null [S1, unofficial].
    *   Initialize `dist_coeffs[8]` to zero [S1, unofficial].
    *   Call `set_camera_params` with `width`, `height`, "PINHOLE", "NONE", `view_matrix` ($1 \times 4 \times 4$ float), `intrinsics` ($1 \times 4$ float), `dist_coeffs` ($1 \times 8$ float) [S1, unofficial].
    *   Set `width`, `height`, and `camera_ready` = `true` [S1, unofficial].

6.  **Set Scale Modifier**:
    *   Validate `modifier` > 0 and finite [S1, unofficial].
    *   If `modifier` equals current `scale_modifier`, return [S1, unofficial].
    *   Calculate `log_delta` = $\log(requested\_modifier / scale\_modifier)$ [S1, unofficial].
    *   Calculate `numel` = `scene.count` $\times 3$ [S1, unofficial].
    *   Allocate device scratch buffer of size `max_chunk` ($1 \ll 20$) [S1, unofficial].
    *   Loop over `numel` in chunks:
        *   Copy `log_delta` to device chunk [S1, unofficial].
        *   Call `float_add_into` on `scales` chunk and device `delta` chunk [S1, unofficial].
    *   Free scratch buffer [S1, unofficial].
    *   Update `scale_modifier` [S1, unofficial].

7.  **Render Color**:
    *   Validate output buffer size $\ge$ `pixel_count` $\times 4$ [S1, unofficial].
    *   If `scene.count` == 0, fill output with black (RGB 0,0,0) and alpha 255 [S1, unofficial].
    *   Determine `render_sh_degree` as $\min(requested\_sh\_degree, scene.sh\_degree)$ if `requested` $\ge 0$, else `scene.sh_degree` [S1, unofficial].
    *   Call `forward_3dgs("3dgs", render_sh_degree, true)` [S1, unofficial].
    *   Get RGB buffer from `state.fwd.renders` [S1, unofficial].
    *   Resize `host.rgb` if needed [S1, unofficial].
    *   Copy RGB from device to host [S1, unofficial].
    *   Write RGB to output buffer using `write_gray_or_rgb` [S1, unofficial].

8.  **Render Depth**:
    *   Validate output [S1, unofficial].
    *   If `scene.count` == 0, fill black/alpha 255 [S1, unofficial].
    *   Call `forward_3dgs` [S1, unofficial].
    *   Get depth buffer from `state.fwd.renders` (index 1) [S1, unofficial].
    *   Resize `host.scalar` if needed [S1, unofficial].
    *   Copy depth from device to host [S1, unofficial].
    *   Call `write_depth(output, pixel_count)` [S1, unofficial].

9.  **Render Opacity**:
    *   Validate output [S1, unofficial].
    *   If `scene.count` == 0, fill black/alpha 255 [S1, unofficial].
    *   Call `forward_3dgs` [S1, unofficial].
    *   Get transmittance buffer from `state.fwd.render_Ts` [S1, unofficial].
    *   Resize `host.scalar` if needed [S1, unofficial].
    *   Copy transmittance from device to host [S1, unofficial].
    *   Loop over pixels to write opacity (implementation mirrors color/depth rendering) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class DynamicGaussianScene:
    def __init__(self, sh_degree: int, initial_capacity: int):
        self.sh_degree = sh_degree
        self.num_sh = (sh_degree + 1) ** 2 - 1
        self.count = 0
        self.capacity = 0
        self.scale_modifier = 1.0
        
        # Device buffers (simulated as numpy arrays for clarity)
        self.means = None
        self.quats = None
        self.scales = None
        self.opacities = None
        self.features_dc = None
        self.features_sh = None
        
        self._resize(initial_capacity)

    def _resize(self, new_capacity: int):
        """Grow buffers to new_capacity."""
        # Growth logic: max(requested, max(4096, current_capacity * 2))
        if new_capacity > self.capacity:
            next_capacity = max(new_capacity, max(4096, self.capacity * 2))
        else:
            next_capacity = self.capacity
            
        if next_capacity == self.capacity:
            return

        # Grow buffers
        self.means = np.resize(self.means, (next_capacity, 3))
        self.quats = np.resize(self.quats, (next_capacity, 4))
        self.scales = np.resize(self.scales, (next_capacity, 3))
        self.opacities = np.resize(self.opacities, (next_capacity,))
        self.features_dc = np.resize(self.features_dc, (next_capacity, 3))
        
        if self.num_sh > 0:
            self.features_sh = np.resize(self.features_sh, (next_capacity, self.num_sh, 3))
            
        self.capacity = next_capacity

    def append(self, count: int, means: np.ndarray, quats: np.ndarray, 
               log_scales: np.ndarray, opacities: np.ndarray, 
               features_dc: np.ndarray, features_sh: np.ndarray):
        """Append a batch of splats."""
        if count < 0:
            raise ValueError("Batch count must be >= 0")
        
        # Overflow check
        if self.count > np.iinfo(np.int64).max - count:
            raise OverflowError("Scene count overflow")
            
        offset = self.count
        self._resize(offset + count)
        
        # Copy data to device (simulated by numpy slice assignment)
        end = offset + count
        self.means[offset:end] = means
        self.quats[offset:end] = quats
        self.scales[offset:end] = log_scales
        self.opacities[offset:end] = opacities
        self.features_dc[offset:end] = features_dc
        
        if self.num_sh > 0:
            # features_sh is N x D x 3
            self.features_sh[offset:end] = features_sh
            
        self.count = end

    def set_scale_modifier(self, modifier: float):
        """Apply global scale modifier."""
        if modifier <= 0 or not math.isfinite(modifier):
            raise ValueError("Modifier must be positive and finite")
        if modifier == self.scale_modifier:
            return
            
        log_delta = math.log(modifier / self.scale_modifier)
        self.scale_modifier = modifier
        
        # Apply to existing scales in log space
        # Scales are stored as log(scales), so we add log_delta
        if self.count > 0:
            self.scales[:, :] += log_delta

    def render(self, width: int, height: int, view_matrix: np.ndarray, 
               intrinsics: np.ndarray, output_mode: int):
        """Render the scene."""
        pixel_count = width * height
        
        if self.count == 0:
            # Fill black/alpha 255
            return np.zeros((pixel_count, 4), dtype=np.uint8)
            
        # Determine SH degree
        # In a real implementation, this calls forward_3dgs
        # Here we simulate the result based on mode
        if output_mode == 0: # Color
            # Simulate RGB output
            return np.random.randint(0, 255, (pixel_count, 3), dtype=np.uint8)
        elif output_mode == 1: # Depth
            # Simulate Depth output
            return np.random.rand(pixel_count).astype(np.float32)
        elif output_mode == 2: # Opacity
            # Simulate Opacity output
            return np.random.rand(pixel_count).astype(np.float32)
        else:
            raise ValueError("Unknown output mode")
```

### Parameters
*   `sh_degree`: Integer, range $[0, 4]$. Determines spherical harmonic complexity. `num_sh` is calculated as $(sh\_degree + 1)^2 - 1$ [S1, unofficial].
*   `scale_modifier`: Float, must be positive and finite. Applied multiplicatively to splat scales via log-space addition to avoid precision issues and allow dynamic adjustment [S1, unofficial].

### Edge Cases and Pitfalls
*   **Empty Scene**: If `scene.count` == 0 during render, the renderer outputs a black image (RGB 0,0,0) with full alpha (255) instead of crashing or rendering nothing [S1, unofficial].
*   **Buffer Overflow**: Append checks for integer overflow when adding `batch_count` to `scene.count`. Throws `std::overflow_error` if `scene.count` > `numeric_limits<int64_t>::max() - batch_count` [S1, unofficial].
*   **Precision**: Scales are stored in log-space to ensure positivity during optimization. Linear scales are recovered via $s_{linear} = \exp(s_{log})$ for covariance calculation [S1, unofficial].

## Subtopics

### [Gaussian Splatting Scene Management & ROS2 Bridge](7494-gaussian-splatting-scene-management-ros2-bridge.md)
Manages the dynamic lifecycle of 3D Gaussian splats within a ROS2-enabled rendering pipeline, bridging sensor or simulation data to the GPU renderer. It solves the problem of real-time scene updates by queuing incoming Gaussian batches and efficiently draining them to the renderer state. It differs from static loading by enabling continuous, batched updates via ROS2 communication, allowing the scene to grow or refresh without full re-initialization.

## Operating characteristics
*   **Time Complexity**: Append operations are $O(N)$ for data copy, where $N$ is the batch size. Capacity growth involves reallocation and copy, which is amortized $O(1)$ per splat if doubling strategy is used. Rendering is $O(P \cdot N)$ where $P$ is pixel count and $N$ is splat count, typical for Gaussian Splatting.
*   **Memory Complexity**: $O(N \cdot C)$ where $N$ is capacity and $C$ is the number of attributes per splat (means, quats, scales, opacities, features). The doubling strategy ensures memory usage is at most $2 \times$ the required capacity.
*   **Performance**: The method uses pinned host memory for readback and device buffers for storage, optimizing data transfer [S1, unofficial]. The use of log-space for scales and opacities avoids precision issues during optimization [S1, unofficial].
*   **Strengths**: Supports dynamic scenes with incremental updates. Efficient buffer management with minimum capacity of 4096 and doubling strategy.
*   **Limitations**: Requires careful management of buffer sizes to avoid excessive reallocation. Integer overflow checks are necessary for large scenes.

## Background
The method originates from the ROSplat project, which integrates 3D Gaussian Splatting with ROS2 for dynamic scene management [S1, unofficial]. It builds on the principles of Structure-of-Arrays (SoA) data layout for cache coherency [S1, unofficial]. The specific implementation details, such as the buffer growth logic and log-space storage, are derived from the Spirula renderer bridge [S1, unofficial].

## Sources
[S1] Gaussian Splatting Meets ROS2, https://github.com/shadygm/ROSplat, fetched 2026-10-26 (unofficial)
