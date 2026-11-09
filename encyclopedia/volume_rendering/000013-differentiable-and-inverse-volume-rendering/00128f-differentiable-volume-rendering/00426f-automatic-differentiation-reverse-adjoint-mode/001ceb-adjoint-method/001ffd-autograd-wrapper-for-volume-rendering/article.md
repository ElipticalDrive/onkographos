# Autograd Wrapper for Volume Rendering

Also called: Python autograd wrapper for volume rendering, RenderEquation autograd function, torch.autograd.Function wrapper, torch.autograd.Function wrapper for volume rendering

## Overview
The Autograd Wrapper for Volume Rendering is a Python `torch.autograd.Function` implementation that serves as the interface between PyTorch's automatic differentiation engine and the underlying C++/CUDA volume rendering kernels. It solves the problem of integrating differentiable volume rendering into deep learning pipelines by routing execution to either a backward-aware CUDA kernel (when gradients are required) or an inference-optimized forward-only kernel (when they are not) [S2, unofficial]. It replaces the previous C++-based autograd layer, moving the gradient management logic into Python to reduce compilation overhead and improve maintainability [S2, unofficial].

## How it works
The wrapper implements the `torch.autograd.Function` interface, defining `forward` and `backward` static methods. It interacts with the C++ library `fvdb` via Python bindings (`_fvdb_cpp`) exposed in `src/python/Bindings.cpp` [S2, unofficial].

### Inputs and Outputs
**Inputs:**
The specific inputs depend on the rendering equation variant. For the PBR (Physically Based Rendering) equation, inputs include `base_color`, `roughness`, `metallic`, `normals`, `viewdirs`, `incidents_shs`, `direct_shs`, `visibility_shs`, `sample_num`, `is_training`, and `debug` [S1, unofficial]. For the general volume rendering, inputs are the density/TF parameters and camera/ray parameters required by the C++ `fvdb::volumeRenderForward` function [S2, unofficial].

**Outputs:**
The forward pass returns the rendered image tensor (e.g., `pbr` or `rgb`), `incident_dirs`, and `diffuse_light` (for PBR) or `depth`, `ws` (weight), and `total_samples` (for general volume rendering) [S1, S2, unofficial].

### Data Structures
The wrapper resides in `fvdb/_volume_render.py` as the `_VolumeRenderFn` class [S2, unofficial]. It relies on the C++ public API defined in `FVDB.h`, which exposes `fvdb::volumeRenderForward` and `fvdb::volumeRenderBackward` [S2, unofficial]. The C++ kernels operate on tensors with channel counts $C$ where $1 \le C \le \text{MAX\_VOLUME\_RENDER\_CHANNELS}$ [S2, unofficial].

### Algorithm Steps

1.  **Gradient Check:** In the `forward` method, check if gradients are enabled and if at least one input tensor has `requires_grad=True` [S2, unofficial].
2.  **Dispatch:**
    *   **If gradients are required:** Call `_VolumeRenderFn.apply` (or the internal logic) which invokes the CUDA forward pass with `needsBackward=true`. This path saves necessary intermediate variables (e.g., `sample_num`, `debug`, input tensors excluding `diffuse_light`) to `ctx` for the backward pass [S1, S2, unofficial].
    *   **If gradients are NOT required (Inference Fast Path):** Call `_fvdb_cpp.volume_render_fwd(..., needs_backward=False)` directly. This path skips saving intermediates and kernel launch overhead associated with backward tracking [S2, unofficial].
3.  **Forward Computation:**
    *   The C++ kernel `fvdb::volumeRenderForward` performs the volume rendering. It uses per-ray register accumulation for `rgb`, `depth`, and `opacity` to minimize memory writes [S2, unofficial].
    *   It includes an early-termination condition where ray marching stops if the transmittance $T$ drops below a threshold `transmittanceThresh` [S2, unofficial].
4.  **Output Handling:**
    *   **Gradient Path:** Return the full output tensors (`rgb`, `opacity`, `depth`, `ws`, `total_samples`) [S2, unofficial].
    *   **Inference Path:** Return the primary outputs (e.g., `rgb`, `opacity`). For `depth`, `ws`, and `total_samples`, return size-0 placeholder tensors with matching `dtype` and `device` [S2, unofficial].
5.  **Backward Pass:**
    *   If the forward pass used the gradient path, the `backward` method is invoked.
    *   Retrieve saved tensors from `ctx` [S1, unofficial].
    *   Handle `None` gradients for outputs by coercing them into zero tensors with matching shape, dtype, and device [S2, unofficial].
    *   Call `_C.render_equation_backward` (or `_fvdb_cpp.volume_render_bwd`) to compute gradients with respect to the inputs [S1, S2, unofficial].
    *   Return a tuple of gradients matching the forward inputs, padding with `None` for unused inputs [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import _fvdb_cpp

class _VolumeRenderFn(torch.autograd.Function):
    @staticmethod
    def forward(ctx, *args):
        # Determine if backward is needed
        # args[0] is assumed to be the first differentiable input
        needs_backward = torch.is_grad_enabled() and any(
            hasattr(arg, 'requires_grad') and arg.requires_grad for arg in args
        )

        if needs_backward:
            # Save tensors for backward pass
            # Specific saved tensors depend on the variant (e.g., sample_num, debug, inputs)
            ctx.save_for_backward(*args)
            
            # Call C++ forward with needsBackward=true
            # The exact signature depends on the specific rendering equation
            # For PBR:
            # pbr, incident_dirs, diffuse_light = _fvdb_cpp.volume_render_fwd(
            #     base_color, roughness, metallic, normals, viewdirs, 
            #     incidents_shs, direct_shs, visibility_shs, sample_num, 
            #     is_training, debug, needs_backward=True
            # )
            
            # For general volume rendering:
            # rgb, opacity, depth, ws, total_samples = _fvdb_cpp.volume_render_fwd(
            #     inputs..., needs_backward=True
            # )
            
            # Note: The actual C++ call structure is abstracted here.
            # The key is that intermediates are saved and the kernel knows to prepare for backward.
            pass 
        else:
            # Inference fast path
            # Call C++ forward with needsBackward=False
            # Returns size-0 placeholders for depth, ws, total_samples
            pass

        return outputs

    @staticmethod
    def backward(ctx, grad_output):
        # Retrieve saved tensors
        saved_tensors = ctx.saved_tensors
        
        # Handle None gradients from upstream
        # Coerce None to zero tensors with matching shape, dtype, device
        if grad_output is None:
            # Create zero tensor matching the expected gradient shape
            # This step is critical for compatibility with autograd
            pass

        # Call C++ backward kernel
        # grad_inputs = _fvdb_cpp.volume_render_bwd(saved_tensors, grad_output)
        
        return grad_inputs
```

### Parameters and Edge Cases
*   **`needsBackward` flag:** Controls whether the C++ kernel saves intermediates. Crucial for performance in inference [S2, unofficial].
*   **`transmittanceThresh`:** Early termination threshold for ray marching. If $T \le \text{transmittanceThresh}$, the ray is terminated [S2, unofficial].
*   **Channel Limit:** $1 \le C \le \text{MAX\_VOLUME\_RENDER\_CHANNELS}$ [S2, unofficial].
*   **Inference Fast Path Pitfall:** The inference path returns size-0 placeholders for `depth`, `ws`, and `total_samples`. Tests asserting these values must force the backward-aware path (by setting `requires_grad=True` on an input) to be observationally equivalent [S2, unofficial].

## Subtopics

### [Differentiable Adaptive Stepsize Rendering](2194-differentiable-adaptive-stepsize-rendering.md)
Differentiable Adaptive Stepsize Rendering is a PyTorch `torch.autograd.Function` that computes the forward pass of adaptive stepsize volume rendering and approximates the backward pass using finite differences to enable gradient-based optimization of stepsize maps. It treats the renderer as a black box, trading computational efficiency for implementation simplicity compared to adjoint-based methods that require deriving analytical gradients for the rendering equation.

### [PyTorch Interface](2902-pytorch-interface.md)
The PyTorch Interface exposes differentiable forward and back projection operations for X-ray computed tomography as a `torch.nn.Module` subclass. It wraps custom C++/CUDA extensions to enable end-to-end automatic differentiation without manual gradient coding. It solves the problem of integrating differentiable projection operations into deep learning pipelines.

### [C++/CUDA Implementation of Moment Bounds](8857-c-cuda-implementation-of-moment-bounds.md)
A hand-written C++/CUDA implementation of the primal (forward) and adjoint (backward) algorithms for computing moment bounds, designed to avoid the overhead of automatic differentiation (AD) graph traversal and JIT compilation limitations. It builds on [Autograd Wrapper for Volume Rendering](2158-autograd-wrapper-for-volume-rendering.md) by providing the low-level compute kernels that the wrapper invokes. Unlike AD-based approaches, it generates highly optimized code by knowing the floating-point type and moment order $n$ at compile time.

### C++ Adjoint Implementation
This variant implements primal and adjoint algorithms entirely in C++, depending only on a small subset of the standard library [S3]. The floating point type and $n$ are known at compile time to generate highly optimized code [S3]. It avoids automatic differentiation frameworks like PyTorch which materialize intermediate results in memory and invoke dedicated kernels with launch overhead [S3].

## Operating characteristics
*   **Performance:** Moving the autograd layer to Python reduced the runtime of the `volume_render` operation from 60.7% to 27.1% of the total pipeline time, shifting the bottleneck to `sample_trilinear` [S2, unofficial].
*   **Memory:** The C++/CUDA variant for moment bounds avoids excessive storage requirements by keeping intermediate values in registers, avoiding register spilling to local memory [S3]. The PyTorch AD approach materializes intermediate results in memory, which can be expensive [S3].
*   **Correctness:** The backward-aware path produces gradients identical to the previous C++ autograd implementation, ensuring correctness is unchanged [S2, unofficial].
*   **Inference:** The inference fast path is observationally equivalent for callers consuming only `rgb` and `opacity`, but requires specific handling for tests checking `ws`/`total_samples` [S2, unofficial].

## Background
The wrapper was introduced in the `fvdb-core` project to optimize the `volume_render` operation [S2, unofficial]. It replaced a C++-based autograd layer (`src/fvdb/detail/autograd/VolumeRender.{cpp,h}`) with a Python `torch.autograd.Function` implementation (`fvdb/_volume_render.py`) to match the pattern of other functional ops like `_gaussian_autograd.py` [S2, unofficial]. This change was part of a broader effort to improve performance and maintainability in differentiable volume rendering pipelines [S2, unofficial]. The underlying C++ kernels (`fvdb::volumeRenderForward` and `fvdb::volumeRenderBackward`) are documented in `FVDB.h` and `detail/ops/VolumeRender.h` [S2, unofficial].

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] Optimizations for `volume_render` and move its autograd layer to Python, https://github.com/openvdb/fvdb-core/pull/640, fetched 2026-10-27 (unofficial)
[S3] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
