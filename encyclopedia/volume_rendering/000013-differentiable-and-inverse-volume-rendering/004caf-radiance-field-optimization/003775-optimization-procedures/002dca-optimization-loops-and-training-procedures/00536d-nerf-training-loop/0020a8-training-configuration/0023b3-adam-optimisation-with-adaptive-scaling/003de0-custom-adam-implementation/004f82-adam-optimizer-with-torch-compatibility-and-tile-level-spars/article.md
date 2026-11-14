# Adam Optimizer with Torch Compatibility and Tile-Level Sparsity

Also called: Adam with zero-gradient replay, adamUpdateTorchCUDA

## Overview
A CUDA implementation of the Adam optimizer designed for differentiable volume rendering that maintains strict behavioral compatibility with PyTorch's Adam while optimizing for tile-level sparsity [S1, unofficial]. It solves the problem of inconsistent state updates in sparse rendering pipelines where some tiles receive zero gradients, by replaying optimization steps for skipped tiles to ensure exponential moving averages decay correctly [S1, unofficial]. It sets itself apart from standard Adam implementations by explicitly handling the temporal gap between optimization steps for untiled regions, ensuring that the state evolution matches what would occur if every parameter were updated at every step, even with zero gradients [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `param`: Current parameter value (tensor).
    *   `grad`: Current gradient for the parameter (tensor).
    *   `exp_avg`: First moment estimate (tensor).
    *   `exp_avg_sq`: Second moment estimate (tensor).
    *   `global_step`: Current global optimization step index (integer).
    *   `lr_history`: Array of learning rates indexed by step (tensor/array).
    *   `b1`: First moment decay rate (float).
    *   `b2`: Second moment decay rate (float).
    *   `eps`: Epsilon for numerical stability (float).
    *   `tiles_touched`: Boolean array indicating which tiles were rendered in the current step (array).
    *   `last_seen_step`: Integer array tracking the last optimization step performed for each tile (array).
    *   `N`: Total number of tiles (integer).
    *   `M`: Number of parameters per tile (integer).
*   **Outputs:**
    *   Updated `param`, `exp_avg`, `exp_avg_sq` in global memory.
    *   Updated `last_seen_step` for the tile if this thread is the leader.

**Data Structures:**
*   `tiles_touched`: A boolean array of size $N$, where $N$ is the number of tiles. `tiles_touched[i]` is true if tile $i$ contributed to the loss in the current step.
*   `last_seen_step`: An integer array of size $N$. `last_seen_step[i]` stores the global step index of the last time tile $i$ was updated.
*   `lr_history`: An array or tensor of size equal to the maximum number of optimization steps, storing the learning rate used at each step.

**Algorithm Steps:**
1.  **Thread Indexing:** Each CUDA thread is assigned a unique parameter index $p\_idx$. Calculate the global tile index $g\_idx = p\_idx / M$.
2.  **Sparsity Check:** If $g\_idx \ge N$ (out of bounds) or `tiles_touched[g_idx]` is false, the thread returns immediately. No update is performed for this parameter in this step [S1, unofficial].
3.  **State Loading:** Load the current values of `param`, `exp_avg`, `exp_avg_sq`, and `param_grad` into registers. Retrieve `prev_seen_step = last_seen_step[g_idx]` [S1, unofficial].
4.  **Zero-Gradient Replay:** If there is a gap between the previous update and the current step (i.e., $prev\_seen\_step + 1 \le global\_step - 1$), the optimizer must replay the Adam updates for all intermediate steps using zero gradients. This ensures the exponential moving averages decay correctly as if updates had occurred [S1, unofficial].
    *   For each step $s$ from $prev\_seen\_step + 1$ to $global\_step - 1$:
        *   Compute bias correction factors for step $s$:
            $$bias\_correction1 = 1 - b1^s$$
            $$bias\_correction2 = 1 - b2^s$$
        *   Compute step size:
            $$step\_size = lr\_history[s - 1] / bias\_correction1$$
        *   Compute denominator (with zero gradient, $exp\_avg$ remains unchanged from previous step, but $exp\_avg\_sq$ decays):
            $$denom = \sqrt{exp\_avg\_sq} / \sqrt{bias\_correction2} + \epsilon$$
        *   Update delta (with zero gradient, the numerator is 0, so delta is 0, but moving averages still decay):
            $$delta = -step\_size \cdot exp\_avg / denom$$
        *   Update parameters and moving averages:
            $$param = param + delta$$
            $$exp\_avg = b1 \cdot exp\_avg + (1 - b1) \cdot 0$$
            $$exp\_avg\_sq = b2 \cdot exp\_avg\_sq + (1 - b2) \cdot 0^2$$
    *   Note: Since the gradient is zero, $exp\_avg$ and $exp\_avg\_sq$ simply decay. The parameter value does not change during replay, but the state variables evolve [S1, unofficial].
5.  **Current Step Update:** Compute the Adam update for the current `global_step` using the actual `grad`.
    *   Update exponential moving averages:
        $$exp\_avg = b1 \cdot exp\_avg + (1 - b1) \cdot grad$$
        $$exp\_avg\_sq = b2 \cdot exp\_avg\_sq + (1 - b2) \cdot grad^2$$
    *   Compute bias correction factors for `global_step`:
        $$bias\_correction1 = 1 - b1^{global\_step}$$
        $$bias\_correction2 = 1 - b2^{global\_step}$$
    *   Compute step size:
        $$step\_size = lr\_history[global\_step - 1] / bias\_correction1$$
    *   Compute denominator:
        $$denom = \sqrt{exp\_avg\_sq} / \sqrt{bias\_correction2} + \epsilon$$
    *   Compute update delta:
        $$delta = -step\_size \cdot exp\_avg / denom$$
    *   Update parameter:
        $$param = param + delta$$
6.  **State Writeback:** Write the updated `param`, `exp_avg`, and `exp_avg_sq` back to global memory.
7.  **Leader Update:** If the thread index $p\_idx$ is a multiple of $M$ (indicating it is the first thread of the tile group), update `last_seen_step[g_idx] = global_step` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adam_update_torch_cuda(
    param: float,
    grad: float,
    exp_avg: float,
    exp_avg_sq: float,
    global_step: int,
    lr_history: list[float],
    b1: float,
    b2: float,
    eps: float,
    tiles_touched: list[bool],
    last_seen_step: list[int],
    g_idx: int,
    p_idx: int,
    M: int,
    N: int
) -> tuple[float, float, float, list[int]]:
    """
    Performs one Adam update step with zero-gradient replay and tile-level sparsity.
    
    Args:
        param: Current parameter value.
        grad: Current gradient.
        exp_avg: First moment estimate.
        exp_avg_sq: Second moment estimate.
        global_step: Current global step index (1-based).
        lr_history: List of learning rates, indexed by step-1.
        b1: First moment decay rate.
        b2: Second moment decay rate.
        eps: Epsilon for numerical stability.
        tiles_touched: Boolean array of touched tiles.
        last_seen_step: Array of last seen steps per tile.
        g_idx: Global tile index.
        p_idx: Global parameter index.
        M: Parameters per tile.
        N: Total number of tiles.
        
    Returns:
        Updated (param, exp_avg, exp_avg_sq, last_seen_step).
    """
    # Step 1 & 2: Sparsity Check
    if g_idx >= N or not tiles_touched[g_idx]:
        return param, exp_avg, exp_avg_sq, last_seen_step

    # Step 3: Load State
    prev_seen_step = last_seen_step[g_idx]

    # Step 4: Zero-Gradient Replay
    # Replay steps from prev_seen_step + 1 to global_step - 1
    for s in range(prev_seen_step + 1, global_step):
        # Bias correction for step s
        bias_correction1 = 1.0 - b1 ** s
        bias_correction2 = 1.0 - b2 ** s
        
        # Step size
        step_size = lr_history[s - 1] / bias_correction1
        
        # Denominator (gradient is 0 for replay)
        denom = (exp_avg_sq ** 0.5) / (bias_correction2 ** 0.5) + eps
        
        # Delta is 0 because exp_avg is not updated with gradient in replay
        # But exp_avg and exp_avg_sq still decay
        delta = -step_size * exp_avg / denom
        
        # Update param (no change if delta is 0, but for correctness)
        param = param + delta
        
        # Decay moving averages with zero gradient
        exp_avg = b1 * exp_avg + (1.0 - b1) * 0.0
        exp_avg_sq = b2 * exp_avg_sq + (1.0 - b2) * 0.0 ** 2

    # Step 5: Current Step Update
    # Update moving averages with current gradient
    exp_avg = b1 * exp_avg + (1.0 - b1) * grad
    exp_avg_sq = b2 * exp_avg_sq + (1.0 - b2) * grad ** 2
    
    # Bias correction for current global_step
    bias_correction1 = 1.0 - b1 ** global_step
    bias_correction2 = 1.0 - b2 ** global_step
    
    # Step size
    step_size = lr_history[global_step - 1] / bias_correction1
    
    # Denominator
    denom = (exp_avg_sq ** 0.5) / (bias_correction2 ** 0.5) + eps
    
    # Update delta
    delta = -step_size * exp_avg / denom
    
    # Update parameter
    param = param + delta

    # Step 6: Writeback
    # (In CUDA, these are written to global memory)
    
    # Step 7: Leader Update
    if p_idx % M == 0:
        last_seen_step[g_idx] = global_step

    return param, exp_avg, exp_avg_sq, last_seen_step
```

**Parameters:**
*   `b1`: First moment decay rate, typically 0.9 [S1, unofficial].
*   `b2`: Second moment decay rate, typically 0.999 [S1, unofficial].
*   `eps`: Epsilon for numerical stability, typically 1e-8 [S1, unofficial].
*   `lr_history`: Array of learning rates indexed by step, used to support dynamic learning rate schedules while maintaining Torch compatibility [S1, unofficial].
*   `tiles_touched`: Boolean array indicating which tiles were rendered in the current step, used to skip updates for untouched tiles [S1, unofficial].
*   `last_seen_step`: Integer array tracking the last optimization step performed for each tile, used to determine if zero-gradient replay is needed [S1, unofficial].

**Pitfalls:**
*   If a tile was not touched in previous steps, the moving averages must be decayed for each skipped step to maintain consistency with PyTorch's Adam behavior, which assumes updates occur at every step even if gradients are zero [S1, unofficial]. Failure to do so results in incorrect state evolution and potentially poor convergence [S1, unofficial].
*   The replay loop can be computationally expensive if large gaps occur between updates for a tile. However, in typical rasterization pipelines, tiles are often touched frequently, minimizing this cost [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per parameter per step, plus $O(k)$ for zero-gradient replay where $k$ is the number of skipped steps. In the worst case, $k$ can be large, but typically $k$ is small due to frequent rasterization of most tiles.
*   **Memory Complexity:** $O(N \cdot M)$ for storing parameter states and $O(N)$ for `tiles_touched` and `last_seen_step`.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with the number of tiles and parameters due to the tile-level sparsity optimization. Untouched tiles are skipped entirely, reducing unnecessary memory bandwidth and computation.
*   **Strengths:** Maintains strict compatibility with PyTorch's Adam, ensuring consistent behavior across different frameworks. Efficiently handles sparse gradients by skipping updates for untouched tiles while correctly maintaining state for those tiles.
*   **Limitations:** The zero-gradient replay can add overhead if tiles are frequently skipped for many steps. Requires careful synchronization for the `last_seen_step` update if multiple threads update the same tile's state concurrently (though the leader thread approach mitigates this).
*   **When to use:** Use when training neural radiance fields with tile-based rasterization where sparsity is high, and strict compatibility with PyTorch's Adam is required.
*   **When not to use:** Avoid if the overhead of zero-gradient replay becomes significant due to extreme sparsity or if framework compatibility is not a concern.

## Background
This method is part of the optimization procedures for NeRF training loops, specifically within the context of custom Adam implementations that exploit CUDA kernel fusion and fast math operations [S1, unofficial]. It was introduced in the context of improving Gaussian Splatting training efficiency by XiaoBin2001 [S1, unofficial].

## Sources
[S1] XiaoBin2001/Improved-GS, https://github.com/XiaoBin2001/Improved-GS, fetched 2026-10-27 (unofficial)
