# Local-GS

Also called: Local Gaussian Splatting, Tile-Local Warp Coherence

## Overview
Local-GS is a warp-coherent rendering paradigm that restructures alpha computation and reorganizes work around SIMT (Single Instruction, Multiple Threads) execution boundaries to reduce warp divergence and redundant computation in 3D Gaussian Splatting [S1, unofficial]. Unlike methods that focus on macroscopic scene culling or data compression, Local-GS operates as a pure kernel-level optimization that shifts focus to microscopic GPU execution boundaries, mitigating intra-block computational overhead [S1, unofficial]. It achieves this by enforcing tile-local execution through three stages: parameter hoisting, warp-level culling, and coherent blending [S1, unofficial].

## How it works
Local-GS operates on the set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$ defined in [Rendering Implementation and Acceleration](21069-rendering-implementation-and-acceleration.md). The method processes image tiles (typically $16 \times 16$ pixels) and subdivides them into warp-aligned strips.

### Inputs and Outputs
*   **Inputs:**
    *   Set of projected 2D Gaussians for the current tile, including conic matrix coefficients $[a, b, c]$ and intrinsic opacity $o$.
    *   Tile top-left global pixel coordinates $T = (T_x, T_y)^T$.
    *   Global pixel coordinates $x = (x, y)^T$ for each thread.
*   **Outputs:**
    *   Alpha values $\alpha$ for each pixel covered by the Gaussians.
    *   Blended color and opacity contributions.

### Data Structures
*   **Hoisted Vector ($V_{\text{hoisted}}$):** A 6D vector $[A, B, C, D, E, F]^T$ stored in shared memory.
    *   Layout: One `float4` containing $[A, B, C, F]$ and one `float2` containing $[D, E]$.
    *   This vector is computed once per Gaussian per tile and reused by all threads in the tile.
*   **Local Basis Vector ($B_{\text{local}}$):** A 6D vector $[x'^2, x'y', y'^2, x', y', 1]^T$ computed per pixel/thread.

### Numbered Steps

1.  **Tile-Local Coordinate Reformulation:**
    Shift the reference frame from global screen space to the local coordinate system of the execution tile.
    *   Calculate tile-local pixel coordinates: $x' = x - T_x$, $y' = y - T_y$.
    *   Calculate Gaussian center offsets relative to the tile: $\Delta x = \mu_x - T_x$, $\Delta y = \mu_y - T_y$.
    *   The displacement vector becomes $\tilde{x} = (x' - \Delta x, y' - \Delta y)^T$.
    *   *Note:* This bounds coordinates to approximately $[-16, 32]$, preventing catastrophic cancellation errors associated with large global coordinates (e.g., $\approx 4000$ in 4K resolution) [S1, unofficial].

2.  **Gaussian Parameter Hoisting (Precomputation):**
    For each Gaussian covering the tile, compute the hoisted vector $V_{\text{hoisted}}$ using the canonical polynomial expansion of the log-alpha.
    *   The log-alpha is expanded as: $\ln(\alpha) = A x'^2 + B x'y' + C y'^2 + D x' + E y' + F$.
    *   Compute $V_{\text{hoisted}}$ via the linear transformation:
        $$
        \begin{bmatrix} A \\ B \\ C \\ D \\ E \\ F \end{bmatrix} = 
        \begin{bmatrix} 
        -0.5 & 0 & 0 & 0 \\ 
        0 & -1.0 & 0 & 0 \\ 
        0 & 0 & -0.5 & 0 \\ 
        \Delta x & \Delta y & 0 & 0 \\ 
        0 & 0 & 0 & 1.0 \\ 
        -0.5\Delta x^2 & -\Delta x \Delta y & -0.5\Delta y^2 & 1.0 
        \end{bmatrix} 
        \begin{bmatrix} a \\ b \\ c \\ \ln(o) \end{bmatrix}
        $$
    *   Store $V_{\text{hoisted}}$ in shared memory.

3.  **Cooperative Staging:**
    All threads in the tile block cooperatively load the hoisted 6D parameter vectors and associated view-dependent colors from global memory into shared memory via coalesced transactions [S1, unofficial].

4.  **Warp-Level Culling:**
    Subdivide the $16 \times 16$ tile into $W_{\text{warp}} = 8$ independent $16 \times 2$ pixel sub-regions, mapping to a single 32-thread hardware warp.
    *   Determine the tight bounding box for the Gaussian within the warp's strip.
    *   The spatial footprint boundary under opacity threshold $\epsilon = 1/255$ is defined by the conic inequality $\tilde{x}^T \Sigma^{-1} \tilde{x} \le \tau$, where $\tau = 2 \ln(255 \cdot o)$ [S1, unofficial].
    *   If the warp's strip does not intersect the Gaussian's footprint, skip the entire warp to prevent wasted memory fetches and arithmetic [S1, unofficial].

5.  **Warp-Coherent Blending:**
    *   Compute the local basis vector for each pixel: $B_{\text{local}} = [x'^2, x'y', y'^2, x', y', 1]^T$.
    *   Compute alpha using the dot product: $\alpha = \exp(V_{\text{hoisted}}^T \cdot B_{\text{local}})$.
    *   Replace per-pixel branching with warp-uniform exit conditions.
    *   Restructure the blending loop into a dense FMA (Fused Multiply-Add) stream.
    *   Use a synchronization check with an early stop condition based on a threshold $T < 0.001$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Local-GS Kernel Execution for a Single Tile
# Input: Tile top-left (Tx, Ty), List of Gaussians covering tile
# Output: Alpha values for pixels in tile

def local_gs_tile_render(Tx, Ty, gaussians):
    # 1. Hoisting Stage: Precompute shared parameters
    hoisted_params = []
    for G in gaussians:
        # Calculate tile-relative offsets
        dx = G.mu_x - Tx
        dy = G.mu_y - Ty
        
        # Compute Hoisted Vector V_hoisted = [A, B, C, D, E, F]
        # Matrix multiplication from source [S1, unofficial]
        a, b, c = G.conic_a, G.conic_b, G.conic_c
        ln_o = log(G.opacity)
        
        A = -0.5 * a
        B = -1.0 * b
        C = -0.5 * c
        D = dx * a + dy * b + ln_o
        E = dx * b + dy * c + ln_o
        F = -0.5 * dx**2 * a - dx * dy * b - 0.5 * dy**2 * c + ln_o
        
        # Store as float4 (A,B,C,F) and float2 (D,E)
        hoisted_params.append((A, B, C, F, D, E))
        
    # 2. Warp-Level Culling and Blending
    # Tile is 16x16, subdivided into 8 warps of 16x2 pixels
    for warp_id in range(8):
        # Determine warp pixel range (y_start to y_end)
        y_start = warp_id * 2
        y_end = y_start + 2
        
        # Check if any Gaussian covers this warp strip
        # Using tight bounding box criterion: x_tilde^T Sigma^-1 x_tilde <= tau
        # tau = 2 * ln(255 * opacity)
        # If no intersection, skip this warp
        if not warp_covers_strip(gaussians, Tx, Ty, warp_id):
            continue
            
        # 3. Cooperative Blending
        for y in range(y_start, y_end):
            for x in range(0, 16):
                # Tile-local coordinates
                xp = x
                yp = y - y_start
                
                # Compute Local Basis B_local = [xp^2, xp*yp, yp^2, xp, yp, 1]
                # Note: Source uses x', y' as local coords. 
                # Assuming x' corresponds to x offset within tile, y' to y offset
                # In the loop, x is 0..15, y is relative to warp start
                
                # Calculate Alpha for each Gaussian
                alpha_sum = 0.0
                color_sum = 0.0
                alpha_weight_sum = 0.0
                
                for i, (A, B, C, F, D, E) in enumerate(hoisted_params):
                    # Basis vector components
                    b0 = xp * xp
                    b1 = xp * yp
                    b2 = yp * yp
                    b3 = xp
                    b4 = yp
                    b5 = 1.0
                    
                    # Dot product V_hoisted . B_local
                    log_alpha = A*b0 + B*b1 + C*b2 + D*b3 + E*b4 + F*b5
                    
                    # Alpha = exp(log_alpha)
                    alpha = exp(log_alpha)
                    
                    # Standard alpha blending
                    # Note: Implementation details for blending loop structure 
                    # (dense FMA stream) are implied by "Warp-Coherent Blending" [S1, unofficial]
                    # Placeholder for specific blending accumulation logic 
                    # as exact loop unrolling is hardware-dependent
                    alpha_sum += alpha
                    # ... accumulate color ...
                    
                    # Early stop condition check [S1, unofficial]
                    if alpha_sum > 0.999: # Threshold T < 0.001 implies remaining alpha is negligible
                        break
                
                # Final alpha and color assignment
                set_pixel_alpha(x, y, alpha_sum)
                set_pixel_color(x, y, color_sum)
```

### Parameters
*   **Tile Size:** $16 \times 16$ pixels [S1, unofficial].
*   **Warp Strip Size:** $16 \times 2$ pixels [S1, unofficial].
*   **Opacity Threshold ($\epsilon$):** $1/255$ for bounding box calculation [S1, unofficial].
*   **Coordinate Bounds:** Tile-local coordinates are bounded within $[-16, 32]$ [S1, unofficial].
*   **Early Stop Threshold:** $T < 0.001$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Catastrophic Cancellation:** In global coordinates (e.g., 4K resolution), $x$ and $\mu_x$ are around 4000. The absolute error bound $E_{\text{global}} \approx 4 \times 4000^2 \times 5.96 \times 10^{-8} \approx 3.81$, which overwhelms small signals (e.g., 0.04) [S1, unofficial]. The tile-local formulation bounds coordinates to $[-16, 32]$, reducing the error bound to $E_{\text{local}} \approx 4 \times 32^2 \times 5.96 \times 10^{-8} \approx 2.44 \times 10^{-4}$ [S1, unofficial].
*   **Register Pressure:** Registers per thread increase from 30 to 44 [S1, unofficial]. Implementation must ensure no register spilling occurs.

## Subtopics

### Data Compression and Storage Optimization
Focuses on reducing the memory footprint of Gaussian attributes. Local-GS is orthogonal to these methods and provides additional performance gains when combined [S1, unofficial].

### Hardware and Computational Acceleration
General techniques for GPU optimization. Local-GS is a specific kernel-level implementation within this family [S1, unofficial].

### Occupancy Region Derivation & $\epsilon$-Intersection Criterion
Provides the geometric basis for culling. Local-GS uses a tight bounding box derived from this criterion for warp-level culling [S1, unofficial].

### Gaussian Piling
A method for organizing Gaussians. Local-GS operates on the rendering kernel level and is compatible with various organization strategies [S1, unofficial].

### Parallel Differentiable Rasterization
Focuses on gradient computation. Local-GS optimizes the forward pass and is compatible with differentiable pipelines [S1, unofficial].

### CoherentRaster
Another rasterization optimization. Local-GS is orthogonal to CoherentRaster [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Significantly reduced SM active cycles. On the truck scene, SM active cycles drop by 64.0% (from 1,270,684 to 456,134) [S1, unofficial].
*   **Memory Complexity:** Registers per thread increase from 30 to 44 [S1, unofficial]. Total L2 sectors reduce by 72.0% (from 15,085,676 to 4,179,566) due to reduced global memory traffic [S1, unofficial].
*   **Measured Performance:**
    *   On Deep-Blending dataset with FlashGS on RTX 4080: 7.76x speedup with zero quality loss [S1, unofficial].
    *   On Tanks & Templates train scene: Kernel time reduces from 2.04 ms to 0.52 ms (PSNR 25.76 dB) [S1, unofficial].
    *   On bicycle scene (RTX 4080): Kernel time drops from 3.58 ms to 1.24 ms (2.89x speedup) [S1, unofficial].
    *   On A100: Speedy-Splat speedup reaches 2.00x at native resolution [S1, unofficial].
*   **Strengths:** Eliminates warp divergence, reduces branch instructions by 56.6%, and reduces divergent branches by 47.8% [S1, unofficial]. Orthogonal to culling, pruning, and compression methods [S1, unofficial].
*   **Limitations:** Increased register usage (30 to 44) [S1, unofficial]. Requires specific tile/warp subdivision logic.

## Background
Local-GS was introduced in the paper "Local-GS: Accelerating 3D Gaussian Splatting via Tile-Local Warp Coherence" [S1, unofficial]. It addresses microarchitectural limitations in 3DGS rendering, specifically warp divergence and redundant computation caused by irregular spatial distribution of Gaussians [S1, unofficial]. The method shifts optimization focus from macroscopic scene culling to microscopic GPU SIMT execution boundaries [S1, unofficial].

## Sources
[S1] Local-GS: Accelerating 3D Gaussian Splatting via Tile-Local Warp Coherence, https://doi.org/10.48550/arXiv.2606.16566, fetched 2026-10-29 (unofficial)
