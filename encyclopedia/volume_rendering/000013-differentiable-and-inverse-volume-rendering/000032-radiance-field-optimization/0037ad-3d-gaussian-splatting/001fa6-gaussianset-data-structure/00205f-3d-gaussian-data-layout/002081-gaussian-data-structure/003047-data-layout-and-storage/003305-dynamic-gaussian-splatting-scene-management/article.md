# Dynamic Gaussian Splatting Scene Management

Also called: Spirula Scene Manager, ROSplat Scene Management

## Overview
Dynamic Gaussian Splatting Scene Management handles the lifecycle, memory allocation, and dynamic modification of a set of 3D Gaussians on the GPU during runtime. It solves the problem of efficiently managing a variable-sized collection of splats (adding, resizing, and modifying) without requiring full scene reconstruction or static buffer allocation. It distinguishes itself from static layout methods by implementing a Structure-of-Arrays (SoA) buffer growth strategy with exponential doubling and supporting real-time parameter adjustments like scale modifiers and camera updates [S1, unofficial].

## How it works
The system manages a `DeviceScene` structure resident on the GPU, which contains contiguous buffers for Gaussian attributes and metadata. The management logic is driven by a set of C API functions that wrap internal C++ renderer state.

### Inputs and Outputs
*   **Inputs:** Host-side arrays for Gaussian attributes (means, quaternions, log-scales, opacity logits, SH features), camera parameters (view matrix, intrinsics), and management commands (reset, append, scale modifier).
*   **Outputs:** Updated GPU scene state, rendered images (RGBA8, Depth, Opacity), and error codes.

### Data Structures
The core state is held in `DeviceScene` (on device) and `HostRender` (on pinned host memory for readback) [S1, unofficial].

**DeviceScene Layout:**
*   `means`: `float3*` (Size: $N \times 3$)
*   `quats`: `float4*` (Size: $N \times 4$)
*   `scales`: `float3*` (Size: $N \times 3$)
*   `opacities`: `float*` (Size: $N$)
*   `features_dc`: `float3*` (Size: $N \times 3$)
*   `features_sh`: `float3*` (Size: $N \times \text{num\_sh}$)
*   `count`: `int64_t` (Current number of splats)
*   `capacity`: `int64_t` (Allocated buffer size)
*   `sh_degree`: `int` (Spherical Harmonics degree)
*   `num_sh`: `int` (Number of SH coefficients per channel, calculated as $(\text{sh\_degree} + 1)^2 - 1$)

**HostRender Layout:**
*   `rgb`: `float3*` (Size: $W \times H$)
*   `scalar`: `float*` (Size: $W \times H$)
*   `rgb_pixels`: `size_t`
*   `scalar_pixels`: `size_t`

### Parameters
*   **SH Degree:** Integer in $[0, 4]$. Determines the complexity of the spherical harmonic representation.
*   **Scale Modifier:** Positive, finite float. A global multiplier applied to all Gaussian scales.
*   **Initial Capacity:** Non-negative integer. The starting buffer size for the scene.

### Numbered Steps

1.  **Initialize/Reset Scene:**
    *   Validate `sh_degree` $\in [0, 4]$ and `initial_capacity` $\ge 0$.
    *   Reset the engine state and release existing scene data.
    *   Set `scene.sh_degree` and calculate `scene.num_sh` as $(\text{sh\_degree} + 1)^2 - 1$.
    *   Reserve memory for `initial_capacity`.
    *   Bind the scene to the engine.

2.  **Reserve/Grow Capacity:**
    *   If the requested capacity exceeds the current `scene.capacity`:
        *   Calculate `next_capacity` as $\max(\text{requested}, \max(4096, \text{current\_capacity} \times 2))$.
        *   Grow all device buffers (`means`, `quats`, `scales`, `opacities`, `features_dc`) to `next_capacity`.
        *   If `num_sh > 0`, grow `features_sh` to `next_capacity * num_sh`.
        *   Update `scene.capacity`.
        *   Bind the scene to update internal state pointers.

3.  **Bind Scene to Engine State:**
    *   Set `state.cur_num_splats` to `scene.count`.
    *   Set `state.max_num_splats` to `scene.capacity`.
    *   Set `state.num_sh` to `scene.num_sh` and `state.sh_degree` to `scene.sh_degree`.
    *   Create `DeviceVector` views for `means`, `quats`, `scales`, `opacities`, and `features_dc` with size `capacity`.
    *   If `num_sh > 0`, create a `DeviceTensor2D` view for `features_sh` with dimensions `(capacity, num_sh, 3)`; otherwise, create an empty tensor.
    *   Set `state.world.initialized` to `true` if `capacity > 0`.

4.  **Append Splats:**
    *   Validate `batch_count` $\ge 0$ and that input arrays are non-null.
    *   Check for integer overflow: if `scene.count > numeric_limits<int64_t>::max() - batch_count`, throw `std::overflow_error`.
    *   Calculate `offset = scene.count`.
    *   Reserve space for `offset + batch_count` (triggering growth if necessary via Step 2).
    *   Copy `means`, `quats`, `scales`, `opacities`, and `features_dc` from host to device starting at `offset`.
    *   If `num_sh > 0`, copy `features_sh` from host to device starting at `offset * num_sh`.
    *   Increment `scene.count` by `batch_count`.
    *   Bind the scene.

5.  **Set Camera:**
    *   Validate `width > 0`, `height > 0`, and that `view_matrix` and `intrinsics` are non-null.
    *   Initialize `dist_coeffs[8]` to zero.
    *   Call `set_camera_params` with:
        *   `width`, `height`
        *   Projection type: `"PINHOLE"`
        *   Distortion type: `"NONE"`
        *   `view_matrix`: $4 \times 4$ float array (row-major world-to-camera)
        *   `intrinsics`: $1 \times 4$ float array $(f_x, f_y, c_x, c_y)$
        *   `dist_coeffs`: $1 \times 8$ float array
    *   Set `width`, `height`, and `camera_ready = true`.

6.  **Set Scale Modifier:**
    *   Validate `modifier > 0` and is finite.
    *   If `modifier` equals the current `scale_modifier`, return.
    *   Calculate `log_delta = log(requested_modifier / scale_modifier)`.
    *   Calculate `numel = scene.count * 3`.
    *   Allocate a device scratch buffer of size `max_chunk` ($2^{20}$).
    *   Loop over `numel` in chunks of `max_chunk`:
        *   Copy `log_delta` to the device chunk.
        *   Call `float_add_into` on the `scales` chunk and the device `delta` chunk.
    *   Free the scratch buffer.
    *   Update `scale_modifier`.

7.  **Render Color:**
    *   Validate output buffer size $\ge \text{pixel\_count} \times 4$.
    *   If `scene.count == 0`, fill output with black (RGB 0,0,0) and alpha 255.
    *   Determine `render_sh_degree` as $\min(\text{requested\_sh\_degree}, \text{scene.sh\_degree})$ if `requested >= 0`, else `scene.sh_degree`.
    *   Call `forward_3dgs("3dgs", render_sh_degree, true)`.
    *   Get the RGB buffer from `state.fwd.renders`.
    *   Resize `host.rgb` if needed.
    *   Copy RGB from device to host.
    *   Write RGB to the output buffer using `write_gray_or_rgb`.

8.  **Render Depth:**
    *   Validate output.
    *   If `scene.count == 0`, fill with black/alpha 255.
    *   Call `forward_3dgs`.
    *   Get the depth buffer from `state.fwd.renders` (index 1).
    *   Resize `host.scalar`.
    *   Copy depth from device to host.
    *   Call `write_depth(output, pixel_count)`.

9.  **Render Opacity:**
    *   Validate output.
    *   If `scene.count == 0`, fill with black/alpha 255.
    *   Call `forward_3dgs`.
    *   Get the transmittance buffer from `state.fwd.render_Ts`.
    *   Resize `host.scalar`.
    *   Copy transmittance from device to host.
    *   Loop over pixels to write opacity (implementation mirrors color/depth rendering logic).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming standard vector/math operations are available
# No external libraries like Eigen or OpenVDB are used.

def reset_scene(sh_degree: int, initial_capacity: int) -> int:
    """
    Initialize renderer with SH degree and initial capacity.
    Returns 0 on success, error code otherwise.
    """
    if not (0 <= sh_degree <= 4):
        return ERROR_INVALID_SH_DEGREE
    if initial_capacity < 0:
        return ERROR_INVALID_CAPACITY

    # Reset engine and release existing data
    engine.reset()
    
    # Set scene properties
    scene.sh_degree = sh_degree
    scene.num_sh = (sh_degree + 1) ** 2 - 1
    
    # Reserve memory
    reserve_memory(initial_capacity)
    
    # Bind scene to engine
    bind_scene_to_engine()
    return 0

def reserve_capacity(requested: int) -> int:
    """
    Grow buffers if requested capacity exceeds current capacity.
    """
    if requested <= scene.capacity:
        return 0
        
    # Calculate next capacity with minimum 4096 and doubling strategy
    next_capacity = max(requested, max(4096, scene.capacity * 2))
    
    # Grow all device buffers
    means.resize(next_capacity * 3)
    quats.resize(next_capacity * 4)
    scales.resize(next_capacity * 3)
    opacities.resize(next_capacity)
    features_dc.resize(next_capacity * 3)
    
    if scene.num_sh > 0:
        features_sh.resize(next_capacity * scene.num_sh)
        
    scene.capacity = next_capacity
    bind_scene_to_engine()
    return 0

def append_splats(count: int, means: list, quats: list, log_scales: list, 
                  opacity_logits: list, features_dc: list, features_sh: list) -> int:
    """
    Append a batch of splats to the scene.
    """
    if count < 0:
        return ERROR_INVALID_COUNT
    if scene.count > (2**63 - 1) - count:
        raise OverflowError("Scene count overflow")
        
    offset = scene.count
    
    # Reserve space
    reserve_capacity(offset + count)
    
    # Copy data to device at offset
    # Note: In a real implementation, this involves memory copies from host to device
    device_means[offset * 3 : (offset + count) * 3] = means
    device_quats[offset * 4 : (offset + count) * 4] = quats
    device_scales[offset * 3 : (offset + count) * 3] = log_scales
    device_opacities[offset : offset + count] = opacity_logits
    device_features_dc[offset * 3 : (offset + count) * 3] = features_dc
    
    if scene.num_sh > 0:
        # features_sh is stored as [N * num_sh]
        sh_offset = offset * scene.num_sh
        sh_end = (offset + count) * scene.num_sh
        device_features_sh[sh_offset : sh_end] = features_sh
        
    scene.count += count
    bind_scene_to_engine()
    return 0

def set_camera(width: int, height: int, view_matrix: list, intrinsics: list) -> int:
    """
    Set camera parameters.
    """
    if width <= 0 or height <= 0:
        return ERROR_INVALID_DIMENSIONS
    if not view_matrix or not intrinsics:
        return ERROR_NULL_POINTER
        
    dist_coeffs = [0.0] * 8
    
    # Call internal camera setup
    set_camera_params(
        width=width,
        height=height,
        projection="PINHOLE",
        distortion="NONE",
        view_matrix=view_matrix, # 4x4 float
        intrinsics=intrinsics,   # 1x4 float (fx, fy, cx, cy)
        dist_coeffs=dist_coeffs  # 1x8 float
    )
    
    scene.width = width
    scene.height = height
    scene.camera_ready = True
    return 0

def set_scale_modifier(modifier: float) -> int:
    """
    Apply a global scale modifier.
    """
    if modifier <= 0 or not is_finite(modifier):
        return ERROR_INVALID_MODIFIER
        
    if modifier == scene.scale_modifier:
        return 0
        
    log_delta = log(modifier / scene.scale_modifier)
    numel = scene.count * 3
    
    # Allocate scratch buffer
    max_chunk = 1 << 20
    scratch_buffer = allocate_device_buffer(max_chunk)
    
    # Loop over numel in chunks
    for i in range(0, numel, max_chunk):
        chunk_size = min(max_chunk, numel - i)
        
        # Copy log_delta to device chunk
        # In practice, this might be a broadcast or kernel launch
        copy_to_device(scratch_buffer, [log_delta] * chunk_size)
        
        # Add delta to scales
        float_add_into(
            target=device_scales[i : i + chunk_size],
            source=scratch_buffer,
            count=chunk_size
        )
        
    free_device_buffer(scratch_buffer)
    scene.scale_modifier = modifier
    return 0

def render_rgba8(output_buffer: list, pixel_count: int) -> int:
    """
    Render color and alpha.
    """
    if len(output_buffer) < pixel_count * 4:
        return ERROR_BUFFER_TOO_SMALL
        
    if scene.count == 0:
        # Fill with black (0,0,0) and alpha 255
        for i in range(pixel_count):
            output_buffer[i*4] = 0
            output_buffer[i*4+1] = 0
            output_buffer[i*4+2] = 0
            output_buffer[i*4+3] = 255
        return 0
        
    # Determine SH degree
    render_sh_degree = scene.sh_degree
    # If requested_sh_degree is provided and >= 0, use min(requested, scene.sh_degree)
    # For this example, we assume default behavior uses scene.sh_degree
    
    # Forward pass
    forward_3dgs(mode="3dgs", sh_degree=render_sh_degree, is_color=True)
    
    # Get RGB buffer from state
    rgb_buffer = state.fwd.renders
    
    # Resize host buffer if needed
    if len(host.rgb) < pixel_count * 3:
        host.rgb.resize(pixel_count * 3)
        
    # Copy from device to host
    copy_device_to_host(rgb_buffer, host.rgb)
    
    # Write to output buffer
    write_gray_or_rgb(host.rgb, output_buffer)
    
    return 0
```

### Edge Cases and Pitfalls
*   **Empty Scene:** If `scene.count == 0` during rendering, the renderer outputs a black image (RGB 0,0,0) with full alpha (255) instead of crashing or rendering nothing [S1, unofficial].
*   **Buffer Overflow:** Appending splats checks for integer overflow when adding `batch_count` to `scene.count`. If `scene.count > numeric_limits<int64_t>::max() - batch_count`, a `std::overflow_error` is thrown [S1, unofficial].
*   **Scale Precision:** The scale modifier is applied in log-space (`log_delta`) to avoid precision issues associated with direct multiplication of small scale values [S1, unofficial].

## Subtopics

### [Gaussian Splatting Scene Management & ROS2 Bridge](7494-gaussian-splatting-scene-management-ros2-bridge.md)
Integrates the scene management with ROS2 for robotic applications, handling data synchronization and bridge communication.

## Operating characteristics
*   **Time Complexity:** Appending splats is $O(N)$ where $N$ is the batch size, plus $O(C)$ for buffer growth if capacity is exceeded. Rendering is $O(P)$ where $P$ is the number of pixels, dependent on the number of splats $N$ and their overlap.
*   **Memory Complexity:** $O(N \times D)$ where $D$ is the number of attributes per splat (position, rotation, scale, opacity, SH features). The buffer growth strategy ensures memory usage is at least the next power-of-two multiple of the current count, capped by a minimum of 4096 splats.
*   **Performance:** The system uses Vulkan for GPU acceleration. Performance scales with the number of splats and the SH degree. Benchmarks show FPS varying with splat count and backend (CPU, Cupy, Torch) [S1, unofficial].
*   **Strengths:** Dynamic resizing allows for real-time scene updates. Log-space scale modification preserves precision.
*   **Limitations:** Buffer growth can cause temporary stalls. High SH degrees increase memory and compute cost.

## Background
The method is implemented in the ROSplat project, which integrates 3D Gaussian Splatting with ROS2 for robotic applications. It uses the Spirula backend for efficient GPU rendering. The system was developed to handle dynamic scenes in real-time robotic environments [S1, unofficial].

## Sources
[S1] Gaussian Splatting Meets ROS2, https://github.com/shadygm/ROSplat, fetched 2026-10-26 (unofficial)
