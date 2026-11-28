# Gradient Accumulation Strategy

Also called: Atomic Accumulation, Parallel Gradient Reduction

## Overview
The Gradient Accumulation Strategy aggregates per-ray adjoint variables into global parameter gradients within the Adjoint Method [S1]. It solves the parallel reduction problem in GPU-based volume rendering, where thousands of rays compute independent gradient contributions simultaneously [S1]. It differs from naive global atomic accumulation by introducing a two-stage hierarchy: first accumulating into per-ray local registers or per-block shared memory to minimize global memory contention, then performing a single atomic reduction per ray or cell [S1]. This approach significantly reduces the number of global atomic operations compared to accumulating directly into global memory for every sample [S1].

## How it works
The method aggregates adjoint variables (gradients) for optimization parameters [S1]. The strategy distinguishes between parameters that vary per-ray (camera pose, step size) and those that are shared across rays (Transfer Function weights, volume density fields) [S1].

**Inputs and Outputs:**
- **Input:** Per-ray adjoint variables computed during the backward pass [S1].
- **Output:** Global gradient buffer for optimization parameters [S1].

**Data Structures:**
1. **Local Registers:** Per-thread storage for camera position, step size, and volume densities to avoid atomic contention during the ray march [S1].
2. **Shared Memory:** Per-block storage for Transfer Function (TF) parameters to allow fast access and accumulation within a thread block [S1].
3. **Global Memory:** The final parameter gradient buffer [G].

**Numbered Steps:**
1. **Initialization:** Initialize local register accumulators to zero at the start of the ray march or TF lookup [G].
2. **Per-Step Accumulation:** During the backward pass, when a parameter is accessed (e.g., a TF texel or a density value), compute its local gradient contribution and add it to the local register (camera/step size/densities) or shared memory (TF) [S1].
3. **Per-Cell/Per-Ray Finalization:** 
   - **Camera/Step Size:** After the ray traversal is complete, the local register holds the total gradient for that ray [S1].
   - **Volume Densities:** On exit of the current cell (if using hierarchical acceleration structures), the local register holds the gradient for that cell [S1].
   - **Transfer Function:** After all threads in the block have finished processing their rays, perform a block-wide reduction in shared memory to compute the total gradient for each TF parameter [S1].
4. **Global Atomic Reduction:** 
   - For camera/step size/densities: Use a global atomic add function to add the per-ray/cell gradient to the global gradient buffer [S1].
   - For TF: Use a global atomic add function to add the block-reduced gradient to the global gradient buffer [S1].

**Formulas:**
Let $\bar{\theta}_{ray}$ be the gradient accumulated in local registers for a specific ray.
Let $\bar{\theta}_{block}$ be the gradient accumulated in shared memory for a block.
Let $\nabla_\theta$ be the global gradient buffer.

For camera/step size/densities:
$$ \nabla_\theta \leftarrow \nabla_\theta + \bar{\theta}_{ray} $$
This is performed using atomic addition: `atomicAdd(&global_grad, local_grad)` [S1].

For TF:
$$ \nabla_\theta \leftarrow \nabla_\theta + \bar{\theta}_{block} $$
This is performed using atomic addition: `atomicAdd(&global_grad, block_grad)` [S1].

**Parameters:**
- **Atomic Function:** `atomicAdd` (CUDA/HIP) [S1].
- **Memory Scope:** `memory_space::global` for final reduction [G].

**Edge Cases and Pitfalls:**
- **Atomic Contention:** Direct global atomic accumulation per sample causes severe contention; the two-stage approach mitigates this [S1].
- **Precision:** Accumulating many small gradients in local registers may suffer from floating-point precision loss if the range of values is large, but for typical TF parameters this is negligible [S1].
- **Initialization:** Failure to zero-initialize local registers leads to garbage gradients [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def accumulate_gradients(ray_idx, block_idx, adjoint_vars, global_grad_buffer, shared_mem_buffer):
    """
    Accumulates gradients for parameters from adjoint variables.
    Args:
        ray_idx: Unique index for the current ray.
        block_idx: Unique index for the current thread block.
        adjoint_vars: Dictionary of per-step gradient contributions.
                      Keys: 'camera', 'step_size', 'tf', 'density'
                      Values: float or list of floats.
        global_grad_buffer: Mutable dictionary simulating global memory.
        shared_mem_buffer: Pre-allocated list simulating shared memory, zero-initialized per block.
    """
    # Stage 1: Per-ray or Per-block accumulation
    local_grads = {}
    
    # Camera and Step Size: Accumulate in local registers
    if 'camera' in adjoint_vars:
        local_grads['camera'] = adjoint_vars['camera']
    if 'step_size' in adjoint_vars:
        local_grads['step_size'] = adjoint_vars['step_size']
        
    # Volume Densities: Accumulate in local registers (per cell)
    if 'density' in adjoint_vars:
        local_grads['density'] = adjoint_vars['density']
        
    # Transfer Function: Accumulate in shared memory
    if 'tf' in adjoint_vars:
        for i, tf_param in enumerate(adjoint_vars['tf']):
            shared_mem_buffer[i] += tf_param
            
    # Stage 2: Global Atomic Reduction
    # This occurs after the ray march or block processing is complete
    
    # Camera and Step Size
    if 'camera' in local_grads:
        global_grad_buffer['camera'] += local_grads['camera']
    if 'step_size' in local_grads:
        global_grad_buffer['step_size'] += local_grads['step_size']
        
    # Volume Densities
    if 'density' in local_grads:
        global_grad_buffer['density'] += local_grads['density']
        
    # Transfer Function
    if 'tf' in adjoint_vars:
        # Perform block-wide reduction in shared memory to get total tf_grads
        block_total_tf_grads = [0.0] * len(shared_mem_buffer)
        for i in range(len(shared_mem_buffer)):
            block_total_tf_grads[i] = shared_mem_buffer[i]
        # In a real GPU kernel, this would be a warp/block shuffle reduction.
        # Here we simulate the final atomic add for the block representative.
        if block_idx == 0:
            for i, grad in enumerate(block_total_tf_grads):
                global_grad_buffer['tf'][i] += grad
```

## Subtopics

### [Adjoint Beer-Lambert Marching](1905-adjoint-beer-lambert-marching.md)
Adjoint Beer-Lambert Marching computes the gradient of the Beer-Lambert line integral with respect to pose parameters in a single forward-like pass, avoiding the storage or recomputation of per-iteration state required by naive reverse-mode differentiation of the marching loop. It differs from the naive reverse-mode transform of the marching loop, which synthesizes per-iteration state handling, by costing roughly one forward march (1-1.5x forward cost) instead of ~7x, providing a significant speedup and eliminating the need to re-render the forward intensity.

### [Inference Fast Path for Volume Rendering](2192-inference-fast-path-for-volume-rendering.md)
The Inference Fast Path is a compile-time specialization of the volume rendering kernel that eliminates backward-only output storage and computation when gradients are not required. It differs from the register accumulation stage by removing the largest remaining source of global-memory traffic: the per-sample `outWs` store and the host-side zero-initialization of the $N$-sized buffer.

### [Sample Matching](4626-sample-matching.md)
Sample Matching is a differentiable volume rendering estimator that implements sample matching to reduce variance in gradient estimation for volumetric path tracing. It solves the high-variance gradient problem in differentiable volume rendering by estimating the transmittance and scattering terms of the Differentiable Radiative Transfer Equation (DRTE) at the same spatial points on each path segment, causing their opposite-sign contributions to cancel. Unlike the base `prbvolpath` method which estimates these terms at unrelated points, Sample Matching confines all modifications to the adjoint pass while keeping the primal image rendering identical.

### Half2 Precision Gradient Accumulation
Uses half2 data type to pack two FP16 values for gradient accumulation, reducing memory traffic and improving throughput while preserving accuracy compared to unpacked atomics [S2, unofficial]. Per-parameter gradients are accumulated per pixel directly into packed half2 buffers via half2 atomicAdd for pairs: (m_x, m_y), (α, scale), (θ, cr), and (cg, cb) [S2, unofficial]. A lightweight post-pass unpacks the half2 buffers to legacy FP16 arrays [S2, unofficial]. Half atomicAdd matched kernel time but degraded numerical accuracy; half atomicCAS recovered accuracy at prohibitive cost; packed half2 atomics preserved accuracy, improved throughput, and reduced memory traffic [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ for the forward/backward pass, plus $O(N_{rays})$ for the final atomic reductions [S1].
- **Memory Complexity:** $O(N_{params})$ for the global gradient buffer; local registers add $O(1)$ per thread; shared memory adds $O(N_{TF})$ per block [S1].
- **Measured Performance:** The two-stage approach reduces global atomic contention, leading to higher throughput compared to direct global accumulation [S1].
- **Strengths:** High parallel efficiency and low global memory bandwidth usage for gradient updates [S1].
- **Limitations:** Requires careful management of shared memory for TF parameters; atomic operations can still bottleneck if the number of unique parameters is small and contention is high [S1].
- **When to Use:** When optimizing parameters that are shared across many rays (TF, density fields) or vary per-ray (camera, step size) in a differentiable volume rendering pipeline [S1].
- **When Not to Use:** When the number of parameters is very small and the overhead of shared memory management outweighs the benefits, or when gradients are not required [G].

## Background
The Gradient Accumulation Strategy is a core component of the Adjoint Method for differentiable volume rendering, as described in "Differentiable Direct Volume Rendering" [S1]. It addresses the inefficiency of naive reverse-mode differentiation, which would require storing all intermediate values or recomputing them, leading to high memory usage or computational cost [S1]. By accumulating gradients locally and reducing them globally, it enables efficient optimization of volume rendering parameters on GPUs [S1].

## Sources
[S1] Differentiable Direct Volume Rendering, https://arxiv.org/abs/2107.12672v1, fetched 2026-10-26
[S2] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
