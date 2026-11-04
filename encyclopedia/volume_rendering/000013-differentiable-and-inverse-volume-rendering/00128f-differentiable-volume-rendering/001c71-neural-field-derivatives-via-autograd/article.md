# Neural Field Derivatives via Autograd

Also called: Differential Operators (Gradient, Divergence, Laplace), Gradient, Laplace via Autograd, autograd operators, neural field derivatives

## Overview
This method computes spatial differential operators (gradient, divergence, Laplacian) of neural field outputs with respect to their spatial inputs using automatic differentiation (autograd) [S1, unofficial]. It enables supervision on gradient targets in differentiable volume rendering by providing exact derivatives of the implicit function, avoiding the need for finite-difference approximations [S1, unofficial]. It differs from [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) by focusing on the internal field derivatives rather than the rendering integral, and from [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md) by relying on framework-level autodiff rather than analytical spline derivatives.

## How it works
**Inputs:**
- `x`: Input spatial coordinates, shape `[... , D]` where $D$ is the spatial dimension (e.g., 3 for 3D).
- `model`: A differentiable neural network (e.g., MLP) mapping `x` to output `y`.
- `y`: Model output, shape `[... , C]` where $C$ is the number of output channels (e.g., 1 for scalar SDF, 3 for RGB).

**Outputs:**
- Gradient, Divergence, or Laplacian of `y` with respect to `x`.

**Data Structures:**
- Tensors `x`, `y` with standard floating-point precision (e.g., `float32`).
- Computation graph maintained by the autograd engine.

**Parameters:**
- `create_graph=True`: Boolean flag passed to the gradient computation function. When `True`, the gradient computation is included in the computation graph, allowing higher-order derivatives (e.g., Laplacian as divergence of gradient) to be computed via further backpropagation [S1, unofficial].

**Steps:**

1. **Gradient Computation:**
   Compute the gradient of the output `y` with respect to the input `x`.
   - Call `torch.autograd.grad` (or equivalent) with `grad_outputs=torch.ones_like(y)` and `create_graph=True`.
   - The result is a tensor of shape `[... , D]` representing $\nabla y$.

2. **Divergence Computation:**
   Compute the divergence of a vector field `y` (where $C=D$) with respect to `x`.
   - Initialize `div = 0`.
   - For each component $i$ from $0$ to $D-1$:
     - Compute the gradient of `y[..., i]` with respect to `x`.
     - Extract the $i$-th component of this gradient (i.e., $\frac{\partial y_i}{\partial x_i}$).
     - Add this component to `div`.
   - The result is a scalar field (shape `[... ]`) representing $\nabla \cdot y$.

3. **Laplacian Computation:**
   Compute the Laplacian of a scalar field `y` (where $C=1$) with respect to `x`.
   - Compute the gradient of `y` with respect to `x` (as in Step 1).
   - Compute the divergence of the resulting gradient tensor (as in Step 2).
   - The result is a scalar field representing $\Delta y = \nabla \cdot (\nabla y)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_gradient(model, x, create_graph=True):
    """Compute gradient of model(x) with respect to x."""
    x.requires_grad_(True)
    y = model(x)
    # grad_outputs=torch.ones_like(y) ensures summing over output channels if needed
    # create_graph=True allows higher-order derivatives
    grad = torch.autograd.grad(
        outputs=y,
        inputs=x,
        grad_outputs=torch.ones_like(y),
        create_graph=create_graph,
        retain_graph=create_graph
    )[0]
    return grad

def compute_divergence(model, x, create_graph=True):
    """Compute divergence of model(x) with respect to x. Assumes model(x) has shape [..., D]."""
    x.requires_grad_(True)
    y = model(x)
    D = y.shape[-1]
    div = torch.zeros_like(y[..., 0])
    for i in range(D):
        # Compute gradient of the i-th component of y w.r.t. x
        grad_i = torch.autograd.grad(
            outputs=y[..., i:i+1],
            inputs=x,
            grad_outputs=torch.ones_like(y[..., i:i+1]),
            create_graph=create_graph,
            retain_graph=create_graph
        )[0]
        # Extract the i-th component of the gradient (diagonal of Jacobian)
        div = div + grad_i[..., i:i+1]
    return div

def compute_laplacian(model, x, create_graph=True):
    """Compute Laplacian of model(x) with respect to x. Assumes model(x) has shape [..., 1]."""
    # Laplacian is divergence of gradient
    grad = compute_gradient(model, x, create_graph=True)
    # Note: grad has shape [..., D]
    # We need to compute divergence of grad w.r.t. x
    # However, compute_divergence expects the input to be the vector field.
    # Here, the vector field is 'grad', and we differentiate w.r.t. 'x'.
    # We can reuse the logic of compute_divergence but with 'grad' as the output.
    
    # Re-implementing divergence logic for 'grad' w.r.t. 'x'
    D = grad.shape[-1]
    lap = torch.zeros_like(grad[..., 0])
    for i in range(D):
        # Gradient of the i-th component of grad w.r.t. x
        grad_grad_i = torch.autograd.grad(
            outputs=grad[..., i:i+1],
            inputs=x,
            grad_outputs=torch.ones_like(grad[..., i:i+1]),
            create_graph=create_graph,
            retain_graph=create_graph
        )[0]
        lap = lap + grad_grad_i[..., i:i+1]
    return lap
```

**Edge Cases and Pitfalls:**
- **Precision:** Higher-order derivatives (Laplacian) are sensitive to numerical precision. Using `float64` may be necessary for stability in some cases [G].
- **Memory:** `create_graph=True` increases memory usage as the gradient computation graph is retained. This can lead to out-of-memory errors for large batches or high-resolution inputs [G].
- **Boundary Conditions:** Autograd computes derivatives at the sampled points. It does not inherently handle boundary conditions of the domain unless the model is trained to respect them [G].

## Subtopics

### [Implicit Differentiation for Sphere Tracing](2012-implicit-differentiation-for-sphere-tracing.md)
A method to compute gradients of the sphere tracing output with respect to input parameters (like SDF parameters) using implicit differentiation, avoiding the need to backpropagate through the iterative loop [2012-implicit-differentiation-for-sphere-tracing.md]. It is chosen when sphere tracing is used for rendering and differentiability with respect to scene geometry is required. It differs from Neural Field Derivatives via Autograd by operating on the fixed-point iteration of sphere tracing rather than the neural field itself.

### SDF Laplacian Estimation
A numerical estimation of the Laplacian $\Delta f$ of an SDF $f$ at a point $p$, using the average value of $f$ over a small sphere centered at $p$ [S2, unofficial]. It is chosen when analytical derivatives are unavailable or when a mesh-free approximation is preferred. It differs from the autograd Laplacian by using finite-difference-like averaging rather than backpropagation.
- **Formula:** $\overline{f}(p, h) \approx f(p) - \frac{(\Delta f)(p)}{2n}h^2$ [S2, unofficial].
- **Approximation:** $(\Delta f)(p) \approx \frac{6}{\epsilon^2} \left( f(p) - \frac{1}{4}\sum_{i=0}^3f(p+\epsilon v_i) \right)$ [S2, unofficial].
- **Parameters:** $n=3$ (dimension), $\epsilon$ (sphere radius), $v_i$ (vectors defining points on the sphere) [S2, unofficial].
- **Pitfall:** The sign convention for the Laplacian is positive-definite [S2, unofficial].

### Jacobian-Free Reparameterization
A technique to compute Jacobian-vector products without explicit matrix construction, using the `linearize` operator in JAX [S3]. It is chosen for memory efficiency when dealing with large Jacobians. It differs from the autograd method by avoiding the construction of the full Jacobian matrix, instead applying the linearized function twice to the covariance matrix $\Sigma$ [S3].

## Operating characteristics
- **Time Complexity:** Gradient computation is $O(N)$ where $N$ is the number of parameters in the model, due to backpropagation. Divergence and Laplacian require multiple gradient computations, scaling linearly with the dimension $D$ (e.g., $D$ times for divergence, $2D$ times for Laplacian if implemented naively, though optimized implementations may reduce this).
- **Memory Complexity:** Memory usage is proportional to the number of parameters and the batch size. `create_graph=True` increases memory overhead as the gradient graph is retained.
- **Performance:** Exact derivatives are computed, avoiding approximation errors from finite differences. However, the cost of multiple backpropagations for higher-order derivatives can be significant.
- **Strengths:** Exact derivatives, easy to implement in modern deep learning frameworks, differentiable end-to-end.
- **Limitations:** High memory usage for higher-order derivatives, potential numerical instability for very high-order derivatives.
- **When to use:** When exact spatial derivatives of a neural field are needed for supervision or optimization.
- **When not to use:** When memory is constrained and finite-difference approximations are acceptable, or when analytical derivatives are available and more efficient.

## Background
The method leverages automatic differentiation, a core feature of modern deep learning frameworks like PyTorch and JAX. It was popularized in the context of differentiable rendering and neural implicit representations, where gradients of the implicit function are required for optimization [S1, unofficial][S3]. The specific application to SDF Laplacian estimation using averaging is discussed in the context of ray marching and SDF manipulation [S2, unofficial].

## Sources
[S1] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
