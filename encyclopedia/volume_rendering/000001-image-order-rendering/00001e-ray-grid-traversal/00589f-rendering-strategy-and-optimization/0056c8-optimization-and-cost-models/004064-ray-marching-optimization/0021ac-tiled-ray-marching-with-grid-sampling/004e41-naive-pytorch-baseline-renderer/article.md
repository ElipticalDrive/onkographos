# Naive PyTorch Baseline Renderer
Also called: Sequential CPU loop, Standard tensor operations renderer

## Overview
The Naive PyTorch Baseline Renderer is a reference implementation for differentiable rendering that processes all primitives for every pixel sequentially using standard PyTorch tensor operations [S1, unofficial]. It serves as a performance baseline to demonstrate the efficiency gains of optimized CUDA implementations, which utilize tile-based culling, per-pixel parallelism, and FP16 optimization [S1, unofficial]. Unlike its siblings, it lacks spatial optimization, tile-based binning, or specialized kernels, resulting in significant memory and computational overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ primitives, where each primitive $i$ is defined by parameters $\{(x_i, y_i, s_i, \theta_i, \nu_i, c_i)\}_{i=1}^N$. Here, $(x_i, y_i)$ is the center, $s_i$ is the scale, $\theta_i$ is the rotation angle, $\nu_i$ is a density parameter, and $c_i$ is the base color. A mask texture $M_i$ is associated with each primitive.
*   **Outputs:** A rendered image tensor $I$ of size $H_{canvas} \times W_{canvas}$.

**Data Structures:**
*   **Canvas Grids:** Two 2D tensors $X$ and $Y$ of shape $(H_{canvas}, W_{canvas})$ representing the normalized coordinates of each pixel.
*   **Accumulators:** Two 2D tensors $I$ (image) and $T$ (transparency) of shape $(H_{canvas}, W_{canvas})$, initialized to zeros and ones respectively.
*   **Intermediate Tensors:** For each primitive, full-canvas tensors are allocated for $\Delta x, \Delta y, u, v,$ and $mask$.

**Algorithm Steps:**
1.  Generate coordinate grids $X, Y$ over the canvas dimensions $H_{canvas}, W_{canvas}$ using `torch.meshgrid`.
2.  Initialize the output image $I$ to zeros and the transparency mask $T$ to ones.
3.  Iterate through each primitive $i$ from $0$ to $N-1$:
    a.  Compute displacement vectors: $\Delta x = X - x_i$ and $\Delta y = Y - y_i$.
    b.  Transform coordinates to the primitive's local space: rotate and scale $\Delta x, \Delta y$ by angle $\theta_i$ and scale $s_i$ to obtain local coordinates $u, v$.
    c.  Sample the primitive's mask texture $M_i$ at $(u, v)$ using `torch.grid_sample` to get the mask value.
    d.  Compute opacity $\alpha_i$ and color $c_i$:
        $$ \alpha_i = \alpha_{max} \cdot \sigma(\nu_i) \cdot \text{mask} $$
        $$ c_i = \sigma(c_i) $$
        where $\sigma$ is the sigmoid function and $\alpha_{max}$ is a maximum opacity constant [S1, unofficial].
    e.  Update the image and transparency using alpha compositing:
        $$ I \leftarrow I + T \cdot \alpha_i \cdot c_i $$
        $$ T \leftarrow T \cdot (1 - \alpha_i) $$
4.  Return the final image $I$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def naive_pytorch_renderer(primitives, H_canvas, W_canvas, alpha_max=1.0):
    """
    primitives: List of dicts or tensors containing x, y, s, theta, nu, c, and mask_texture M.
    H_canvas, W_canvas: Integer dimensions of the output image.
    alpha_max: Maximum opacity scaling factor.
    """
    # 1. Create meshgrid X, Y over canvas dimensions
    # Assuming normalized coordinates [-1, 1] or [0, 1] consistent with grid_sample
    y, x = torch.meshgrid(
        torch.linspace(-1, 1, H_canvas, device=primitives[0]['x'].device),
        torch.linspace(-1, 1, W_canvas, device=primitives[0]['x'].device),
        indexing='ij'
    )
    
    # 2. Initialize output image I to zeros and transparency mask T to ones
    I = torch.zeros((H_canvas, W_canvas), device=primitives[0]['x'].device)
    T = torch.ones((H_canvas, W_canvas), device=primitives[0]['x'].device)
    
    N = len(primitives)
    
    # 3. Loop i from 0 to N-1
    for i in range(N):
        prim = primitives[i]
        xi = prim['x']
        yi = prim['y']
        si = prim['s']
        theta_i = prim['theta']
        nu_i = prim['nu']
        ci = prim['c']
        Mi = prim['mask_texture']
        
        # 4. Compute deltas ∆x = X - xi, ∆y = Y - yi
        dx = x - xi
        dy = y - yi
        
        # 5. Rotate/scale deltas by primitive angle θi and scale si to get u, v
        # Rotation matrix for angle theta:
        # u = (dx * cos(theta) + dy * sin(theta)) / scale
        # v = (-dx * sin(theta) + dy * cos(theta)) / scale
        cos_theta = torch.cos(theta_i)
        sin_theta = torch.sin(theta_i)
        
        u = (dx * cos_theta + dy * sin_theta) / si
        v = (-dx * sin_theta + dy * cos_theta) / si
        
        # 6. Sample mask from primitive's mask texture Mi at (u, v) using torch.grid_sample
        # grid_sample expects input of shape (N, C, H, W) and grid of shape (..., 2)
        # Assuming Mi is (1, 1, H_m, W_m) and u,v are (H, W)
        # We need to unsqueeze u,v to (1, H, W, 2)
        grid = torch.stack([u, v], dim=-1).unsqueeze(0)
        mask = F.grid_sample(Mi, grid, mode='bilinear', padding_mode='zeros', align_corners=True)
        mask = mask.squeeze() # Remove batch and channel dims if present
        
        # 7. Compute opacity αi = αmax · σ(νi) · mask and color ci = σ(ci)
        # Note: ci in the source formula refers to the base color, 
        # but the formula uses sigma(ci). Assuming ci is a scalar or tensor passed to sigmoid.
        # If ci is RGB, sigmoid is applied element-wise.
        alpha_i = alpha_max * torch.sigmoid(nu_i) * mask
        color_i = torch.sigmoid(ci)
        
        # 8. Update image I = I + T · αi · ci and transparency T = T · (1 - αi)
        I = I + T * alpha_i * color_i
        T = T * (1 - alpha_i)
        
    # 9. Return I
    return I
```

**Edge Cases and Pitfalls:**
*   **Memory Allocation:** The method allocates full-canvas tensors for every primitive in the loop. For 1000 primitives on a $512 \times 512$ canvas, this results in approximately 5 GB of memory allocation at FP32 precision due to intermediate tensors $\Delta x, \Delta y, u, v,$ and $mask$ [S1, unofficial].
*   **Wasted Evaluations:** Every primitive is evaluated at all $H \times W$ pixels, even if the primitive contributes nothing to the final image (e.g., outside the canvas or with zero opacity). For 1000 primitives on a $512^2$ canvas, this yields 262 million wasted evaluations [S1, unofficial].
*   **Sequential Execution:** The Python `for` loop prevents primitive-level parallelism and forces repeated kernel launches, significantly slowing down execution compared to GPU-optimized approaches [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \times H \times W)$, as every primitive is evaluated at all $H \times W$ pixels [S1, unofficial].
*   **Memory Complexity:** $O(N \times H \times W)$ per iteration because each primitive creates full-canvas tensors for intermediate calculations [S1, unofficial].
*   **Performance:** The naive version is 30-50x slower than the optimized CUDA implementation [S1, unofficial].
*   **Strengths:** Simple implementation using standard PyTorch operations; serves as a clear baseline for comparison.
*   **Limitations:** Massive memory allocation, sequential processing, lack of spatial culling, and generic interpolation operations [S1, unofficial].
*   **When to use:** Only for small-scale testing or as a reference for correctness against optimized renderers.
*   **When not to use:** For large scenes, high-resolution images, or any performance-critical application.

## Background
The Naive PyTorch Baseline Renderer was introduced in the paper "DiffBMP: Differentiable Rendering with Bitmap Primitives" [S1, unofficial]. It was designed to demonstrate the efficiency of a custom CUDA implementation by comparing it against a straightforward PyTorch-based renderer that uses standard tensor operations without tile-based culling or specialized kernels [S1, unofficial]. The paper highlights the major bottlenecks of this approach, including no spatial culling, massive memory allocation, sequential loops, and generic operations [S1, unofficial].

## Sources
[S1] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
