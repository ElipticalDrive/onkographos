# Neural Field Derivatives via Autograd

Also called: Differential Operators (Gradient, Divergence, Laplace), Gradient, Laplace via Autograd, autograd operators, neural field derivatives

## Overview
This method computes spatial derivatives (gradient, divergence, Laplacian) of neural field outputs with respect to their spatial inputs using reverse-mode automatic differentiation (AD) [S1, unofficial]. It enables supervision on gradient targets, such as enforcing Eikonal constraints or smoothness priors, by differentiating the model's output directly [S1, unofficial]. Unlike the Adjoint Method, which propagates adjoint variables backward through the rendering algorithm to optimize parameters, this approach focuses on computing specific differential operators on the field values themselves, often requiring the retention of the computational graph for higher-order derivatives [S1, unofficial]. It differs from Forward-Mode AD by computing gradients for all inputs simultaneously with respect to a single output, making it suitable for scalar loss terms derived from field properties [S1, unofficial][S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A neural field model $f(\mathbf{x})$ (e.g., an MLP taking 3D coordinates $\mathbf{x}$ and returning density, color, or SDF values), and a batch of input points $\mathbf{x}$.
*   **Output:** The computed derivative tensor (e.g., gradient vector, divergence scalar, or Laplacian scalar) with respect to $\mathbf{x}$.

**Data Structures:**
*   **Tensors:** PyTorch or JAX tensors representing the input coordinates $\mathbf{x}$ (shape $[N, 3]$) and the model output $y = f(\mathbf{x})$ (shape $[N, C]$).
*   **Computation Graph:** The forward pass must be recorded in the AD engine's graph.

**Parameters:**
*   `create_graph=True`: A boolean flag passed to the differentiation function. When true, the gradient computation itself is added to the computational graph, allowing for higher-order derivatives (e.g., Laplacian as divergence of gradient) [S1, unofficial].

**Steps:**

1.  **Forward Pass:** Compute the model output $y = f(\mathbf{x})$.
2.  **Gradient Computation:** Compute the gradient of the output $y$ with respect to the input $\mathbf{x}$ using the AD engine.
    *   Formula: $\nabla_{\mathbf{x}} y = \frac{\partial y}{\partial \mathbf{x}}$.
    *   Implementation detail: Use `torch.autograd.grad` with `grad_outputs=torch.ones_like(y)` and `create_graph=True` [S1, unofficial].
3.  **Divergence Computation (if required):** Compute the divergence of a vector field $\mathbf{v}$ (where $\mathbf{v}$ is the output of a model or the gradient from step 2).
    *   Formula: $\nabla \cdot \mathbf{v} = \sum_{i} \frac{\partial v_i}{\partial x_i}$.
    *   Implementation: For each component $i$ of the output $\mathbf{v}$, compute the gradient of $v_i$ with respect to $\mathbf{x}$, then sum the diagonal elements corresponding to the spatial dimensions [S1, unofficial].
4.  **Laplacian Computation (if required):** Compute the Laplacian $\Delta y$ as the divergence of the gradient.
    *   Formula: $\Delta y = \nabla \cdot (\nabla y) = \sum_{i} \frac{\partial^2 y}{\partial x_i^2}$.
    *   Implementation: Apply the divergence operator to the gradient computed in Step 2 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_gradient(model, x, create_graph=True):
    """
    Compute the gradient of the model output with respect to input x.
    
    Args:
        model: A PyTorch nn.Module representing the neural field.
        x: Input coordinates tensor of shape [N, 3].
        create_graph: Boolean, if True, keeps the graph for higher-order derivatives.
        
    Returns:
        grad: Gradient tensor of shape [N, C] where C is the output dimension.
    """
    x.requires_grad_(True)
    y = model(x)
    # grad_outputs=torch.ones_like(y) ensures we compute the sum of gradients
    # for all output channels if y is multi-dimensional.
    grad = torch.autograd.grad(
        y, 
        x, 
        grad_outputs=torch.ones_like(y), 
        create_graph=create_graph,
        retain_graph=True # Necessary if we need to compute divergence later
    )[0]
    return grad

def compute_divergence(model, x, create_graph=True):
    """
    Compute the divergence of the model output vector field.
    Assumes model outputs a 3D vector field [N, 3].
    """
    x.requires_grad_(True)
    y = model(x)
    
    div = 0.0
    # Sum of partial derivatives of each component with respect to its coordinate
    for i in range(3):
        # Compute gradient of the i-th output component w.r.t x
        grad_i = torch.autograd.grad(
            y[:, i], 
            x, 
            grad_outputs=torch.ones_like(y[:, i]), 
            create_graph=create_graph,
            retain_graph=True
        )[0]
        # Sum the diagonal: sum over batch of d(y_i)/d(x_i)
        # Note: grad_i shape is [N, 3]. We want sum(grad_i[:, i])
        div += grad_i[:, i].sum()
        
    return div / x.shape[0] # Average over batch

def compute_laplacian(model, x, create_graph=True):
    """
    Compute the Laplacian of the model output scalar field.
    Assumes model outputs a scalar [N, 1] or [N].
    """
    # Laplacian is divergence of gradient
    # First, get gradient (vector field)
    grad = compute_gradient(model, x, create_graph=True)
    
    # Now compute divergence of this gradient field
    # Note: grad is [N, 3] if input was [N, 3] and output was scalar
    # We treat grad as the vector field v
    x.requires_grad_(True) # Ensure x is tracked for the second derivative
    
    laplacian = 0.0
    for i in range(3):
        # Gradient of the i-th component of the gradient (i.e., second derivative)
        grad_grad_i = torch.autograd.grad(
            grad[:, i], 
            x, 
            grad_outputs=torch.ones_like(grad[:, i]), 
            create_graph=False, # No need for 3rd order
            retain_graph=True
        )[0]
        laplacian += grad_grad_i[:, i].sum()
        
    return laplacian / x.shape[0]
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Using `create_graph=True` stores the gradient computation in the graph, which increases memory consumption compared to standard backpropagation [S1, unofficial].
*   **Precision:** Higher-order derivatives (Laplacian) can be sensitive to numerical noise in the neural field outputs [S1, unofficial].
*   **Batch Size:** Divergence and Laplacian calculations often involve summing over the batch dimension; ensure normalization is consistent [S1, unofficial].

## Subtopics

### [Implicit Differentiation for Sphere Tracing](2012-implicit-differentiation-for-sphere-tracing.md)
This method computes the gradients of the sphere tracing output (specifically the final converged ray positions or associated radiance) with respect to the input parameters of the Signed Distance Function (SDF) model, without backpropagating through the iterative sphere tracing loop. It solves the problem of differentiating an iterative fixed-point solver by applying the implicit function theorem, treating the convergence condition as an implicit constraint. It differs from naive backpropagation through the loop by avoiding the accumulation of computational graph overhead from multiple iterations and by handling the non-differentiable convergence mask explicitly.

### SDF Laplacian Estimation
A numerical estimation of the Laplacian $\Delta f$ of an SDF $f$ at a point $p$, using the average value of $f$ over a small sphere centered at $p$ [S2, unofficial].
*   **Formula:**
    $$ \overline{f}(p, h) \approx f(p) - \frac{(\Delta f)(p)}{2n}h^2 $$
    $$ \overline{f}(p, \epsilon) \approx \frac{1}{4}\sum_{i=0}^3f(p+\epsilon v_i) $$
    $$ (\Delta f)(p) \approx \frac{6}{\epsilon^2} \left( f(p) - \frac{1}{4}\sum_{i=0}^3f(p+\epsilon v_i) \right) $$
*   **Parameters:**
    *   $n$: The dimension of the space (3 for 3D SDFs) [S2, unofficial].
    *   $\epsilon$: The radius of the small sphere used for averaging [S2, unofficial].
    *   $v_i$: Vectors defining the points on the sphere for averaging ($i=0$ to $3$) [S2, unofficial].
*   **Pitfall:** The sign convention for the Laplacian is positive-definite [S2, unofficial].

### Jacobian-Free Reparameterization
In Jax, compute the Jacobian $J_f(\mu)$ application without explicit matrix construction by using the `linearize` operator, applying it twice in sequence to $\Sigma$, with dimensions of the covariance matrix transposed after each application [S3].

### Efficient Second-Order Derivatives for ReLU MLPs
A technique to compute second-order derivatives of ReLU-based MLPs efficiently by exploiting the property that the second derivative of the MLP output with respect to its input is zero, simplifying the chain rule calculations for backpropagation of Eikonal loss [S4].
*   **Definitions:**
    *   $P_l^j = G_l H_{l-1} \dots G_2 H_1^{(-j)}$
    *   $S_l^i = H_L^{(i,-)} G_L \dots H_{l+1} G_{l+1}$
    *   $G_l = \begin{cases} 1, & H_{l-1} \dots g(H_1 x) > 0 \\ 0, & \text{otherwise} \end{cases}$
    *   $\frac{\partial^2 y}{\partial x^2} = 0$
*   **Parameters:**
    *   $L$: number of hidden layers in the ReLU-based MLP [S4].
    *   $H_l$: weight matrix for layer $l$, dimensions $\mathbb{R}^{n_l \times \mathbb{R}^{n_{l-1}}}$ [S4].
    *   $n_l$: number of neurons in layer $l$ [S4].
    *   $P_l^j$: matrix in $\mathbb{R}^{n_{l-1} \times 1}$, used in second-order derivative calculation [S4].
    *   $S_l^i$: matrix in $\mathbb{R}^{1 \times n_l}$, used in second-order derivative calculation [S4].

## Operating characteristics
*   **Time Complexity:** Computing the gradient is $O(N \cdot D_{in} \cdot D_{out})$ for a single forward/backward pass, where $N$ is the batch size. Computing the Laplacian requires two passes (gradient then divergence), effectively doubling the computational cost [S1, unofficial].
*   **Memory Complexity:** Using `create_graph=True` increases memory usage as the gradient computation graph is stored. This can be significant for large batches or high-dimensional outputs [S1, unofficial].
*   **Strengths:** Simple to implement using standard deep learning frameworks; enables direct supervision on geometric properties (e.g., Eikonal loss) [S1, unofficial].
*   **Limitations:** High memory consumption for higher-order derivatives; numerical instability for noisy fields [S1, unofficial].
*   **When to use:** When gradient-based supervision is required for neural fields (e.g., NeuS, SDFs) and the computational graph can be retained [S1, unofficial].
*   **When not to use:** When memory is constrained and higher-order derivatives are not strictly necessary, or when the field is too noisy for stable gradient computation [S1, unofficial].

## Background
The method is rooted in the use of automatic differentiation in differentiable rendering, particularly in neural implicit surface reconstruction methods like NeuS [S4]. NeuS leverages volume rendering to learn signed distance fields and radiance fields from multi-view images, requiring differentiable operations on the SDF [S4]. The use of `create_graph=True` in PyTorch allows for the computation of higher-order derivatives necessary for enforcing geometric constraints like the Eikonal equation [S1, unofficial]. The efficiency of second-order derivatives for ReLU MLPs, as described in NeuS2, further optimizes this process by exploiting the sparsity of second-order gradients [S4].

## Sources
[S1] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S4] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
