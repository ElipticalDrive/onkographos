# Gradient Accumulation Strategy

Also called: Atomic Accumulation, Parallel Gradient Reduction

## Overview
The Gradient Accumulation Strategy is the mechanism used in the Adjoint Method to aggregate per-ray gradients into global parameter updates. It solves the problem of parallel reduction in GPU-based volume rendering, where thousands of rays compute independent gradient contributions simultaneously. It differs from naive global atomic accumulation by introducing a two-stage hierarchy: first accumulating into per-ray local registers or per-block shared memory to minimize global memory contention, and then performing a single atomic reduction per ray or cell. This approach significantly reduces the number of global atomic operations compared to accumulating directly into global memory for every sample.

## How it works
The method aggregates adjoint variables (gradients) for optimization parameters. The strategy distinguishes between parameters that vary per-ray (camera pose, step size) and those that are shared across rays (Transfer Function weights, volume density fields).

**Inputs and Outputs:**
- **Input:** Per-ray adjoint variables $\bar{\theta}_{ray}$ computed during the backward pass.
- **Output:** Global gradient $\nabla_\theta$ stored in global memory for the optimizer.

**Data Structures:**
1.  **Local Registers:** Per-thread storage for intermediate gradient sums. Size: 1-4 floats depending on parameter count.
2.  **Shared Memory:** Per-block storage for Transfer Function (TF) parameters. Size: $N_{TF}$ floats, where $N_{TF}$ is the number of TF parameters.
3.  **Global Memory:** The final parameter gradient buffer.

**Numbered Steps:**
1.  **Initialization:** Initialize local register accumulators to zero at the start of the ray march or TF lookup.
2.  **Per-Step Accumulation:** During the forward or backward pass, when a parameter $\theta$ is accessed (e.g., a TF texel or a density value), compute its local gradient contribution $\Delta \bar{\theta}$. Add this to the local register accumulator for that parameter.
    - For camera/step size: Accumulate in local registers.
    - For TF: Accumulate in shared memory.
3.  **Per-Cell/Per-Ray Finalization:**
    - **Camera/Step Size:** After the ray traversal is complete, the local register holds the total gradient for that ray.
    - **Volume Densities:** On exit of the current cell (if using hierarchical acceleration structures), the local register holds the gradient for that cell.
    - **Transfer Function:** After all threads in the block have finished processing their rays, perform a block-wide reduction in shared memory to compute the total gradient for each TF parameter.
4.  **Global Atomic Reduction:**
    - For camera/step size/densities: Use a global atomic add function to add the per-ray/cell gradient to the global gradient buffer.
    - For TF: Use a global atomic add function to add the block-reduced gradient to the global gradient buffer.

**Formulas:**
Let $\bar{\theta}_{ray}$ be the gradient accumulated in local registers for a specific ray.
Let $\bar{\theta}_{block}$ be the gradient accumulated in shared memory for a block.
Let $\nabla_\theta$ be the global gradient buffer.

For camera/step size/densities:
$$ \nabla_\theta \leftarrow \nabla_\theta + \bar{\theta}_{ray} $$
This is performed using atomic addition: `atomicAdd(&global_grad, local_grad)`.

For TF:
$$ \nabla_\theta \leftarrow \nabla_\theta + \bar{\theta}_{block} $$
This is performed using atomic addition: `atomicAdd(&global_grad, block_grad)`.

**Parameters:**
- **Atomic Function:** `atomicAdd` (CUDA/HIP).
- **Memory Scope:** `memory_space::global` for final reduction.

**Edge Cases and Pitfalls:**
- **Atomic Contention:** Direct global atomic accumulation per sample causes severe contention. The two-stage approach mitigates this.
- **Precision:** Accumulating many small gradients in local registers may suffer from floating-point precision loss if the range of values is large. However, for typical TF parameters, this is negligible.
- **Initialization:** Failure to zero-initialize local registers leads to garbage gradients.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Gradient Accumulation Strategy
# Assumes CUDA-like parallel execution model

def accumulate_gradients(ray_idx, block_idx, adjoint_vars):
    """
    Accumulates gradients for parameters from adjoint variables.
    
    Args:
        ray_idx: Unique index for the current ray.
        block_idx: Unique index for the current thread block.
        adjoint_vars: Dictionary of per-step gradient contributions.
                      Keys: 'camera', 'step_size', 'tf', 'density'
                      Values: float or list of floats.
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
    # Note: This requires a block-wide reduction step before global atomic
    if 'tf' in adjoint_vars:
        # Assume shared memory buffer 'tf_grads' is allocated per block
        # and initialized to zero at block start
        for i, tf_param in enumerate(adjoint_vars['tf']):
            tf_grads[i] += tf_param
            
    # Stage 2: Global Atomic Reduction
    # This occurs after the ray march or block processing is complete
    
    # Camera and Step Size
    if 'camera' in local_grads:
        atomic_add(global_grad_buffer['camera'], local_grads['camera'])
    if 'step_size' in local_grads:
        atomic_add(global_grad_buffer['step_size'], local_grads['step_size'])
        
    # Volume Densities
    if 'density' in local_grads:
        atomic_add(global_grad_buffer['density'], local_grads['density'])
        
    # Transfer Function
    # This step is performed once per block, after all threads have contributed
    if 'tf' in adjoint_vars:
        # Perform block-wide reduction in shared memory to get total tf_grads
        block_total_tf_grads = reduce_block(tf_grads)
        for i, grad in enumerate(block_total_tf_grads):
            atomic_add(global_grad_buffer['tf'][i], grad)

def atomic_add(target, value):
    """
    Simulates global atomic add operation.
    In CUDA, this is atomicAdd(&target, value).
    """
    target += value
```

## Subtopics

### [Adjoint Beer-Lambert Marching](1905-adjoint-beer-lambert-marching.md)
Adjoint Beer-Lambert Marching computes the gradient of the Beer-Lambert line integral with respect to pose parameters in a single forward-like pass, avoiding the storage or recomputation of per-iteration state required by naive reverse-mode differentiation of the marching loop. It differs from the naive reverse-mode transform of the marching loop, which synthesizes per-iteration state handling, by costing roughly one forward march (1-1.5x forward cost) instead of ~7x, providing a significant speedup and eliminating the need to re-render the forward intensity.

### [Inference Fast Path for Volume Rendering](2192-inference-fast-path-for-volume-rendering.md)
The Inference Fast Path is a compile-time specialization of the volume rendering kernel that eliminates backward-only output storage and computation when gradients are not required. It differs from the register accumulation stage by removing the largest remaining source of global-memory traffic: the per-sample `outWs` store and the host-side zero-initialization of the $N$-sized buffer.

### [Sample Matching](4626-sample-matching.md)
Sample Matching is a differentiable volume rendering estimator that implements sample matching to reduce variance in gradient estimation for volumetric path tracing. It solves the high-variance gradient problem in differentiable volume rendering by estimating the transmittance and scattering terms of the Differentiable Radiative Transfer Equation (DRTE) at the same spatial points on each path segment, causing their opposite-sign contributions to cancel. Unlike the base `prbvolpath` method which estimates these terms at unrelated points, Sample Matching confines all modifications to the adjoint pass while keeping the primal image rendering identical.

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ for the forward/backward pass, plus $O(N_{rays})$ for the final atomic reductions. The two-stage accumulation reduces the constant factor of the reduction phase significantly.
- **Memory Complexity:** $O(N_{params})$ for the global gradient buffer. Local registers add $O(1)$ per thread. Shared memory adds $O(N_{TF})$ per block.
- **Measured Performance:** The two-stage approach reduces global atomic contention, leading to higher throughput compared to direct global accumulation. The cost of the reduction phase is negligible compared to the forward/backward pass.
- **Strengths:** High parallel efficiency, low global memory bandwidth usage for gradient updates.
- **Limitations:** Requires careful management of shared memory for TF parameters. Atomic operations can still be a bottleneck if the number of unique parameters is small and contention is high.
- **When to Use:** When optimizing parameters that are shared across many rays (TF, density fields) or vary per-ray (camera, step size) in a differentiable volume rendering pipeline.
- **When Not to Use:** When the number of parameters is very small and the overhead of shared memory management outweighs the benefits, or when gradients are not required (use Inference Fast Path).

## Background
The Gradient Accumulation Strategy is a core component of the Adjoint Method for differentiable volume rendering, as described in "Differentiable Direct Volume Rendering" [S1]. It addresses the inefficiency of naive reverse-mode differentiation, which would require storing all intermediate values or recomputing them, leading to high memory usage or computational cost. By accumulating gradients locally and reducing them globally, it enables efficient optimization of volume rendering parameters on GPUs.

## Sources
[S1] Differentiable Direct Volume Rendering, https://arxiv.org/abs/2107.12672v1, fetched 2026-10-26
