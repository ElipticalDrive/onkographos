# Gradient Accumulation Strategy

Also called: Atomic Accumulation, Parallel Gradient Reduction

## Overview
The Gradient Accumulation Strategy is a memory-efficient technique for computing gradients in differentiable volume rendering, specifically within the Adjoint Method framework. It solves the problem of high memory bandwidth and storage costs associated with storing per-sample gradients for every ray and sample. It sets itself apart from the [Inversion Trick](964-inversion-trick.md) by avoiding the inversion of the rendering equation, and from the [Autograd Wrapper for Volume Rendering](2158-autograd-wrapper-for-volume-rendering.md) by explicitly managing the reduction of gradient variables across parallel threads rather than relying on automatic differentiation frameworks to handle the reduction implicitly.

## How it works
The method accumulates adjoint variables (gradients) for input parameters by first reducing them locally within a thread or thread block, and then aggregating these local sums into global memory using atomic operations. This minimizes global memory contention and bandwidth usage.

### Inputs and Outputs
- **Inputs**: 
  - Gradients of the loss with respect to the rendered image ($\frac{\partial L}{\partial I}$).
  - Forward pass intermediate data (or recomputed values) required to calculate local gradients.
  - Parameters to optimize: Camera pose, step size, Transfer Function (TF) coefficients, or volume densities.
- **Outputs**: 
  - Accumulated gradients for the parameters, stored in global memory.

### Data Structures
1. **Local Registers**: Used for per-ray parameters (camera position, step size). Each thread maintains a private accumulator.
2. **Shared Memory**: Used for Transfer Function (TF) parameters. Each thread block maintains a shared accumulator for TF entries accessed by threads in that block.
3. **Global Memory**: Final storage for the accumulated gradients.

### Numbered Steps
1. **Initialize Local Accumulators**: For each ray processed by a thread, initialize local register variables for camera/step-size gradients to zero. Initialize shared memory buffers for TF gradients to zero.
2. **Forward Pass (or Recomputation)**: Perform the volume rendering march. If using the adjoint method, store necessary intermediate values (e.g., transmittance $T_i$) or recompute them on-the-fly to save memory [S1].
3. **Compute Local Gradients**: 
   - For each sample $i$ along the ray, compute the local gradient contribution for the current parameter based on the adjoint variable $\bar{y}$ (gradient of loss w.r.t. output) and the derivative of the rendering equation.
   - For TF parameters, the gradient is computed only for the currently accessed texel [S1].
4. **Accumulate into Local/Shared Memory**:
   - Add the local gradient contribution to the thread's local register accumulator for camera/step-size.
   - Add the local gradient contribution to the shared memory accumulator for the specific TF entry index.
5. **Block-Level Reduction (TF only)**: After all samples for the rays in a block are processed, perform a reduction within the thread block to sum the shared memory values. This step ensures all threads in the block contribute to the final TF gradient [S1].
6. **Global Atomic Accumulation**:
   - Once all rays have been traversed (for camera, step size, TF) or upon exiting the current voxel cell (for volume densities), use global atomic add operations to accumulate the local/block-level sums into the global gradient buffer [S1].
   - For volume densities, this occurs on exit of the current cell [S1].

### Formulas
Let $\bar{I}$ be the gradient of the loss with respect to the image.
Let $\theta$ be a parameter (e.g., TF coefficient, camera pose).
Let $\frac{\partial I}{\partial \theta}$ be the derivative of the image with respect to the parameter.

The total gradient is:
$$ \frac{\partial L}{\partial \theta} = \sum_{\text{rays}} \sum_{\text{samples}} \bar{I} \cdot \frac{\partial I}{\partial \theta} $$

In the accumulation strategy, this sum is split:
$$ \frac{\partial L}{\partial \theta} = \text{AtomicAdd}\left( \text{GlobalGrad}_\theta, \sum_{\text{local}} \bar{I} \cdot \frac{\partial I}{\partial \theta} \right) $$

For TF parameters, the derivative is sparse: only the texel $t$ accessed at sample $i$ receives a non-zero gradient contribution [S1].

### Parameters
- **Atomic Function**: Standard CUDA `atomicAdd` or equivalent for the target hardware.
- **Shared Memory Size**: Sized to hold the number of TF entries or a subset thereof, depending on the block size and TF resolution.

### Edge Cases and Pitfalls
- **Precision**: Accumulating many small gradients into a single global variable can lead to precision loss. Using local registers mitigates this for per-ray parameters.
- **Atomic Contention**: High contention on global atomic operations for frequently accessed parameters (like TF entries) can become a bottleneck. Shared memory reduction helps alleviate this for TF.
- **$\alpha = 1$**: In practice, $\alpha = 1$ (full opacity) is not possible because it requires infinite absorption in the TF, which can lead to numerical instability or division by zero in gradient calculations [S1].
- **Memory Management**: Storing all intermediate values is memory-prohibitive. The method relies on recomputing values on-the-fly or storing only critical variables (like transmittance) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Gradient Accumulation Strategy
# Assumes a CUDA-like kernel structure

def gradient_accumulation_kernel(ray_data, tf_data, global_grads):
    # Shared memory for TF gradients (size depends on TF resolution or block optimization)
    shared_tf_grads = shared_memory.allocate(size=tf_data.num_entries)
    
    # Local registers for per-ray parameters
    local_camera_grad = 0.0
    local_stepsize_grad = 0.0
    
    # Initialize shared memory
    for i in range(tf_data.num_entries):
        shared_tf_grads[i] = 0.0
        
    # Forward pass / Adjoint pass
    for sample in ray_data.samples:
        # Compute forward values (or retrieve stored values)
        density = sample.density
        tf_color = sample.tf_color
        transmittance = sample.transmittance
        
        # Compute local gradient contributions
        # Example: Gradient w.r.t. TF entry
        tf_entry_index = sample.tf_entry_index
        tf_grad_contribution = compute_tf_gradient(sample, global_grads.image_grad)
        
        # Accumulate into shared memory
        atomic_add(shared_tf_grads, tf_entry_index, tf_grad_contribution)
        
        # Example: Gradient w.r.t. camera/stepsize
        camera_grad_contribution = compute_camera_gradient(sample, global_grads.image_grad)
        local_camera_grad += camera_grad_contribution
        
        stepsize_grad_contribution = compute_stepsize_gradient(sample, global_grads.image_grad)
        local_stepsize_grad += stepsize_grad_contribution
        
    # Block-level reduction for TF (if needed, or rely on atomicAdd to shared memory)
    # Note: atomicAdd to shared memory is not standard; usually a reduction is performed
    # Here we assume shared_tf_grads are reduced to a single value per entry per block
    # or that atomicAdd to global memory is done directly from shared memory after reduction.
    
    # Global Atomic Accumulation
    # For camera/stepsize (per ray, accumulated once per ray)
    atomic_add(global_grads.camera, local_camera_grad)
    atomic_add(global_grads.stepsize, local_stepsize_grad)
    
    # For TF (accumulated once per block or per ray, depending on implementation)
    # If shared memory was used for reduction:
    for i in range(tf_data.num_entries):
        atomic_add(global_grads.tf, i, shared_tf_grads[i])
        
    # For volume densities (accumulated on exit of current cell)
    # This logic is typically inside the marching loop, not at the end
    # atomic_add(global_grads.densities, cell_id, local_density_grad)
```

## Subtopics

### [Adjoint Beer-Lambert Marching](1905-adjoint-beer-lambert-marching.md)
Adjoint Beer-Lambert Marching computes the gradient of the Beer-Lambert line integral with respect to pose parameters in a single forward-like pass, avoiding the storage or recomputation of per-iteration state required by naive reverse-mode differentiation of the marching loop. It differs from the naive reverse-mode transform of the marching loop, which synthesizes per-iteration state handling, by costing roughly one forward march (1-1.5x forward cost) instead of ~7x, providing a significant speedup and eliminating the need to re-render the forward intensity.

### [Inference Fast Path for Volume Rendering](2192-inference-fast-path-for-volume-rendering.md)
The Inference Fast Path is a compile-time specialization of the volume rendering kernel that eliminates backward-only output storage and computation when gradients are not required. It differs from the register accumulation stage by removing the largest remaining source of global-memory traffic: the per-sample `outWs` store and the host-side zero-initialization of the $N$-sized buffer.

### Sample Matching
Sample Matching is a differentiable volume rendering estimator that implements sample matching to reduce variance in gradient estimation for volumetric path tracing [S2, unofficial]. It differs from standard adjoint methods by using a two-stage adjoint computation where probes run in small dedicated kernels rather than inside the OptiX megakernel (enabled by `defer_probes=True`) [S2, unofficial]. It also supports a fused C++ walk (`real_interaction_fused=True`) that fuses the majorant-supergrid DDA with accept-until-real null tracking, which is faster than the stock `prbvolpath` because null collisions do not spin the 'fat kernel' [S2, unofficial]. The estimator offers variants: `prbvolpath_sm` (O(n²) quad cost), `linear_cost=True` (O(n) linear cost using one reservoir-picked suffix per path), and `segment_reservoir=4` (bounded cost, exact for ≤4 segments) [S2, unofficial].

## Operating characteristics
- **Time Complexity**: The complexity is largely constant with respect to the number of parameters because only the derivative of the currently accessed parameter (e.g., TF texel) is computed in the adjoint code, unlike the forward method where derivatives of all TF entries need to be propagated [S1].
- **Memory Complexity**: Significantly reduced compared to storing all per-sample gradients. Local registers and shared memory minimize global memory usage.
- **Performance**: The use of atomic operations can introduce contention, but local accumulation mitigates this. The method is efficient for optimizing parameters with sparse gradient contributions (like TF).
- **Strengths**: Memory efficient, scalable to large scenes and high-resolution TFs.
- **Limitations**: Atomic contention can be a bottleneck for highly accessed parameters. Precision issues may arise from accumulating many small values.

## Background
The Gradient Accumulation Strategy is a key component of differentiable volume rendering, particularly in the context of the Adjoint Method. It was introduced in the paper "Differentiable Direct Volume Rendering" [S1]. The method addresses the memory bottlenecks of naive reverse-mode differentiation by leveraging parallel reduction techniques (atomic operations) to aggregate gradients efficiently.

## Sources
[S1] Differentiable Direct Volume Rendering, https://arxiv.org/abs/2107.12672v1, fetched 2026-10-26
[S2] AuroraRyan0301/SampleMatching-mi39, https://github.com/AuroraRyan0301/SampleMatching-mi39, fetched 2026-10-26 (unofficial)
