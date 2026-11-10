# Automatic Differentiation for Rendering

Also called: Autograd-based Differentiable Rendering, Differentiable Rendering via Computational Graphs

## Overview
Automatic Differentiation (AD) for rendering computes the derivatives of interior rendering integrands by applying reverse-mode automatic differentiation directly to the rendering program [S1]. It solves the problem of efficiently obtaining gradients for optimization-based inverse rendering by leveraging the fact that discretization commutes with differentiation for smooth interior integrands [S1]. Unlike the Adjoint Method, which derives custom adjoint equations for specific rendering paths, AD treats the renderer as a black-box computational graph, allowing the use of existing AD frameworks [S1]. However, it is distinct from Source Transformation approaches in that it typically operates via runtime tracing or tape recording rather than compile-time code generation [S1, S2].

## How it works
**Inputs and Outputs:**
- **Input:** A forward rendering program (or computational graph) $F$ that maps scene parameters $\theta$ (e.g., material properties, geometry) to pixel values $I$.
- **Output:** The gradient $\frac{\partial L}{\partial \theta}$ of a loss function $L$ (typically a function of $I$) with respect to $\theta$.

**Data Structures:**
The method relies on the computational graph representation provided by the deep learning framework.
- **Forward Tape/Graph:** A record of operations performed during the forward pass. For tracing-based AD, this is an implicit or explicit list of operations $(op, inputs, outputs)$ [S1].
- **Gradient Accumulators:** Memory buffers storing partial derivatives $\frac{\partial L}{\partial x}$ for each variable $x$ in the graph.

**Steps:**
1. **Forward Pass:** Execute the rendering program $F(\theta)$ using the deep learning framework's tensor operations. The framework records the sequence of operations and intermediate values required for backpropagation [S1].
2. **Loss Computation:** Compute the scalar loss $L$ from the rendered image $I$.
3. **Backward Pass (Reverse AD):** Traverse the recorded computational graph in reverse order. For each operation $y = f(x)$, compute the local gradient $\frac{\partial y}{\partial x}$ and propagate the upstream gradient $\frac{\partial L}{\partial y}$ to $\frac{\partial L}{\partial x}$ using the chain rule:
   $$ \frac{\partial L}{\partial x} = \frac{\partial L}{\partial y} \cdot \frac{\partial y}{\partial x} $$
   [S1]
4. **Gradient Aggregation:** Sum gradients for shared variables (e.g., if a material parameter is used in multiple paths) [S1].
5. **Return:** Output the accumulated gradients $\frac{\partial L}{\partial \theta}$ for use in an optimizer [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode illustrating the usage of a framework like PyTorch
# Note: The rendering engine itself is treated as a sequence of differentiable ops.

def render_forward(scene_params):
    """
    Simulates a differentiable rendering pipeline using standard tensor ops.
    In practice, this involves ray tracing, shading, etc., implemented via
    differentiable kernels or libraries.
    """
    # Example: Simple differentiable shading
    # 1. Sample rays
    rays = sample_rays(scene_params.camera)
    
    # 2. Intersect with geometry (requires differentiable intersection)
    hits = intersect_geometry(rays, scene_params.geometry)
    
    # 3. Compute shading (diffuse, specular, etc.)
    # This part must be implemented with differentiable operations
    color = compute_shading(hits, scene_params.materials)
    
    return color

def compute_loss(rendered_image, target_image):
    """Compute MSE loss."""
    return torch.mean((rendered_image - target_image) ** 2)

def optimize_scene(scene_params, target_image, num_iterations):
    """
    Uses reverse-mode AD to optimize scene parameters.
    """
    # Ensure gradients are tracked
    scene_params.requires_grad_(True)
    
    optimizer = torch.optim.Adam([scene_params], lr=0.01)
    
    for i in range(num_iterations):
        # 1. Forward pass: Record operations in the computational graph
        rendered_image = render_forward(scene_params)
        
        # 2. Compute loss
        loss = compute_loss(rendered_image, target_image)
        
        # 3. Backward pass: Compute gradients via reverse AD
        loss.backward()
        
        # 4. Update parameters
        optimizer.step()
        
        # 5. Zero gradients for next iteration
        optimizer.zero_grad()
        
    return scene_params
```

**Edge Cases and Pitfalls:**
- **Memory Usage:** The computational graph for rendering is extremely large (millions of operations) and unstructured. Storing the forward tape for reverse AD can consume excessive memory, leading to out-of-memory errors [S1].
- **Arithmetic Intensity:** Rendering operations often have low arithmetic intensity (e.g., many additions, few multiplications). This leads to excessive memory traffic when storing intermediate values for backpropagation, making it significantly slower than high-intensity neural network operations [S1].
- **Discontinuities:** AD assumes smoothness. It fails to correctly handle discontinuities in the rendering integrand (e.g., visibility changes, shadow boundaries) because the derivative is undefined or zero almost everywhere at the discontinuity. Special handling (e.g., path space deformation) is required for these cases, which standard AD does not provide [S1].

## Subtopics

### [Source Transformation Approach](7118-source-transformation-approach.md)
The Source Transformation Approach is an automatic differentiation (AD) strategy that converts the source code of a program into its derivative at compile-time, treating differentiation as a one-time compilation step rather than a runtime operation. It solves the problem of runtime overhead and optimization inhibition inherent in tracing-based AD by generating optimized, standalone derivative code. Unlike tracing approaches, it preserves the original control flow, allowing generated programs to function correctly regardless of loop iteration counts or dynamic branching.

### [Dressi-AD](10746-dressi-ad.md)
Dressi-AD is a hardware-agnostic automatic differentiation library for differentiable rendering (DR) built on the Vulkan API. It solves the problem of integrating reverse-mode AD into a graphics pipeline by using 2D images as primitive data structures instead of multi-dimensional tensors, maintaining compatibility with hardware rasterizers. Unlike general-purpose AD libraries that rely on multi-dimensional tensors, Dressi-AD aligns with graphics pipeline constraints and differentiable rendering workflows by utilizing fragment shaders with dummy vertex shaders for computation.

## Operating characteristics
- **Time Complexity:** The backward pass is proportional to the forward pass, typically $O(N)$ where $N$ is the number of operations in the computational graph [S1]. However, due to low arithmetic intensity, the constant factor is high due to memory bandwidth limitations [S1].
- **Memory Complexity:** Requires storing all intermediate values from the forward pass for the backward pass, leading to $O(N)$ memory usage [S1]. This is a major bottleneck for large scenes [S1].
- **Measured Performance:** Implementing rendering pipelines directly in frameworks like PyTorch or TensorFlow is extremely inefficient compared to specialized AD compilers or adjoint methods due to the unstructured nature of rendering graphs and the overhead of tensor management [S1].
- **Strengths:** Easy to implement if the rendering primitives are already differentiable; leverages existing AD infrastructure; no need to manually derive adjoint equations [S1].
- **Limitations:** High memory consumption; poor performance on GPUs due to low arithmetic intensity; cannot handle discontinuities in the integrand correctly [S1].
- **When to Use:** Suitable for small-scale differentiable rendering tasks or when rapid prototyping is prioritized over performance. Not recommended for production-scale inverse rendering with large scenes [S1].

## Background
The technique is based on the principle that for interior rendering integrands, discretization commutes with differentiation, allowing standard automatic differentiation techniques to be applied directly to the rendering program [S1]. Widely used deep learning frameworks such as PyTorch and TensorFlow provide built-in automatic differentiation interfaces [S1]. However, these frameworks were designed for linear algebra and neural networks, which have high arithmetic intensity and structured graphs, unlike rendering algorithms which produce large, unstructured graphs with low arithmetic intensity [S1]. This mismatch necessitates specialized AD compilers for efficient differentiable rendering [S1].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
