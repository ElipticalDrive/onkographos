# Differentiable Advection Gradients

Also called: Differentiable SL Advection Gradients

## Overview
Differentiable Advection Gradients provide the backward pass for semi-Lagrangian (SL) advection operators used in differentiable volume rendering and fluid reconstruction. They solve the problem of computing gradients of the advected field with respect to both the input scalar field (e.g., density) and the velocity field, enabling end-to-end optimization via automatic differentiation. Unlike implicit differentiation methods that solve linear systems for sphere tracing [2012-implicit-differentiation-for-sphere-tracing.md], this method relies on explicit gradient scattering and component-wise multiplication, leveraging the structure of the interpolation kernel.

## How it works
The method computes the gradients $\nabla \rho_t^A = \frac{\partial A(\rho_t, u_t)}{\partial \rho_t}$ and $\nabla u_t^A = \frac{\partial A(\rho_t, u_t)}{\partial u_t}$ required for back-propagation through the advection operator $A$. The implementation is typically provided as a custom CUDA operation (e.g., for TensorFlow) to handle the non-differentiable nature of standard grid lookups [S1].

**Inputs:**
- $\rho_t$: Input scalar field (density) on a 3D grid.
- $u_t$: Input velocity field on a 3D grid.
- $\nabla \rho_{t+1}^A$: Gradient of the loss with respect to the advected output $A(\rho_t, u_t)$.

**Outputs:**
- $\nabla \rho_t^A$: Gradient with respect to the input density.
- $\nabla u_t^A$: Gradient with respect to the input velocity.

**Data Structures:**
- Grids are stored as dense 3D arrays.
- Interpolation uses bilinear or trilinear weights $w_i$ associated with the 8 neighboring grid cells for each particle trace.

**Algorithm Steps:**

1. **Forward Trace:** For each grid cell $(i,j,k)$ in the output domain, trace the particle backward using $u_t$ to find the source position $p$ in the $\rho_t$ grid.
2. **Interpolation Weights:** Compute the interpolation weights $w_0, \dots, w_7$ based on the fractional part of $p$.
3. **Compute $\nabla u_t^A$:**
   - Compute the spatial gradient of the scalar field $\nabla \rho_t$ at the source positions. This is done by applying the same interpolation weights to the gradient of the loss with respect to the output, but specifically for the velocity term.
   - The formula is:
     $$ \nabla u_t^A = A(\nabla \rho_t, u_t) $$
     In practice, this means the gradient flowing back to the velocity is the advected spatial gradient of the density. Specifically, the gradient $\nabla \rho_{t+1}^A$ is multiplied component-wise with the advected spatial gradients of $\rho_t$ [S1].
4. **Compute $\nabla \rho_t^A$:**
   - Scatter the gradient $\nabla \rho_{t+1}^A$ back to the source grid positions.
   - For each output cell, distribute $\nabla \rho_{t+1}^A$ to the 8 neighboring source cells, scaled by the corresponding interpolation weight $w_i$.
   - Formula:
     $$ \nabla \rho_t^A = \sum_{i=0}^{7} w_i \cdot \nabla \rho_{t+1}^A $$
     The gradients are scattered to the grid positions used for interpolation in the forward step, multiplied by interpolation weights [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_sl_advection_backward(
    rho_t: Tensor, 
    u_t: Tensor, 
    grad_output: Tensor, 
    grid_size: Tuple[int, int, int]
) -> Tuple[Tensor, Tensor]:
    """
    Computes gradients for differentiable semi-Lagrangian advection.
    
    Args:
        rho_t: Input density field [D, H, W, C]
        u_t: Input velocity field [D, H, W, 3]
        grad_output: Gradient of loss w.r.t. advected output [D, H, W, C]
        grid_size: Spatial dimensions (D, H, W)
        
    Returns:
        grad_rho: Gradient w.r.t. input density
        grad_u: Gradient w.r.t. input velocity
    """
    # Initialize gradients
    grad_rho = torch.zeros_like(rho_t)
    grad_u = torch.zeros_like(u_t)
    
    # 1. Trace particles backward to find source positions
    # This step is identical to the forward pass but accumulates gradients
    source_positions, weights = trace_backward(u_t, grid_size)
    
    # 2. Compute grad_rho: Scatter grad_output back to source positions
    # For each output voxel, scatter grad_output * weight to the 8 neighbors
    for voxel in output_voxels:
        pos = source_positions[voxel]
        w = weights[voxel]
        # Scatter operation: add w * grad_output to the 8 neighboring grid cells
        scatter_add(grad_rho, pos, w * grad_output[voxel])
        
    # 3. Compute grad_u: Multiply grad_output with advected spatial gradients of rho
    # First, compute spatial gradient of rho_t (e.g., using finite differences or autograd)
    grad_rho_spatial = compute_spatial_gradient(rho_t)
    
    # The gradient w.r.t. velocity is the advection of the spatial gradient of density
    # Formula: grad_u = A(grad_rho_spatial, u_t)
    # This involves tracing grad_rho_spatial with u_t and interpolating
    advected_grad_rho = advect_field(grad_rho_spatial, u_t, grid_size)
    
    # Component-wise multiplication with the gradient of the output
    # Note: The source states "multiplied component wise with the advected spatial gradients"
    # This implies the gradient flow to u is driven by how changes in u affect the advection of grad_rho
    grad_u = advected_grad_rho * grad_output # Element-wise
    
    return grad_rho, grad_u
```

**Edge Cases and Pitfalls:**
- **Boundary Handling:** Open boundaries are used. Velocities at boundaries are unconstrained. Look-ups falling outside the volume are clamped to the boundary values. Gradients are accumulated at the boundaries accordingly [S1].
- **Precision:** The scattering operation must handle overlapping contributions from multiple output voxels tracing to the same source voxel correctly (additive accumulation).

## Subtopics

### [Physical Prior / Transport Losses](20694-physical-prior-transport-losses.md)
Uses these gradients to enforce physical fluid motion constraints between frames.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of grid cells and particles traced. The backward pass requires a second trace (for velocity gradient) and a scatter operation.
- **Memory Complexity:** Requires storing the forward pass intermediate values (source positions, weights) or recomputing them, plus gradients for $\rho$ and $u$.
- **Strengths:** Efficient implementation via CUDA kernels; compatible with standard automatic differentiation frameworks when wrapped as a custom op.
- **Limitations:** Relies on the differentiability of the interpolation kernel; open boundaries may introduce artifacts if not handled carefully.

## Background
The method was introduced in the context of global transport for fluid reconstruction with learned self-supervision [S1]. It enables the optimization of both density and velocity fields in differentiable fluid simulations by providing explicit gradients for the semi-Lagrangian advection step, which is otherwise non-differentiable due to grid lookups.

## Sources
[S1] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
