# Cascaded Voxel Cone Tracing Debug Visualization

Also called: Voxel Cone Tracing Debug Views

## Overview
This method renders ray-traced voxel data from a cascaded voxel cone tracing system into a 2D image buffer to visualize specific voxel attributes [S1, unofficial]. It supports four distinct debug modes (opacity, color, energy, and normals) to inspect the internal state of the voxelization and lighting pipeline [S1, unofficial]. Unlike standard rendering which produces a final lit image, this technique isolates and composites raw voxel data (such as energy vectors or packed normals) for diagnostic purposes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera view and projection matrices (inverted), voxel cascade data (opacity, color, energy, normals), debug mode selector, and output resolution.
*   **Outputs:** A 2D RGBA16F image buffer containing the debug visualization.

**Data Structures**
The implementation relies on specific Uniform Buffer Objects (UBOs) and Storage Buffer Objects (SSBOs)/Textures [S1, unofficial]:

1.  **`CommonUBO`** (std140, set 2, binding 0):
    *   `uvec4 cascade_size`: xyz component stores size.
    *   `uvec4 cascade_mask`.
    *   `uint n_cascades`.
    *   `float min_voxel_size`.
    *   `vec4 anchors[MAX_CASCADES]`.
2.  **`DebugUBO`** (std140, set 2, binding 1):
    *   `mat4 camera_view_matrix_inv`.
    *   `mat4 camera_projection_matrix_inv`.
    *   `uint debug_view`: Selects the visualization mode (0-3).
3.  **`DiffuseCascades`** (std430, set 0, binding 3): Flat array of `uint` values representing packed voxel data.
4.  **`OpacityCascades`** (std430, set 0, binding 2): Flat array of `float` values.
5.  **`u_energy_cascades`** (set 0, binding 0): 3D sampler (`sampler3D`).
6.  **`u_color_cascades`** (set 0, binding 1): 3D sampler (`sampler3D`).
7.  **`u_color_target`** (set 1, binding 0): 2D storage image with format `rgba16f`.

**Algorithm Steps**
The visualization is performed per-pixel via a compute or fragment shader loop [S1, unofficial].

1.  **Ray Initialization:**
    *   Compute `ray_origin` using the inverse view matrix applied to the camera position (typically origin in view space, transformed to world space):
        $$ray_{origin} = V_{inv} \times [0, 0, 0, 1]^T$$
    *   Unproject the current pixel's UV coordinates through the inverse projection matrix to get near (`src`) and far (`dst`) plane points in world space.
    *   Transform `src` and `dst` by the inverse view matrix.
    *   Compute direction: $ray_{dir} = normalize(dst - src)$.

2.  **Accumulation Setup:**
    *   Initialize accumulator: $acc = vec4(0.0)$.
    *   Initialize position: $pos = ray_{origin}$.
    *   Initialize step size: $step = min\_voxel\_size \times 0.25$.

3.  **Ray Marching Loop:**
    *   Iterate up to 1024 times [S1, unofficial].
    *   Increment distance $d$ by $step$.
    *   Advance position: $pos += step \times ray_{dir}$.
    *   Increase step size by 1%: $step_{i+1} = step_i \times 1.01$.
    *   **Boundary Check:** If the global invocation ID exceeds the output resolution (640x360), return early [S1, unofficial].

4.  **Attribute Fetching and Accumulation:**
    Depending on `debug_view`, fetch voxel data and update $acc$ using the standard alpha compositing rule:
    $$C_{acc} = C_{acc} + C_{voxel} \times \rho \times (1 - \rho_{acc})$$
    $$\rho_{acc} = \rho_{acc} + \rho \times (1 - \rho_{acc})$$
    Where $C_{voxel}$ is the voxel color/value, $\rho$ is opacity, and $\rho_{acc}$ is accumulated alpha.

    *   **Debug View 0 (Opacity):**
        *   Fetch opacity: $\rho = voxelAt(pos)$.
        *   Fetch cascade index: $c = cascadeAt(pos)$.
        *   Fetch cascade color: $C_{voxel} = cascade\_colors[c \pmod 8]$.
        *   Apply exponential decay: $C_{voxel} *= max(exp(-d \times 0.1), 0.01)$.
        *   Accumulate: $acc.rgb += C_{voxel} \times \rho \times (1.0 - acc.a)$; $acc.a += \rho \times (1.0 - acc.a)$.

    *   **Debug View 1 (Color):**
        *   Fetch voxel color: $C_{voxel} = voxelAt2(pos)$.
        *   Accumulate: $acc.rgb += 0.5 \times C_{voxel}.rgb \times C_{voxel}.a \times (1.0 - acc.a)$; $acc.a += C_{voxel}.a \times (1.0 - acc.a)$.

    *   **Debug View 2 (Energy):**
        *   Compute basis vector: $basis = vec4(-ray_{dir} \times \sqrt{\frac{3}{4\pi}}, \frac{1}{\sqrt{4\pi}})$.
        *   Fetch voxel energy: $E = voxelEnergyAt(pos)$.
        *   Compute energy value: $energy = dot(basis, E)$.
        *   Fetch voxel color: $C_{base} = voxelColorAt(pos)$.
        *   Combine: $C_{voxel} = vec4(energy \times C_{base}.rgb, C_{base}.a)$.
        *   Accumulate: $acc.rgb += 0.5 \times C_{voxel}.rgb \times C_{voxel}.a \times (1.0 - acc.a)$; $acc.a += C_{voxel}.a \times (1.0 - acc.a)$.

    *   **Debug View 3 (Normals):**
        *   Fetch packed data: $packed = packedAt(pos)$.
        *   Unpack normal: $N = unpackNormal(packed)$.
        *   Unpack opacity: $\rho = unpackOpacity(packed)$.
        *   Accumulate: $acc.rgb += N \times \rho \times (1.0 - acc.a)$; $acc.a += \rho \times (1.0 - acc.a)$.

    *   **Early Exit:** If $acc.a > 0.99$, break the loop [S1, unofficial].

5.  **Final Compositing:**
    *   Compute background color from ray direction: $C_{bg} = 0.5 \times ray_{dir} + 0.5$.
    *   Blend accumulated color with background:
        $$C_{out} = (1 - \rho_{acc}) C_{bg} + \rho_{acc} C_{acc}$$
    *   Store result to `u_color_target` at `global_id` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_ITERATIONS = 1024
OPACITY_THRESHOLD = 0.99
STEP_GROWTH = 1.01
INITIAL_STEP_MULT = 0.25
DECAY_FACTOR = 0.1
MIN_COLOR_VALUE = 0.01
COLOR_MULTIPLIER = 0.5
OUTPUT_WIDTH = 640
OUTPUT_HEIGHT = 360

# Helper functions (implementation details depend on specific voxel grid layout)
def voxelAt(pos): ...
def cascadeAt(pos): ...
def voxelAt2(pos): ...
def voxelEnergyAt(pos): ...
def voxelColorAt(pos): ...
def packedAt(pos): ...
def unpackNormal(packed): ...
def unpackOpacity(packed): ...

def compute_debug_view(global_id, u_color_target, camera_view_inv, camera_proj_inv, 
                       min_voxel_size, n_cascades, debug_view):
    
    # 1. Ray Initialization
    # Assuming global_id maps to uv coordinates
    uv = vec2(global_id.x / OUTPUT_WIDTH, global_id.y / OUTPUT_HEIGHT)
    
    # Unproject UV to get src and dst in world space
    # Note: Exact unprojection depends on projection matrix structure (perspective/orthographic)
    src = unproject(uv, camera_proj_inv, camera_view_inv, near_plane)
    dst = unproject(uv, camera_proj_inv, camera_view_inv, far_plane)
    
    ray_dir = normalize(dst - src)
    ray_origin = camera_view_inv[3].xyz # Assuming camera is at origin in view space, transformed to world
    
    # 2. Accumulation Setup
    acc = vec4(0.0)
    pos = ray_origin
    step = min_voxel_size * INITIAL_STEP_MULT
    
    # 3. Ray Marching Loop
    for i in range(MAX_ITERATIONS):
        d = (i + 1) * step # Approximate distance, though step grows
        
        # Advance position
        pos = pos + step * ray_dir
        
        # Increase step size
        step = step * STEP_GROWTH
        
        # Boundary check
        if global_id.x >= OUTPUT_WIDTH or global_id.y >= OUTPUT_HEIGHT:
            return

        # Fetch and Accumulate based on debug_view
        if debug_view == 0: # Opacity
            occlusion = voxelAt(pos)
            cascade_idx = cascadeAt(pos)
            if cascade_idx >= n_cascades:
                occlusion = 0.0
            
            # Cascade colors palette (example definition)
            cascade_colors = [
                vec4(1.0, 0.0, 0.0, 1.0), vec4(0.0, 1.0, 0.0, 1.0),
                vec4(0.0, 0.0, 1.0, 1.0), vec4(1.0, 1.0, 0.0, 1.0),
                vec4(1.0, 0.0, 1.0, 1.0), vec4(0.0, 1.0, 1.0, 1.0),
                vec4(1.0, 0.5, 0.0, 1.0), vec4(0.5, 0.5, 0.5, 1.0)
            ]
            voxel_color = cascade_colors[cascade_idx % 8]
            
            # Exponential decay
            decay = max(exp(-d * DECAY_FACTOR), MIN_COLOR_VALUE)
            voxel_color.rgb *= decay
            
            acc.rgb += voxel_color.rgb * occlusion * (1.0 - acc.a)
            acc.a += occlusion * (1.0 - acc.a)
            
        elif debug_view == 1: # Color
            voxel_color = voxelAt2(pos)
            if cascadeAt(pos) >= n_cascades:
                voxel_color = vec4(0.0)
                
            acc.rgb += COLOR_MULTIPLIER * voxel_color.rgb * voxel_color.a * (1.0 - acc.a)
            acc.a += voxel_color.a * (1.0 - acc.a)
            
        elif debug_view == 2: # Energy
            basis = vec4(-ray_dir * sqrt(3.0 / (4.0 * PI)), 1.0 / sqrt(4.0 * PI))
            voxel_energy = voxelEnergyAt(pos)
            if cascadeAt(pos) >= n_cascades:
                voxel_energy = vec4(0.0)
                
            energy = dot(basis, voxel_energy)
            voxel_color_base = voxelColorAt(pos)
            if cascadeAt(pos) >= n_cascades:
                voxel_color_base = vec4(0.0)
                
            voxel_color = vec4(energy * voxel_color_base.rgb, voxel_color_base.a)
            
            acc.rgb += COLOR_MULTIPLIER * voxel_color.rgb * voxel_color.a * (1.0 - acc.a)
            acc.a += voxel_color.a * (1.0 - acc.a)
            
        elif debug_view == 3: # Normals
            packed = packedAt(pos)
            if cascadeAt(pos) >= n_cascades:
                packed = 0
                
            normal = unpackNormal(packed)
            opacity = unpackOpacity(packed)
            
            acc.rgb += normal * opacity * (1.0 - acc.a)
            acc.a += opacity * (1.0 - acc.a)
            
        # Early exit
        if acc.a > OPACITY_THRESHOLD:
            break
            
    # 4. Final Compositing
    bg_color = 0.5 * ray_dir + 0.5
    out_color = (1.0 - acc.a) * bg_color + acc.a * acc
    
    # Store result
    imageStore(u_color_target, ivec2(global_id.x, global_id.y), out_color)
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of ray steps, capped at 1024 iterations per pixel. The step size growth (1%) means the ray covers increasing distances, potentially reducing the number of steps required to traverse large empty spaces, but increasing the risk of missing thin features if the step becomes too large relative to voxel size.
*   **Memory Complexity:** Low. The method primarily uses the existing voxel cascade storage buffers and textures. No additional large temporary buffers are required beyond the output image.
*   **Performance:** The implementation targets a resolution of 640x360 pixels [S1, unofficial]. The early exit condition ($acc.a > 0.99$) significantly reduces computation for opaque regions.
*   **Strengths:** Provides immediate visual feedback on voxel data integrity, cascade boundaries, and energy distribution. The use of exponential decay in Debug View 0 helps visualize depth attenuation.
*   **Limitations:** The fixed maximum iteration count (1024) may lead to incomplete tracing for very deep scenes if the step size grows too quickly. The visualization is specific to the implementation's voxel layout and helper functions (`voxelAt`, etc.), which are not standardized.

## Background
This technique is part of an implementation of cascaded voxel cone tracing designed to handle dynamic geometry and large scenes [S1, unofficial]. It was developed as part of a project using Zig and SDL3, visualizing data from the Sponza scene [S1, unofficial]. The debug views allow developers to inspect the internal state of the voxelization pipeline, such as opacity accumulation, color storage, energy vectors, and normal packing.

## Sources
[S1] Sandalmoth/voxel-cone-tracing, https://github.com/Sandalmoth/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
