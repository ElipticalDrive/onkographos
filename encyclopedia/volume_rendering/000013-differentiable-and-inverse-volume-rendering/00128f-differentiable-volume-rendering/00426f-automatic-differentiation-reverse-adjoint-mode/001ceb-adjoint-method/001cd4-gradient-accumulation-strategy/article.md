# Gradient Accumulation Strategy

Also called: Atomic Accumulation, Parallel Gradient Reduction

## Overview
The Gradient Accumulation Strategy computes gradients for differentiable volume rendering by first reducing adjoint variables locally per ray or thread block, then aggregating them into global memory via atomic operations [S1]. It avoids the high memory bandwidth and storage costs of storing per-sample gradients for every ray and sample [G]. It differs from the [Inversion Trick](964-inversion-trick.md) by not inverting the rendering equation, and from the [Autograd Wrapper for Volume Rendering](2158-autograd-wrapper-for-volume-rendering.md) by explicitly managing parallel gradient reduction rather than relying on framework-level implicit reduction [G].

## How it works
### Inputs and Outputs
- **Inputs**: Gradients of the loss with respect to the rendered image, forward pass intermediate data (recomputed on-the-fly), and parameters to optimize (camera pose, step size, Transfer Function coefficients, volume densities) [S1].
- **Outputs**: Accumulated gradients for the parameters, stored in global memory [S1].

### Data Structures
1. **Local Registers**: Used for per-ray parameters like camera position and step size to avoid atomic contention during the ray march [S1].
2. **Shared Memory**: Used for Transfer Function (TF) parameters to allow fast access and accumulation within a thread block [S1].
3. **Global Memory**: Final storage for the accumulated gradients [S1].

### Numbered Steps
1. **Initialize Local Accumulators**: For each ray processed by a thread, initialize local register variables for camera/step-size gradients to zero. Initialize shared memory buffers for TF gradients to zero [G].
2. **Forward Pass (or Recomputation)**: Perform the volume rendering march. All intermediate values are recomputed on-the-fly rather than stored [S1].
3. **Compute Local Gradients**: For each sample along the ray, compute the local gradient contribution for the current parameter based on the adjoint variable (gradient of loss w.r.t. output) and the derivative of the rendering equation. For TF parameters, only the derivative of the currently accessed texel is computed when accessed in the adjoint code [S1].
4. **Accumulate into Local/Shared Memory**: Add the local gradient contribution to the thread's local register accumulator for camera/step-size. Add the local gradient contribution to the shared memory accumulator for the specific TF entry index [S1].
5. **Global Atomic Accumulation**: Once all rays have been traversed (for camera, step size, TF) or upon exiting the current voxel cell (for volume densities), use global atomic add operations to accumulate the local/block-level sums into the global gradient buffer [S1].

### Formulas
Let $\bar{I}$ be the gradient of the loss with respect to the image.
Let $\theta$ be a parameter (e.g., TF coefficient, camera pose).
The total gradient is computed by splitting the summation across parallel threads:
$$ \frac{\partial L}{\partial \theta} = \sum_{\text{rays}} \sum_{\text{samples}} \bar{I} \cdot \frac{\partial I}{\partial \theta} $$
In the accumulation strategy, this sum is split into local reductions followed by global atomic addition [S1]:
$$ \frac{\partial L}{\partial \theta} = \text{AtomicAdd}\left( \text{GlobalGrad}_\theta, \sum_{\text{local}} \bar{I} \cdot \frac{\partial I}{\partial \theta} \right) $$
For TF parameters, the derivative is sparse: only the texel $t$ accessed at sample $i$ receives a non-zero gradient contribution [S1].

### Parameters
- **Atomic Function**: Standard CUDA `atomicAdd` or equivalent for the target hardware [G].
- **Shared Memory Size**: Sized to hold the number of TF entries or a subset thereof, depending on the block size and TF resolution [G].

### Edge Cases and Pitfalls
- **Precision**: Accumulating many small gradients into a single global variable can lead to precision loss. Using local registers mitigates this for per-ray parameters [G].
- **Atomic Contention**: High contention on global atomic operations for frequently accessed parameters (like TF entries) can become a bottleneck. Shared memory reduction helps alleviate this for TF [G].
- **$\alpha = 1$**: In practice, $\alpha = 1$ (full opacity) is not possible because it requires infinite absorption in the TF, which can lead to numerical instability or division by zero in gradient calculations [S1].
- **Memory Management**: Storing all intermediate values is memory-prohibitive. The method relies on recomputing values on-the-fly [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_accumulation_kernel(ray_data, tf_data, global_grads):
    # Shared memory for TF gradients
    shared_tf_grads = [0.0] * tf_data.num_entries
    
    # Local registers for per-ray parameters
    local_camera_grad = 0.0
    local_stepsize_grad = 0.0
    
    # Forward pass / Adjoint pass
    for sample in ray_data.samples:
        # Compute forward values (recomputed on-the-fly)
        density = sample.density
        tf_color = sample.tf_color
        transmittance = sample.transmittance
        
        # Compute local gradient contributions
        tf_entry_index = sample.tf_entry_index
        # Standard adjoint step: multiply output gradient by local derivative
        tf_grad_contribution = sample.image_grad * sample.d_tf_d_param
        
        # Accumulate into shared memory
        shared_tf_grads[tf_entry_index] += tf_grad_contribution
        
        # Example: Gradient w.r.t. camera/stepsize
        camera_grad_contribution = sample.image_grad * sample.d_camera_d_param
        local_camera_grad += camera_grad_contribution
        
        stepsize_grad_contribution = sample.image_grad * sample.d_stepsize_d_param
        local_stepsize_grad += stepsize_grad_contribution
        
    # Global Atomic Accumulation
    # For camera/stepsize (per ray, accumulated once per ray)
    global_grads.camera += local_camera_grad
    global_grads.stepsize += local_stepsize_grad
    
    # For TF (accumulated once per block or per ray)
    for i in range(tf_data.num_entries):
        global_grads.tf[i] += shared_tf_grads[i]
        
    # For volume densities (accumulated on exit of current cell)
    # global_grads.densities[cell_id] += local_density_grad
```

## Subtopics

### [Adjoint Beer-Lambert Marching](1905-adjoint-beer-lambert-marching.md)
Adjoint Beer-Lambert Marching computes the gradient of the Beer-Lambert line integral with respect to pose parameters in a single forward-like pass, avoiding the storage or recomputation of per-iteration state required by naive reverse-mode differentiation of the marching loop. It differs from the naive reverse-mode transform of the marching loop, which synthesizes per-iteration state handling, by costing roughly one forward march (1-1.5x forward cost) instead of ~7x, providing a significant speedup and eliminating the need to re-render the forward intensity.

### [Inference Fast Path for Volume Rendering](2192-inference-fast-path-for-volume-rendering.md)
The Inference Fast Path is a compile-time specialization of the volume rendering kernel that eliminates backward-only output storage and computation when gradients are not required. It differs from the register accumulation stage by removing the largest remaining source of global-memory traffic: the per-sample `outWs` store and the host-side zero-initialization of the $N$-sized buffer.

### [Sample Matching](4626-sample-matching.md)
Sample Matching is a differentiable volume rendering estimator that implements sample matching to reduce variance in gradient estimation for volumetric path tracing [G].

## Operating characteristics
- **Time Complexity**: The complexity is largely constant with respect to the number of parameters because only the derivative of the currently accessed parameter (e.g., TF texel) is computed in the adjoint code, unlike the forward method where derivatives of all TF entries need to be propagated [S1].
- **Memory Complexity**: Significantly reduced compared to storing all per-sample gradients. Local registers and shared memory minimize global memory usage [G].
- **Performance**: The use of atomic operations can introduce contention, but local accumulation mitigates this. The method is efficient for optimizing parameters with sparse gradient contributions (like TF) [S1].
- **Strengths**: Memory efficient, scalable to large scenes and high-resolution TFs [G].
- **Limitations**: Atomic contention can be a bottleneck for highly accessed parameters. Precision issues may arise from accumulating many small values [G].

## Background
The Gradient Accumulation Strategy is a key component of differentiable volume rendering, particularly in the context of the Adjoint Method. It was introduced in the paper "Differentiable Direct Volume Rendering" [S1]. The method addresses the memory bottlenecks of naive reverse-mode differentiation by leveraging parallel reduction techniques (atomic operations) to aggregate gradients efficiently [G].

## Sources
[S1] Differentiable Direct Volume Rendering, https://arxiv.org/abs/2107.12672v1, fetched 2026-10-26
