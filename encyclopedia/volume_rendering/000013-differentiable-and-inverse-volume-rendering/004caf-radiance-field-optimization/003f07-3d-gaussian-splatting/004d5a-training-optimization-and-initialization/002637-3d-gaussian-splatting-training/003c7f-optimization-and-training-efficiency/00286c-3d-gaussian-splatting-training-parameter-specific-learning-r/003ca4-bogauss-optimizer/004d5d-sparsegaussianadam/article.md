# SparseGaussianAdam
Also called: Sparse Adam for Gaussian Splatting

## Overview
SparseGaussianAdam is a sparse variant of the Adam optimizer designed specifically for 3D Gaussian Splatting [S1, unofficial]. It addresses the computational and memory inefficiencies of standard Adam by updating parameters only for visible Gaussians—those contributing to the rendered image—thereby saving memory and computation [S1, unofficial]. It differs from standard Adam by explicitly managing per-Gaussian visibility and leveraging a custom CUDA kernel to apply updates selectively, rather than processing the entire parameter tensor globally [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Parameter tensor $P$, gradient tensor $G$, visibility mask $V$ (boolean or integer indicating which Gaussians are visible), learning rate $lr$, epsilon $\epsilon$, number of Gaussians $N$, and hyperparameters $\beta_1=0.9$, $\beta_2=0.999$.
- **Outputs:** Updated parameter tensor $P$.

**Data Structures:**
The optimizer maintains state variables for each parameter group:
- `step`: A scalar float32 representing the current optimization step count.
- `exp_avg`: A zero tensor with the same shape as $P$, storing the first moment estimate.
- `exp_avg_sq`: A zero tensor with the same shape as $P$, storing the second moment estimate.
- Memory format is preserved using `torch.preserve_format` [S1, unofficial].

**Parameters:**
- $lr$: Learning rate, passed from the optimizer group configuration [S1, unofficial].
- $\epsilon$: Epsilon value for numerical stability, passed from the optimizer group configuration [S1, unofficial].
- $N$: The number of Gaussians, used to normalize the state update scope [S1, unofficial].
- $M$: The number of elements per Gaussian, calculated as $M = \text{param.numel()} // N$ [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** If the optimizer state for a parameter is empty (lazy initialization), initialize `step` to $0.0$ (float32), and `exp_avg` and `exp_avg_sq` as zero tensors matching the parameter's shape [S1, unofficial].
2. **Validation:** Assert that each parameter group contains exactly one tensor. If the parameter has no gradient, skip the update for this parameter [S1, unofficial].
3. **Calculation:** Calculate $M$ as the total number of elements in the parameter tensor divided by $N$ (number of Gaussians) [S1, unofficial].
4. **Update:** Invoke the custom C++/CUDA function `_C.adamUpdate` with the following arguments:
   - `param`: The parameter tensor $P$.
   - `param.grad`: The gradient tensor $G$.
   - `exp_avg`: The first moment estimate.
   - `exp_avg_sq`: The second moment estimate.
   - `visibility_mask`: The mask $V$ indicating visible Gaussians.
   - `lr`: The learning rate.
   - `beta1`: $0.9$.
   - `beta2`: $0.999$.
   - `eps`: The epsilon value $\epsilon$.
   - `N`: The number of Gaussians.
   - `M`: The elements per Gaussian [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sparse_gaussian_adam_step(param, grad, state, visibility_mask, lr, eps, N):
    """
    Performs one optimization step for SparseGaussianAdam.
    
    Args:
        param (torch.Tensor): The parameter tensor.
        grad (torch.Tensor): The gradient tensor.
        state (dict): The optimizer state containing 'step', 'exp_avg', 'exp_avg_sq'.
        visibility_mask (torch.Tensor): Boolean mask of visible Gaussians.
        lr (float): Learning rate.
        eps (float): Epsilon for numerical stability.
        N (int): Number of Gaussians.
        
    Returns:
        None (updates param and state in-place).
    """
    # Lazy initialization
    if 'step' not in state:
        state['step'] = torch.tensor(0.0, dtype=torch.float32, device=param.device)
        state['exp_avg'] = torch.zeros_like(param, memory_format=torch.preserve_format)
        state['exp_avg_sq'] = torch.zeros_like(param, memory_format=torch.preserve_format)
    
    step = state['step']
    exp_avg = state['exp_avg']
    exp_avg_sq = state['exp_avg_sq']
    
    # Calculate M: elements per Gaussian
    M = param.numel() // N
    
    # Increment step
    step.add_(1)
    
    # Call the custom CUDA kernel
    # Note: This is a placeholder for the actual C++/CUDA implementation _C.adamUpdate
    # The kernel handles the sparse update logic internally based on visibility_mask
    _C.adamUpdate(
        param,
        grad,
        exp_avg,
        exp_avg_sq,
        visibility_mask,
        lr,
        0.9,  # beta1
        0.999, # beta2
        eps,
        N,
        M
    )
    
    # Update state step counter
    state['step'] = step

# Example usage within an optimizer loop
def optimize_gaussians(gaussians, images, cameras, pipeline, opt):
    for iteration in range(opt.iterations):
        # Forward pass and loss computation
        # ...
        
        # Backward pass
        loss.backward()
        
        # Get visibility mask from the rendering process
        # This mask indicates which Gaussians contributed to the rendered image
        visibility_mask = gaussians.get_visibility_mask()
        
        # Update parameters using SparseGaussianAdam
        for param_group in gaussians.optimizer.param_groups:
            param = param_group['params'][0]
            grad = param.grad
            
            if grad is None:
                continue
                
            sparse_gaussian_adam_step(
                param=param,
                grad=grad,
                state=gaussians.optimizer.state[param],
                visibility_mask=visibility_mask,
                lr=param_group['lr'],
                eps=param_group['eps'],
                N=gaussians._xyz.shape[0] # Number of Gaussians
            )
            
        # Zero gradients
        gaussians.optimizer.zero_grad()
```

**Edge Cases and Pitfalls:**
- **Precision:** The use of `float32` for the `step` counter is specified [S1, unofficial].
- **Boundaries:** The calculation of $M$ assumes that the total number of elements in the parameter tensor is evenly divisible by $N$. If not, the behavior is undefined in the sources [S1, unofficial].
- **Degenerate Input:** If a parameter has no gradient, the update is skipped [S1, unofficial].

## Subtopics

### [GaussianAdam Optimizer](13955-gaussianadam-optimizer.md)
GaussianAdam is a custom variant of the Adam optimizer designed specifically for 3D Gaussian Splatting that tracks per-Gaussian visibility via a last-seen step counter and maintains a learning rate history buffer for adaptive scheduling. It differs from standard Adam by explicitly managing per-Gaussian visibility and a dedicated learning rate history buffer, enabling step-dependent learning rate scheduling per Gaussian rather than a global schedule.

## Operating characteristics
**Time and Memory Complexity:**
- **Memory:** By updating only visible Gaussians, SparseGaussianAdam reduces memory usage compared to standard Adam, which updates all parameters regardless of visibility [S1, unofficial].
- **Computation:** The custom CUDA kernel `_C.adamUpdate` is designed to efficiently handle sparse updates, reducing computation time for scenes with many invisible Gaussians [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Saves memory and computation by focusing updates on visible Gaussians [S1, unofficial].
- **Limitations:** Relies on a custom C++/CUDA implementation, which may limit portability and ease of implementation compared to pure PyTorch solutions [S1, unofficial].

**When to Use:**
- Use when optimizing 3D Gaussian Splatting models where many Gaussians are not visible in each training step, aiming to reduce computational overhead [S1, unofficial].

**When Not to Use:**
- Not given in the sources.

## Background
SparseGaussianAdam is described as a sparse variant of the Adam optimizer designed for Gaussian Splatting [S1, unofficial]. It is part of the optimization and training efficiency strategies for 3D Gaussian Splatting, building on the principles of adaptive learning rates and sparse updates [S1, unofficial].

## Sources
[S1] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
