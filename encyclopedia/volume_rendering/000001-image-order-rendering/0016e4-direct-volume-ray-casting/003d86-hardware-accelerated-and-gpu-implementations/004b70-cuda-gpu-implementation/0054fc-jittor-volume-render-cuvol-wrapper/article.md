# Jittor Volume Render Cuvol Wrapper


## Overview
The Jittor Volume Render Cuvol Wrapper is a custom Jittor function that fuses the forward and backward passes of volume rendering into a single CUDA kernel launch sequence, managing memory aliases and gradient propagation for a sparse grid representation [S1, unofficial]. It differs from standard CPU-based ray marchers by executing the core ray casting logic on the GPU via CUDA, while maintaining compatibility with Jittor's automatic differentiation engine [S1, unofficial]. It builds on the [CUDA GPU Implementation](13016-cuda-gpu-implementation.md) by providing a high-level API that handles the integration of sparse grid data structures (offsets, scaling, links) into the CUDA kernels.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse grid object (containing density, SH coefficients, basis data, links, offsets, scaling), ray origins and directions (tensors), and a `RenderOptions` struct.
*   **Outputs:** Rendered color buffer, confidence/confidence map, and optionally a log-transparency buffer if background rendering is enabled.

**Data Structures:**
*   **RenderOptions:** A struct passed to CUDA kernels containing:
    *   `background_brightness`: float
    *   `step_size`: float
    *   `sigma_thresh`: float (density threshold for early ray termination)
    *   `stop_thresh`: float (transparency threshold for early ray termination)
    *   `near_clip`: float
    *   `use_spheric_clip`: bool
    *   `last_sample_opaque`: bool
*   **Sparse Grid:** Contains `_links` (connectivity), `_offset`, `_scaling`, `basis_dim`, `basis_type`, `basis_data`, `density_data`, `sh_data`, `background_links`, `background_data`.

**Steps:**
1.  **Prepare Grid Offsets and Scaling:**
    *   If `grid_coords` is true, use identity scaling.
    *   Otherwise, scale grid offsets and scaling by the number of links:
        $$ gsz = \text{shape}(\text{grid}.\_links)[0] $$
        $$ \text{self}.\_grid\_offset = \text{grid}.\_offset \times gsz - 0.5 $$
        $$ \text{self}.\_grid\_scaling = \text{grid}.\_scaling \times gsz $$
    *   [G] Ensure `gsz` is treated as a Jittor array for broadcasting.

2.  **Initialize Log Transmit Buffer (if background is used):**
    *   Create an empty buffer `log_transmit` with shape `[rays.origins.shape[0]]` and dtype matching `rays.origins`.
    *   Call `.stop_grad()` on `log_transmit` to prevent gradient flow through this buffer during backpropagation.

3.  **Launch Forward CUDA Kernel (`render_ray_kernel`):**
    *   Pass grid data (density, SH, links, offset, scaling, background links/data, basis info), ray origins/directions, `RenderOptions`, output buffer, and `log_transmit` pointer (or `nullptr` if no background).
    *   The kernel performs ray marching, sampling the sparse grid, and compositing colors.

4.  **Launch Background Kernel (if background is used):**
    *   Launch `render_background_kernel` subsequently to handle background color integration using the `log_transmit` buffer.

5.  **Gradient Computation (`grad`):**
    *   Initialize gradient buffers for density, SH, basis, and background data.
    *   Determine if gradients are needed for basis and background data based on `requires_grad` flags.

6.  **Launch Backward CUDA Kernel (`render_ray_backward_kernel`):**
    *   Pass gradients, color cache, rays, `RenderOptions`, and output gradient structure.
    *   Computes gradients for density, SH, and basis data based on the forward pass results.

7.  **Launch Background Backward Kernel (if background is used):**
    *   Launch `render_background_backward_kernel` to compute gradients related to background parameters.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jittor as jt
import numpy as np

class VolumeRenderCuvol(jt.Function):
    def execute(self, grid, rays, opt, use_background=False):
        # 1. Prepare grid offsets and scaling
        if grid.grid_coords:
            self._grid_offset = 0.0
            self._grid_scaling = 1.0
        else:
            gsz = jt.array(np.array(grid._links.shape[0]))
            self._grid_offset = grid._offset * gsz - 0.5
            self._grid_scaling = grid._scaling * gsz
            
        # 2. Initialize log_transmit buffer if background is used
        if use_background:
            self.log_transmit = jt.empty([rays.origins.shape[0]], dtype=rays.origins.dtype).stop_grad()
        else:
            self.log_transmit = None
            
        # 3. Launch forward CUDA kernel
        # Note: PackedVar32 is a Jittor-specific packing structure for CUDA
        output = jt.empty([rays.origins.shape[0], 3], dtype=rays.origins.dtype)
        
        # Kernel launch parameters
        blocks = (rays.origins.shape[0] + TRACE_RAY_CUDA_THREADS - 1) // TRACE_RAY_CUDA_THREADS
        
        render_ray_kernel[blocks, TRACE_RAY_CUDA_THREADS](
            grid_density_data=grid.density_data,
            grid_sh_data=grid.sh_data,
            grid_links=grid._links,
            grid_offset=self._grid_offset,
            grid_scaling=self._grid_scaling,
            grid_background_links=grid.background_links,
            grid_background_data=grid.background_data,
            basis_dim=grid.basis_dim,
            basis_type=grid.basis_type,
            grid_basis_data=grid.basis_data,
            rays_origins=rays.origins,
            rays_dirs=rays.dirs,
            opt=opt,
            output=output,
            log_transmit=self.log_transmit.ptr<float>() if use_background else None
        )
        
        # 4. Launch background kernel if needed
        if use_background:
            render_background_kernel[blocks, TRACE_RAY_CUDA_THREADS](
                # ... parameters for background kernel ...
            )
            
        return output

    def grad(self, grad_output):
        # 5. Initialize gradient buffers
        grad_density = jt.zeros_like(self.grid.density_data)
        grad_sh = jt.zeros_like(self.grid.sh_data)
        grad_basis = jt.zeros_like(self.grid.basis_data)
        grad_background = jt.zeros_like(self.grid.background_data) if self.grid.requires_grad_background else None
        
        # 6. Launch backward CUDA kernel
        blocks = (self.rays.origins.shape[0] + TRACE_RAY_CUDA_THREADS - 1) // TRACE_RAY_CUDA_THREADS
        
        render_ray_backward_kernel[blocks, TRACE_RAY_CUDA_THREADS](
            grad_output=grad_output,
            color_cache=self.color_cache, # Stored from forward pass
            rays_origins=self.rays.origins,
            rays_dirs=self.rays.dirs,
            opt=self.opt,
            grad_density=grad_density,
            grad_sh=grad_sh,
            grad_basis=grad_basis,
            grad_background=grad_background
        )
        
        # 7. Launch background backward kernel if needed
        if self.use_background:
            render_background_backward_kernel[blocks, TRACE_RAY_CUDA_THREADS](
                # ... parameters for background backward kernel ...
            )
            
        return grad_density, grad_sh, grad_basis, grad_background
```

**Parameters:**
*   `opt`: `RenderOptions` struct. Typical values for `sigma_thresh` and `stop_thresh` are small positive floats (e.g., 1e-4, 1e-5) to control early ray termination. `step_size` depends on the desired sampling density.

**Edge Cases and Pitfalls:**
*   **Precision:** Use of `--use_fast_math` in the underlying CUDA compilation may affect precision. [S1, unofficial]
*   **Memory Aliases:** The wrapper manages memory aliases for sparse grid data. Incorrect handling of `_links` and `_offset` can lead to out-of-bounds access.
*   **Background Rendering:** If `use_background` is true, the `log_transmit` buffer must be correctly initialized and passed to both forward and backward kernels.

## Subtopics

### Slab-based Raycasting
Uses slab-based traversal for efficient ray-grid intersection. Choose when the grid is dense and regular. [11383-slab-based-raycasting.md]

### GPU-accelerated ray casting
General GPU acceleration techniques. Choose when hardware acceleration is needed but specific sparse grid optimizations are not required. [12256-gpu-accelerated-ray-casting.md]

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and samples per ray. Early ray termination (controlled by `sigma_thresh` and `stop_thresh`) can reduce the average number of samples.
*   **Memory Complexity:** Linear with respect to the number of non-zero grid cells and rays. The sparse grid representation helps reduce memory usage compared to dense volumes.
*   **Performance:** High performance on NVIDIA GPUs due to CUDA optimization. Performance scales with the number of CUDA cores and memory bandwidth.
*   **Strengths:** Efficient handling of sparse data, integration with Jittor's autograd, support for background rendering.
*   **Limitations:** Requires NVIDIA GPU, dependent on Jittor framework, potential precision issues with fast math.
*   **When to use:** When using Jittor for differentiable volume rendering with sparse data and needing GPU acceleration.
*   **When not to use:** When using other frameworks (PyTorch, TensorFlow) or when CPU-only execution is required.

## Background
The Jittor Volume Render Cuvol Wrapper is part of the JNeRF project, an unofficial implementation of Neural Radiance Fields (NeRF) using the Jittor deep learning framework [S1, unofficial]. It was developed to leverage Jittor's custom function capabilities for efficient GPU-accelerated volume rendering with automatic differentiation. The method builds on the [CUDA GPU Implementation](13016-cuda-gpu-implementation.md) to provide a seamless integration with Jittor's computational graph.

## Sources
[S1] Jittor/JNeRF, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
