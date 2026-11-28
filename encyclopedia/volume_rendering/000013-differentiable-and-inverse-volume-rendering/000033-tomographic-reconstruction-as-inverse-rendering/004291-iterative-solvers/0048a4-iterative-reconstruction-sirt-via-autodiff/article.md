# Iterative Reconstruction (SIRT via Autodiff)
Also called: Gradient Descent Reconstruction

## Overview
This method performs tomographic reconstruction by treating the volume estimation as a differentiable optimization problem, using automatic differentiation to minimize the mean squared error between simulated and measured projections [S1, unofficial]. It differs from standard [SIRT (Simultaneous Iterative Reconstruction Technique)](3101-sirt-simultaneous-iterative-reconstruction-technique.md) by replacing explicit algebraic matrix operations with gradient descent via an autodiff engine, allowing for easier integration of custom physics constraints or differentiable regularizers [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Measured projections as a tensor of shape $(N, H, W)$ where $N$ is the number of angles and $H, W$ are detector dimensions [S1, unofficial]. Projection geometry parameters defining source-detector distances and angles (e.g., $DSO$, $DSD$, detector pixel size, and a list of $N$ angles) [S1, unofficial]. Volume dimensions $D \times D \times D$ [S1, unofficial].
- **Outputs:** Reconstructed volume as a tensor of shape $(D, D, D)$ representing the estimated attenuation coefficients [S1, unofficial].

**Data Structures:**
- `volume`: A PyTorch tensor of shape $(D, D, D)$ initialized with zeros or a prior estimate. It must have `requires_grad=True` to track gradients [S1, unofficial].
- `optimizer`: An Adam optimizer instance initialized with the `volume` parameters [S1, unofficial].

**Numbered Steps:**
1. **Initialization**: Create a tensor `volume` of shape $(D, D, D)$ with `requires_grad=True`. Initialize an Adam optimizer with `volume` as the parameter and `lr=1e-2` [S1, unofficial].
2. **Iteration Loop**: For $k = 1$ to `iters`:
   a. **Zero Gradients**: Call `optimizer.zero_grad()` to clear previous gradients [S1, unofficial].
   b. **Forward Projection**: Compute simulated projections $P_{sim}$ from the current `volume` using a differentiable forward projector $F(\cdot)$ defined by the geometry. $P_{sim} = F(\text{volume})$ [S1, unofficial].
   c. **Loss Computation**: Calculate the Mean Squared Error (MSE) between the simulated projections $P_{sim}$ and the measured projections $P_{meas}$ [S1, unofficial].
   d. **Backpropagation**: Compute gradients of the loss with respect to the volume: $\nabla_{\text{volume}} L = \frac{\partial L}{\partial \text{volume}}$. This is handled automatically by the autodiff engine [S1, unofficial].
   e. **Optimization Step**: Update the volume parameters using the optimizer. Adam applies adaptive learning rates internally rather than a fixed step size [S1, unofficial].

**Formulas:**
$$ L = \frac{1}{NHW} \sum_{i=1}^{N} \sum_{j=1}^{H} \sum_{l=1}^{W} (P_{sim}[i,j,l] - P_{meas}[i,j,l])^2 $$
where $L$ is the loss, $N$ is the number of angles, $H$ and $W$ are detector dimensions, and $P_{sim}$ and $P_{meas}$ are the simulated and measured projection tensors respectively [S1, unofficial].

**Parameters:**
- `lr`: Learning rate, typically $1 \times 10^{-2}$ [S1, unofficial].
- `iters`: Number of optimization steps, default 50 [S1, unofficial].
- `geometry`: Object containing $DSO$ (Source-to-Object), $DSD$ (Source-to-Detector), detector resolution, and angles [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Input**: If projections are all zeros or constant, the gradient may vanish, leading to no update. Initialize with a small random noise or a prior estimate (e.g., FDK) to break symmetry [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def iterative_reconstruction_autodiff(projections, geometry, vol_shape, iters=50, lr=1e-2):
    """
    Reconstruct volume using SIRT-like gradient descent via autodiff.
    
    Args:
        projections: Tensor [N, H, W] of measured projections.
        geometry: Object with forward_project method.
        vol_shape: Tuple (D, D, D) for volume dimensions.
        iters: Number of optimization steps.
        lr: Learning rate for Adam optimizer.
        
    Returns:
        Reconstructed volume tensor [D, D, D].
    """
    # Initialize volume tensor with gradients enabled
    volume = torch.zeros(vol_shape, requires_grad=True, device=projections.device)
    
    # Initialize Adam optimizer
    optimizer = torch.optim.Adam([volume], lr=lr)
    
    for step in range(iters):
        # Zero gradients from previous step
        optimizer.zero_grad()
        
        # Forward projection: simulate projections from current volume estimate
        simulated_projections = geometry.forward_project(volume)
        
        # Compute loss: Mean Squared Error between simulated and measured projections
        loss = F.mse_loss(simulated_projections, projections)
        
        # Backpropagate to compute gradients on the volume
        loss.backward()
        
        # Update volume parameters
        optimizer.step()
        
    return volume.detach()
```

## Subtopics

### [ACID (Adversarial Calibration Iterative Deep learning)](16417-acid-adversarial-calibration-iterative-deep-learning.md)
ACID is an iterative reconstruction framework that stabilizes an unstable deep reconstruction neural network by combining it with a Compressed Sensing (CS)-based sparsity-promoting module. It minimizes an objective function balancing neural network output consistency, data fidelity, residual error energy, and sparsity. ACID searches for a solution good in three aspects: image sparsity, big-data-driven prior, and iterative calibration to eliminate unexplained residual data. It differs from standard sparsity-minimization reconstruction by incorporating a data-driven deep prior ($\Phi$) to fill gaps where sparsity alone may not yield a unique solution, especially when the measurement matrix does not satisfy the Restricted Isometry Property (RIP). Unlike traditional iterative reconstruction which searches for a feasible solution within the space of sparse solutions, ACID simultaneously optimizes for sparsity, data-driven priors, and residual calibration.

### [Implicit Neural Representation (INR) for Tomography](19578-implicit-neural-representation-inr-for-tomography.md)
A method that uses an implicit neural network to represent the 3D volume density field, optimizing the network parameters to fit the 2D projection images while jointly optimizing the 3D orientation (pose) of each projection.

## Operating characteristics
- **Time Complexity**: [G] Not given in the sources.
- **Memory Complexity**: [G] Not given in the sources.
- **Performance**: [G] Not given in the sources.
- **Strengths**: Easy to integrate custom constraints (e.g., non-negativity, sparsity) by modifying the loss function. No need to explicitly derive or implement the backprojection matrix [G].
- **Limitations**: Convergence depends heavily on the learning rate and the condition number of the problem. May get stuck in local minima if the initial guess is poor [G].
- **When to Use**: When custom physics constraints are needed, or when integrating with deep learning components (e.g., neural networks as priors) [G].
- **When Not to Use**: For standard CT reconstruction with no special constraints, where FDK or algebraic methods are faster and sufficient [G].

## Background
This approach leverages the automatic differentiation capabilities of deep learning frameworks (like PyTorch) to solve inverse problems traditionally addressed by algebraic reconstruction techniques [S1, unofficial]. The specific implementation described here uses a cone-beam forward projector and Adam optimizer to minimize projection error [S1, unofficial].

## Sources
[S1] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
