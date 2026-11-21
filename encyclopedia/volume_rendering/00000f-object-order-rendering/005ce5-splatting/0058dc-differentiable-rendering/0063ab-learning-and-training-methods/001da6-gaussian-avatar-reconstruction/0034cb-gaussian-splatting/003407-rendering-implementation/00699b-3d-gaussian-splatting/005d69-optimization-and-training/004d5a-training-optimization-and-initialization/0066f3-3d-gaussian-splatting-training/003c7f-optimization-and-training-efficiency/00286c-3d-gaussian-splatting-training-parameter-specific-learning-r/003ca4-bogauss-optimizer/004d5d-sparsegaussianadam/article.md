# SparseGaussianAdam

Also called: Sparse Adam for Gaussian Splatting, Sparse Gaussian Adam Optimizer

## Overview
SparseGaussianAdam is a variant of the Adam optimizer designed for 3D Gaussian Splatting that restricts parameter updates to only those Gaussians currently visible in the rendered image [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of biased state estimation and inefficient computation caused by sparse visibility in differentiable rendering, where most Gaussians do not contribute to the loss in any given frame [S1, unofficial][S2, unofficial][S3, unofficial]. It differs from standard Adam by explicitly compensating for the learning rate bias correction based on the ratio of active parameters to total parameters, and from its parent [BOGausS Optimizer](9298-bogauss-optimizer.md) by relying on a custom CUDA kernel for visibility-aware updates rather than unbiased stochastic gradient estimation [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Parameter tensor $P$ (shape $[N, M]$), gradient tensor $G$ (shape $[N, M]$), visibility mask $V$ (boolean tensor of length $N$, where $V_i = \text{True}$ if Gaussian $i$ is visible), base learning rate $\eta$, epsilon $\epsilon$, momentum coefficients $\beta_1, \beta_2$.
- **Output:** Updated parameter tensor $P$.

**Data Structures:**
The optimizer maintains state per parameter group. Each state entry contains:
- `step`: A scalar `float32` representing the current optimization step count [S3, unofficial].
- `exp_avg`: A tensor of the same shape as the parameter, initialized to zeros, storing the first moment estimate [S1, unofficial][S2, unofficial][S3, unofficial].
- `exp_avg_sq`: A tensor of the same shape as the parameter, initialized to zeros, storing the second moment estimate [S1, unofficial][S2, unofficial][S3, unofficial].

**Numbered Steps:**
1. **Initialization:** If the state for the parameter is empty, initialize `step` to $0.0$ (float32), `exp_avg` to a zero tensor with the same shape and memory format as the parameter, and `exp_avg_sq` to a zero tensor with the same shape and memory format [S1, unofficial][S2, unofficial][S3, unofficial].
2. **Increment Step:** Increment the step counter: $step \leftarrow step + 1$ [S1, unofficial].
3. **Retrieve State:** Retrieve `exp_avg` and `exp_avg_sq` from the optimizer state [S2, unofficial][S3, unofficial].
4. **Calculate Dimensions:** Determine $N$, the number of visible Gaussians (active parameters), and $M$, the number of elements per Gaussian, calculated as $M = \text{param.numel()} // N$ [S1, unofficial][S2, unofficial][S3, unofficial]. Note: $N$ is derived from the number of `True` values in the visibility mask [S1, unofficial].
5. **Compute Effective Learning Rate:** Calculate the bias-corrected effective learning rate $\eta_{eff}$:
   $$ \eta_{eff} = \eta \frac{\sqrt{1 - \beta_2^{step}}}{1 - \beta_1^{step}} $$
   where $\eta$ is the base learning rate, $\beta_1 = 0.9$, and $\beta_2 = 0.999$ [S1, unofficial].
6. **Apply Update:** Invoke the custom CUDA kernel `_C.adamUpdate` with the following arguments:
   - `param`: The parameter tensor $P$.
   - `param.grad`: The gradient tensor $G$.
   - `exp_avg`: The first moment estimate.
   - `exp_avg_sq`: The second moment estimate.
   - `visibility_mask`: The boolean mask $V$.
   - `lr`: The effective learning rate $\eta_{eff}$ [S1, unofficial].
   - `beta1`: $0.9$ [S2, unofficial][S3, unofficial].
   - `beta2`: $0.999$ [S2, unofficial][S3, unofficial].
   - `eps`: The epsilon value $\epsilon$ [S1, unofficial][S3, unofficial].
   - `N`: The number of visible Gaussians [S1, unofficial][S2, unofficial][S3, unofficial].
   - `M`: The number of elements per Gaussian [S2, unofficial][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_gaussian_adam_step(param, grad, visibility_mask, lr, eps, beta1, beta2):
    """
    Performs one step of the SparseGaussianAdam optimizer.
    
    Args:
        param (torch.Tensor): The parameter tensor of shape (N_total, M).
        grad (torch.Tensor): The gradient tensor of shape (N_total, M).
        visibility_mask (torch.Tensor): Boolean tensor of shape (N_total,) indicating visible Gaussians.
        lr (float): Base learning rate.
        eps (float): Epsilon for numerical stability.
        beta1 (float): Exponential decay rate for the first moment estimates.
        beta2 (float): Exponential decay rate for the second moment estimates.
        
    Returns:
        torch.Tensor: Updated parameter tensor.
    """
    # 1. Initialize state if empty [S3, unofficial]
    if not hasattr(sparse_gaussian_adam_step, 'state'):
        sparse_gaussian_adam_step.state = {}
    
    state = sparse_gaussian_adam_step.state.get(id(param))
    if state is None:
        state = {
            'step': 0.0,
            'exp_avg': torch.zeros_like(param, memory_format=torch.preserve_format),
            'exp_avg_sq': torch.zeros_like(param, memory_format=torch.preserve_format)
        }
        sparse_gaussian_adam_step.state[id(param)] = state
    
    # 2. Increment step [S1, unofficial]
    state['step'] += 1
    step = state['step']
    
    # 3. Retrieve state [S2, unofficial][S3, unofficial]
    exp_avg = state['exp_avg']
    exp_avg_sq = state['exp_avg_sq']
    
    # 4. Calculate dimensions [S1, unofficial][S2, unofficial][S3, unofficial]
    # N is the number of visible Gaussians
    N = visibility_mask.sum().item()
    # M is the number of elements per Gaussian
    M = param.numel() // N
    
    # 5. Compute effective learning rate [S1, unofficial]
    # Bias correction for Adam
    bias_correction1 = 1 - beta1 ** step
    bias_correction2 = 1 - beta2 ** step
    lr_eff = lr * (bias_correction2 ** 0.5) / bias_correction1
    
    # 6. Apply update via CUDA kernel [S1, unofficial][S2, unofficial][S3, unofficial]
    # The kernel handles the sparse update logic internally using the visibility mask
    # _C.adamUpdate(param, grad, exp_avg, exp_avg_sq, visibility_mask, lr_eff, beta1, beta2, eps, N, M)
    
    # Since we cannot call the actual C++ kernel in pure Python pseudocode,
    # we describe the logical update for visible elements only.
    # For elements where visibility_mask is False, exp_avg and exp_avg_sq are NOT updated.
    # For elements where visibility_mask is True, standard Adam update is applied.
    
    if N > 0:
        # Select visible parameters and gradients
        visible_param = param[visibility_mask]
        visible_grad = grad[visibility_mask]
        visible_exp_avg = exp_avg[visibility_mask]
        visible_exp_avg_sq = exp_avg_sq[visibility_mask]
        
        # Standard Adam update for visible parameters [G]
        # exp_avg = beta1 * exp_avg + (1 - beta1) * grad
        visible_exp_avg.mul_(beta1).add_(visible_grad, alpha=1 - beta1)
        
        # exp_avg_sq = beta2 * exp_avg_sq + (1 - beta2) * grad^2
        visible_exp_avg_sq.mul_(beta2).addcmul_(visible_grad, visible_grad, value=1 - beta2)
        
        # Bias-corrected moment estimates
        denom = (visible_exp_avg_sq.sqrt() / (bias_correction2 ** 0.5)).add_(eps / bias_correction1)
        
        # Update parameters
        step_size = lr_eff / denom
        visible_param.addcdiv_(visible_exp_avg, denom, value=-step_size)
        
        # Write back to full tensors
        param[visibility_mask] = visible_param
        exp_avg[visibility_mask] = visible_exp_avg
        exp_avg_sq[visibility_mask] = visible_exp_avg_sq
    else:
        # No visible Gaussians, skip update [S1, unofficial]
        pass
        
    return param
```

**Parameters:**
- `lr`: Base learning rate. Typical values depend on the specific Gaussian attribute (e.g., position, rotation) [S1, unofficial][S3, unofficial].
- `eps`: Epsilon for numerical stability. Passed to the update function [S1, unofficial][S3, unofficial].
- `N`: Number of visible Gaussians. Used to scale the learning rate compensation and passed to the C++ update function [S1, unofficial][S3, unofficial].
- `M`: Number of parameter elements per Gaussian (e.g., 3 for means, 4 for rotations) [S2, unofficial][S3, unofficial].
- `beta1`: First moment decay rate, typically $0.9$ [S2, unofficial][S3, unofficial].
- `beta2`: Second moment decay rate, typically $0.999$ [S2, unofficial][S3, unofficial].

**Edge Cases and Pitfalls:**
- If the parameter gradient is `None` or has zero elements, the step is skipped to avoid errors in state initialization or update [S1, unofficial].
- Precision issues may arise if $N$ is very small, leading to unstable bias correction. The sources do not specify a minimum $N$ threshold [G].
- The calculation of $M$ assumes that the parameter tensor is evenly divisible by $N$. If $N=0$, division by zero occurs. The pseudocode handles this by skipping the update [G].

## Subtopics

### [GaussianAdam Optimizer](13955-gaussianadam-optimizer.md)
GaussianAdam is a custom variant of the Adam optimizer designed specifically for 3D Gaussian Splatting that tracks per-Gaussian visibility via a last-seen step counter and maintains a learning rate history buffer for adaptive scheduling. It differs from standard Adam by explicitly managing per-Gaussian visibility and a dedicated learning rate history buffer, enabling step-dependent learning rate scheduling per Gaussian rather than a global schedule.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$ per step, where $N$ is the number of visible Gaussians and $M$ is the number of elements per Gaussian. This is significantly faster than standard Adam ($O(N_{total} \cdot M)$) when visibility is sparse [S2, unofficial][S3, unofficial].
- **Memory Complexity:** $O(N_{total} \cdot M)$ for storing `exp_avg` and `exp_avg_sq` for all Gaussians, even if only $N$ are visible. The sources do not specify if sparse tensors are used for the state [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of visible Gaussians. Performance improves as visibility becomes sparser [S2, unofficial][S3, unofficial].
- **Strengths:** Reduces computation by a factor of $N$ (number of Gaussians) by only updating visible Gaussians [S2, unofficial]. Compensates for learning rate bias correction in sparse updates [S1, unofficial].
- **Limitations:** Requires a custom CUDA kernel (`_C.adamUpdate`) for efficient implementation [S1, unofficial][S2, unofficial][S3, unofficial]. Not suitable for scenarios where all Gaussians are visible, as the overhead of the custom kernel may outweigh the benefits [G].
- **When to use:** When training 3D Gaussian Splatting models with sparse visibility, such as in large scenes or early training stages [S1, unofficial][S2, unofficial][S3, unofficial].
- **When not to use:** When visibility is dense, or when a standard Adam implementation is preferred for simplicity [G].

## Background
SparseGaussianAdam originates from the 3D Gaussian Splatting literature, specifically in implementations that optimize for training efficiency and memory usage [S1, unofficial][S2, unofficial][S3, unofficial]. It is a variant of the Adam optimizer tailored to the sparse nature of Gaussian visibility in differentiable rendering [S1, unofficial][S2, unofficial][S3, unofficial]. The method is used in various Gaussian Splatting training pipelines to accelerate convergence and reduce computational overhead [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
[S2] fastgs/FastGS, https://github.com/fastgs/FastGS, fetched 2026-10-27 (unofficial)
[S3] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
