# Differentiable Bitmap Splatting

Also called: DiffBMP, DiffBMP Renderer, Differentiable Rendering with Bitmap Primitives

## Overview
Differentiable Bitmap Splatting (DiffBMP) is a scalable rendering engine for optimizing the position, rotation, scale, color, and opacity of thousands of bitmap primitives using custom CUDA kernels for forward and backward passes [S1, unofficial]. It addresses the limitation that traditional differentiable renderers are constrained to vector graphics, enabling the optimization of raster images which constitute most visual content [S1, unofficial]. Unlike prior bitmap methods that lacked specialized parallelism and were limited to repetitive, opaque patterns, DiffBMP supports diverse, semi-transparent compositions with high performance [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ bitmap primitives. Each primitive $i$ has parameters: position $(x_i, y_i)$, scale $s_i$, rotation $\theta_i$, opacity logit $\nu_i$, and RGB color logit $\mathbf{c}_i \in \mathbb{R}^3$. Each primitive has an associated bitmap texture of size $H_i \times W_i$.
*   **Outputs:** A rendered image $I(x,y) \in [0,1]^3$ and gradients with respect to all primitive parameters.

**Data Structures**
*   **Primitive Parameters:** Stored in arrays accessible by GPU threads.
*   **Bitmap Textures:** 2D textures sampled via bilinear interpolation.
*   **Gradient Buffers:** Packed `half2` buffers for accumulating per-parameter gradients per pixel.

**Steps**
1.  **Tile Partitioning (CPU):** Partition the image plane into $T \times T$ tiles (default $T=32$). Assign each primitive to every tile whose bounding box on the image plane overlaps the tile region, with a small padding margin [S1, unofficial].
2.  **Kernel Launch (GPU):** Launch one CUDA thread block per tile, with $T \times T$ threads per block [S1, unofficial].
3.  **Coordinate Transformation:** For each pixel $(x,y)$ and primitive $i$, compute normalized primitive coordinates $(u,v)$:
    $$
    \begin{bmatrix} u \\ v \end{bmatrix} = \begin{bmatrix} \cos \theta_i & \sin \theta_i \\ -\sin \theta_i & \cos \theta_i \end{bmatrix} \begin{bmatrix} (x-x_i)/s_i \\ (y-y_i)/s_i \end{bmatrix} \in [-1,1]^2
    $$
4.  **Bitmap Coordinate Mapping:** Map $(u,v)$ to discrete bitmap coordinates $(U,V)$:
    $$
    \begin{bmatrix} U \\ V \end{bmatrix} = \begin{bmatrix} (u+1)/2 \cdot (W_i-1) \\ (v+1)/2 \cdot (H_i-1) \end{bmatrix} \in [0, W_i-1] \times [0, H_i-1]
    $$
5.  **Primitive Contribution Sampling:** Compute $M_i(x,y)$ via bilinear interpolation using the four nearest integer grid points $\{\lfloor U \rfloor, \lfloor U \rfloor+1\} \times \{\lfloor V \rfloor, \lfloor V \rfloor+1\}$ from the primitive's bitmap texture [S1, unofficial].
6.  **Alpha Compositing:** Compute the alpha value for primitive $i$ at pixel $(x,y)$:
    $$
    \alpha_i(x,y) = \alpha_{\max} \cdot \sigma(\nu_i) \cdot M_i(x,y)
    $$
    where $\sigma$ is the sigmoid function and $\alpha_{\max} \in (0,1]$ is a hyperparameter [S1, unofficial].
7.  **Final Color Compositing:** Compute transmittance $T_k(x,y)$ and final color $I(x,y)$ using Porter-Duff over compositing:
    $$
    T_k(x,y) = \prod_{j=0}^{k-1}(1 - \alpha_j(x,y))
    $$
    $$
    I(x,y) = \sum_{k=0}^{N-1} T_k(x,y) \alpha_k(x,y) \sigma(\mathbf{c}_k) \in [0,1]^3
    $$
    Primitives are composited in front-to-back order [S1, unofficial].
8.  **Backward Pass:** Accumulate per-parameter gradients per pixel directly into packed `half2` buffers via `half2` atomicAdd over two-channel gradient groups. A lightweight post-pass unpacks these to legacy FP16 arrays [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Forward Pass (CUDA Kernel per Tile)
def diffbmp_forward(tile_primitives, pixel_coords, textures, alpha_max):
    # tile_primitives: list of primitive indices assigned to this tile
    # pixel_coords: (x, y) for each thread
    # textures: 2D texture objects for each primitive
    # alpha_max: scalar hyperparameter
    
    # Initialize output buffers for this tile
    color = (0.0, 0.0, 0.0)
    transmittance = 1.0
    
    # Sort primitives by depth or use fixed order (front-to-back assumed)
    # In DiffBMP, order is fixed per primitive index k
    
    for k, prim_idx in enumerate(tile_primitives):
        # Extract parameters for primitive k
        x_k, y_k = prim_params[prim_idx].pos
        s_k = prim_params[prim_idx].scale
        theta_k = prim_params[prim_idx].rotation
        nu_k = prim_params[prim_idx].opacity_logit
        c_k = prim_params[prim_idx].color_logit
        W_k, H_k = prim_params[prim_idx].texture_size
        
        # 1. Coordinate Transformation
        dx = (pixel_coords.x - x_k) / s_k
        dy = (pixel_coords.y - y_k) / s_k
        u = dx * cos(theta_k) + dy * sin(theta_k)
        v = -dx * sin(theta_k) + dy * cos(theta_k)
        
        # 2. Bitmap Coordinate Mapping
        U = (u + 1.0) / 2.0 * (W_k - 1.0)
        V = (v + 1.0) / 2.0 * (H_k - 1.0)
        
        # 3. Bilinear Interpolation (Primitive Contribution M_k)
        # Get floor coordinates
        U_floor = floor(U)
        V_floor = floor(V)
        # Get fractional parts
        u_frac = U - U_floor
        v_frac = V - V_floor
        
        # Sample 4 corners from texture
        # Note: Handle boundary conditions (clamp or repeat) as per implementation
        c00 = texture_sample(textures[prim_idx], U_floor, V_floor)
        c10 = texture_sample(textures[prim_idx], U_floor + 1, V_floor)
        c01 = texture_sample(textures[prim_idx], U_floor, V_floor + 1)
        c11 = texture_sample(textures[prim_idx], U_floor + 1, V_floor + 1)
        
        # Interpolate
        c_top = lerp(c00, c10, u_frac)
        c_bot = lerp(c01, c11, u_frac)
        M_k = lerp(c_top, c_bot, v_frac)
        
        # 4. Alpha Compositing
        alpha_k = alpha_max * sigmoid(nu_k) * M_k
        
        # 5. Porter-Duff Over
        color = color + transmittance * alpha_k * sigmoid(c_k)
        transmittance = transmittance * (1.0 - alpha_k)
        
        # If transmittance is negligible, break early (optional optimization)
        if transmittance < 1e-6:
            break
            
    return color, transmittance

# Backward Pass (Gradient Accumulation)
def diffbmp_backward(tile_primitives, pixel_coords, textures, alpha_max, d_color):
    # d_color: gradient of loss w.r.t. output color
    
    # Initialize gradient buffers (packed half2)
    grad_pos = zeros_like(prim_params.pos)
    grad_scale = zeros_like(prim_params.scale)
    grad_rot = zeros_like(prim_params.rotation)
    grad_nu = zeros_like(prim_params.opacity_logit)
    grad_c = zeros_like(prim_params.color_logit)
    
    # Compute gradients via reverse-mode autodiff or manual chain rule
    # The paper uses custom CUDA kernels with atomicAdd
    
    for k, prim_idx in enumerate(tile_primitives):
        # Re-run forward logic to get intermediates (u, v, U, V, M_k, alpha_k, T_k)
        # ... (same as forward) ...
        
        # Compute d_alpha_k / d_params
        # d_alpha_k / d_nu_k = alpha_max * sigmoid(nu_k) * (1 - sigmoid(nu_k)) * M_k
        # d_alpha_k / d_M_k = alpha_max * sigmoid(nu_k)
        
        # d_color / d_alpha_k and d_color / d_T_k from chain rule
        # ... (standard differentiable compositing gradients) ...
        
        # Gradient for M_k (texture contribution)
        # d_M_k = d_alpha_k / d_M_k * d_alpha_k / d_color ...
        
        # Gradient for texture coordinates U, V via bilinear interpolation
        # d_U, d_V from d_M_k
        # d_U, d_V from u, v via Jacobian of mapping
        # d_u, d_v from U, V
        # d_u, d_v from x_k, y_k, s_k, theta_k via Jacobian of affine transform
        
        # Accumulate gradients into packed half2 buffers using atomicAdd
        atomic_add(grad_pos[prim_idx], ...)
        atomic_add(grad_scale[prim_idx], ...)
        atomic_add(grad_rot[prim_idx], ...)
        atomic_add(grad_nu[prim_idx], ...)
        atomic_add(grad_c[prim_idx], ...)
        
    return grad_pos, grad_scale, grad_rot, grad_nu, grad_c
```

**Parameters**
*   $T$: Tile size for partitioning the image plane on the CPU. Default value is 32 [S1, unofficial].
*   $\alpha_{\max}$: Maximum alpha scaling factor. Range $(0, 1]$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Non-integer Coordinates:** $U$ and $V$ are almost surely non-integer, requiring bilinear interpolation for differentiable sampling [S1, unofficial].
*   **Precision:** Texture fetches and per-pixel temporaries use FP16 (packed as `half2` where profitable). Accumulation of displayed color/transmittance is evaluated in FP16 arithmetic. Per-parameter gradients are accumulated per pixel directly into packed `half2` buffers via `half2` atomicAdd over two-channel gradient groups, followed by a lightweight post pass that unpacks to legacy FP16 arrays [S1, unofficial].
*   **Atomic Operations:** The use of atomic operations in optimization kernels makes them unsuitable for generating layered PSD files directly; a dedicated export kernel is required for non-differentiable rendering [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The forward and backward passes are parallelized across pixels and primitives using CUDA. The complexity is dominated by the number of primitives $N$ and the image resolution, with tile-based processing ensuring efficient memory access [S1, unofficial].
*   **Memory Complexity:** Uses FP16 precision to reduce bandwidth and VRAM usage. Gradient accumulation uses packed `half2` buffers [S1, unofficial].
*   **Performance:** Can optimize the position, rotation, scale, color, and opacity of thousands of bitmap primitives in under 1 minute using a consumer GPU [S1, unofficial].
*   **Strengths:** Scalable, efficient, supports diverse bitmap compositions (painterly, assemblage, video), enables differentiable optimization of raster primitives.
*   **Limitations:** Requires CUDA hardware. Atomic operations prevent direct export of editable layers during optimization. Limited to 2D bitmap primitives.
*   **When to use:** When optimizing raster-based compositions (e.g., digital art, video editing) where vector graphics are insufficient or impractical.
*   **When not to use:** When layered PSD export is needed during optimization (use the dedicated export kernel instead). When hardware does not support CUDA.

## Background
DiffBMP was introduced as a practical tool designed to integrate into creative workflows, supporting exporting compositions to a native, layered file format. It addresses the limitation that traditional differentiable renderers are constrained to vector graphics, given that most images in the world are bitmaps [S1, unofficial]. The method builds on Spatial Transformer Networks [17] and prior bitmap methods by Reddy et al. [43], which were implemented without specialized parallelism and limited to repetitive, opaque patterns. DiffBMP unlocks diverse bitmap optimization through custom CUDA kernels [S1, unofficial].

## Sources
[S1] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
