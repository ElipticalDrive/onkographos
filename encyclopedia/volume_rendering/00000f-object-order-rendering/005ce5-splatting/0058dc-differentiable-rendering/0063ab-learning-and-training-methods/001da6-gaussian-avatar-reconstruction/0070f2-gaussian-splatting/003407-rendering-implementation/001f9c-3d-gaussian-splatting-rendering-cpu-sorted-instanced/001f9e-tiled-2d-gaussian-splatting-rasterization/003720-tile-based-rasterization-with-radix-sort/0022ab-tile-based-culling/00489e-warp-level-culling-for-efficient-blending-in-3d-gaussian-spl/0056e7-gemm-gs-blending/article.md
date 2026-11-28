# GEMM-GS Blending

Also called: GEMM-compatible Blending, Intra-Tile Coordinate Transformation

## Overview
GEMM-GS is a rendering optimization for 3D Gaussian Splatting (3DGS) that reformulates the opacity computation into a General Matrix Multiplication (GEMM) compatible dot-product form to leverage GPU Tensor Cores [S1]. It addresses the low compute efficiency of standard element-wise quadratic evaluations in the blending stage by exploiting intra-tile relative coordinates to transform quadratic power terms into linear dot products [S1]. Unlike [Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting](12278-warp-level-culling-for-efficient-blending-in-3d-gaussian-spl.md), which focuses on geometric culling to reduce the number of blending operations, GEMM-GS accelerates the remaining blending computations themselves [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of sorted 3D Gaussians overlapping with the current image tile; tile dimensions (typically 16×16 pixels); batch size of Gaussians (typically 256).
*   **Output:** Rendered color values for all pixels in the tile.

**Data Structures:**
*   **Reference Pixel ($p_c$):** A fixed pixel within the tile, typically the center, with coordinates $(x_c, y_c)$ [S1].
*   **Intra-tile Relative Coordinates ($\bar{x}_{p_j}, \bar{y}_{p_j}$):** For any pixel $p_j$ in the tile, defined such that $(x_{p_j}, y_{p_j}) = (x_c - \bar{x}_{p_j}, y_c - \bar{y}_{p_j})$ [S1]. These are constant for all tiles and precomputed offline [S1].
*   **Gaussian Offset ($\hat{x}_{g_i}, \hat{y}_{g_i}$):** The difference between the Gaussian's projected center and the reference pixel: $\hat{x}_{g_i} = x_{g_i} - x_c$, $\hat{y}_{g_i} = y_{g_i} - y_c$ [S1].
*   **Gaussian Vector ($\hat{v}_{g_i}$):** A 6-dimensional vector per Gaussian [S1].
*   **Pixel Vector ($\hat{v}_{p_j}$):** A 6-dimensional vector per pixel [S1].
*   **Matrix $M_g$:** Dimensions $B \times 6$, where $B$ is the batch size (e.g., 256). Rows are $\hat{v}_{g_i}^T$ [S1].
*   **Matrix $M_p$:** Dimensions $P \times 6$, where $P$ is the number of pixels in the tile (e.g., 256). Rows are $\hat{v}_{p_j}^T$ [S1]. This matrix is identical across all tiles and stored in global memory for reuse [S1].
*   **Matrix $M_{power}$:** Dimensions $B \times P$. Result of $M_g \cdot M_p$ [S1].

**Formulas:**
The opacity $\alpha_{ij}$ for Gaussian $g_i$ at pixel $p_j$ is:
$$ \alpha_{ij} = o_i \times \exp(-\frac{1}{2} \mathbf{x}_{g_i}^T \Sigma_i^{-1} \mathbf{x}_{g_i}) $$
where $\mathbf{x}_{g_i} = [\Delta x_{ij} \ \Delta y_{ij}]^T$ is the vector from the Gaussian center to the pixel, and $\Sigma_i^{-1} = \begin{bmatrix} A_i & B_i \\ B_i & C_i \end{bmatrix}$ is the inverse covariance matrix [S1].

The exponent term (power) is expanded as:
$$ power_{ij} = -\frac{1}{2} A_i \Delta x_{ij}^2 - B_i \Delta x_{ij} \Delta y_{ij} - \frac{1}{2} C_i \Delta y_{ij}^2 $$
[ S1 ]

By substituting $\Delta x_{ij} = \hat{x}_{g_i} + \bar{x}_{p_j}$ and $\Delta y_{ij} = \hat{y}_{g_i} + \bar{y}_{p_j}$, the power term is transformed into a dot product:
$$ power_{ij} = \hat{v}_{g_i} \cdot \hat{v}_{p_j} $$
[ S1 ]

The components of $\hat{v}_{g_i}$ are:
$$ \hat{v}_{g_i} = \left[ -\frac{1}{2}A_i, -\frac{1}{2}C_i, -B_i, -A_i\hat{x}_{g_i} - B_i\hat{y}_{g_i}, -C_i\hat{y}_{g_i} - B_i\hat{x}_{g_i}, -\frac{1}{2}A_i\hat{x}_{g_i}^2 - \frac{1}{2}C_i\hat{y}_{g_i}^2 - B_i\hat{x}_{g_i}\hat{y}_{g_i} \right] $$
[ S1 ]

The components of $\hat{v}_{p_j}$ are derived from the expansion of the quadratic form with respect to $\bar{x}_{p_j}$ and $\bar{y}_{p_j}$:
$$ \hat{v}_{p_j} = \left[ -\frac{1}{2}\bar{x}_{p_j}^2, -\frac{1}{2}\bar{y}_{p_j}^2, -\bar{x}_{p_j}\bar{y}_{p_j}, \bar{x}_{p_j}, \bar{y}_{p_j}, 1 \right]^T $$
*Note: The source [S1] provides the matrix multiplication structure $M_{power} = M_g \cdot M_p$ and the explicit $\hat{v}_{g_i}$ vector. The $\hat{v}_{p_j}$ vector is derived from the factorization shown in [S1] where the pixel-dependent terms are isolated. Specifically, the expansion in [S1] shows the pixel terms as $[-\frac{1}{2}\bar{x}_{p_j}^2, -\frac{1}{2}\bar{y}_{p_j}^2, -\bar{x}_{p_j}\bar{y}_{p_j}, \bar{x}_{p_j}, \bar{y}_{p_j}, 1]^T$ matching the Gaussian coefficients.*

**Parameters:**
*   Tile size: 16×16 pixels (256 pixels total) [S1].
*   Batch size: 256 Gaussians [S1].
*   Thread block size: 256 threads (8 warps) [S1].

**Steps:**
1.  **Precomputation:** Offline, compute $\hat{v}_{p_j}$ for all 256 pixels in a 16×16 tile relative to the center reference pixel. Store this as matrix $M_p$ in global memory [S1].
2.  **Stage 1 (Load Indices):** Load Gaussian indices for the current batch into shared memory. Use double buffering (`IBuffer`) to overlap data transfer with computation [S1].
3.  **Stage 2 (Fetch Attributes):** Fetch Gaussian attributes (projected coordinates, covariance parameters $A_i, B_i, C_i$, opacity $o_i$, color) from global memory. Compute $\hat{x}_{g_i}, \hat{y}_{g_i}$ and construct $\hat{v}_{g_i}$ for each Gaussian in the batch. Store the resulting $M_g$ matrix in shared memory using double buffering (`MBuffer`) to prevent synchronization stalls [S1].
4.  **Stage 3 (GEMM and Rendering):**
    *   Perform matrix multiplication $M_{power} = M_g \cdot M_p$ [S1].
    *   Due to limited shared memory, use a mini-batch design. Multiply a 16×8 submatrix of $M_g$ with $M_p$ [S1].
    *   Execute this via four rounds of warp-level `mma(m16n8k8)` PTX instructions. A thread block of 8 warps makes 32 warp-level calls to complete an effective m256n16k8 multiplication [S1].
    *   Compute opacity: $\alpha_{ij} = o_i \times \exp(M_{power}[i][j])$ [S1].
    *   Apply $\alpha$-skipping if $\alpha_{ij} \le 1/255$ [S1].
    *   Perform alpha blending accumulation with early termination if transmittance $T_j \le 0$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed global constant for a 16x16 tile centered at (0,0) relative coords
# M_p_global is 256x6, precomputed offline
M_p_global = compute_pixel_vectors()

def gemm_gs_blend(Gaussian_List, Tile_Ref_Center):
    """
    Gaussian_List: List of sorted Gaussians overlapping the tile.
    Tile_Ref_Center: (x_c, y_c) of the reference pixel (e.g., tile center).
    """
    batch_size = 256
    tile_pixels = 256
    
    # Shared memory buffers (double buffered)
    # I_Buffer: indices of Gaussians
    # F_Buffer: Gaussian attributes (A, B, C, x_g, y_g, o, color)
    # M_Buffer: M_g matrix (batch_size x 6)
    
    # Stage 1: Load Indices
    # Load batch_size indices from Gaussian_List into I_Buffer
    # Use async copy to overlap with next stage
    
    # Stage 2: Fetch Attributes and Construct M_g
    # For each Gaussian i in batch:
    #   Fetch A_i, B_i, C_i, x_g_i, y_g_i, o_i, color_i
    #   Compute offsets: x_hat = x_g_i - x_c, y_hat = y_g_i - y_c
    #   Compute v_hat_g_i vector:
    #     v_hat_g_i[0] = -0.5 * A_i
    #     v_hat_g_i[1] = -0.5 * C_i
    #     v_hat_g_i[2] = -B_i
    #     v_hat_g_i[3] = -A_i * x_hat - B_i * y_hat
    #     v_hat_g_i[4] = -C_i * y_hat - B_i * x_hat
    #     v_hat_g_i[5] = -0.5 * A_i * x_hat**2 - 0.5 * C_i * y_hat**2 - B_i * x_hat * y_hat
    #   Store v_hat_g_i in M_Buffer[i]
    
    # Stage 3: GEMM and Rendering
    # M_p is loaded from global memory (shared across tiles)
    
    # Mini-batch GEMM: M_g (256x6) * M_p (256x6)^T -> M_power (256x256)
    # Note: M_p in source is 256x6, so M_g * M_p^T or M_g * M_p depending on layout.
    # Source says M_g is 256x6, M_p is 256x6, M_power is 256x256.
    # This implies M_power[i][j] = dot(M_g[i], M_p[j]).
    # So we need M_g * M_p^T if using standard GEMM, or specific layout.
    # Source Algorithm 2: M_power <- M_g * M_p. 
    # If M_g is (Bx6) and M_p is (Px6), standard GEMM (Bx6) * (6xP) is needed.
    # So M_p must be transposed or stored as (6xP). 
    # Source text: "M_p is 256x6... M_power is 256x256... M_g * M_p".
    # This implies M_p is treated as (6x256) in the multiplication or the notation 
    # implies row-wise dot products. Given "rows are v_hat_p_j", M_p is Px6.
    # To get BxP result from Bx6 and Px6, we compute M_g * M_p^T.
    
    M_power = matmul(M_g, transpose(M_p_global))
    
    # Volume Rendering
    for i in range(batch_size):
        o_i = Gaussian_List[i].opacity
        color_i = Gaussian_List[i].color
        for j in range(tile_pixels):
            power = M_power[i][j]
            alpha = o_i * exp(power)
            
            # Alpha skipping
            if alpha <= 1/255:
                continue
                
            # Accumulate color
            # T_j is transmittance of pixel j
            # C_j = C_j + alpha * T_j * color_i
            # T_j = T_j * (1 - alpha)
            
            # Early termination if T_j <= 0
            if T_j <= 0:
                break
```

**Edge Cases and Pitfalls:**
*   **Shared Memory Limits:** The mini-batch design (16x8 submatrices) is required due to limited shared memory [S1].
*   **Precision:** The transformation relies on exact arithmetic for the dot product decomposition; floating-point errors in coordinate differences may affect accuracy, though not explicitly detailed in sources [S1].
*   **Boundary Conditions:** The method assumes a fixed tile size (16x16). Tiles at image boundaries may require padding or special handling not detailed in the source [S1].

## Operating characteristics
*   **Time Complexity:** The core blending operation is reduced to GEMM, which is highly optimized on Tensor Cores. The complexity is dominated by the matrix multiplication $O(B \cdot P \cdot 6)$ per batch, where $B$ is batch size and $P$ is pixels per tile [S1].
*   **Memory Complexity:** Requires storage for $M_p$ (256x6 floats) in global memory, plus shared memory for double-buffered indices, features, and $M_g$ [S1].
*   **Measured Performance:** On NVIDIA A100, GEMM-GS achieves a 1.42× speedup over vanilla 3DGS. On H100, it achieves a 1.37× speedup [S1]. It provides an average 1.19× speedup over FlashGS, 1.42× over StopThePop, and 1.50× over SpeedySplat on A100 [S1].
*   **Strengths:** High utilization of GPU Tensor Cores; efficient memory hierarchy usage via double buffering and warp-level synchronization [S1].
*   **Limitations:** Requires fixed tile and batch sizes for optimal performance; relies on specific PTX instructions for warp-level GEMM [S1].
*   **When to use:** When rendering 3DGS on modern NVIDIA GPUs with Tensor Cores and high throughput is required [S1].
*   **When not to use:** On hardware without Tensor Cores or warp-level MMA support [S1].

## Background
GEMM-GS was introduced in the paper "GEMM-GS: Accelerating 3D Gaussian Splatting on Tensor Cores with GEMM-Compatible Blending" by Haomin Li, Bowen Zhu, Fangxin Liu, et al., published in DAC ’26 (July 26–29, 2026, Long Beach, CA, USA) [S1]. The work addresses the computational bottleneck of the blending stage in 3DGS, which traditionally uses element-wise quadratic calculations that do not map efficiently to GPU hardware [S1].

## Sources
[S1] GEMM-GS: Accelerating 3D Gaussian Splatting on Tensor Cores with GEMM-Compatible Blending, https://doi.org/10.1145/3770743.3803899, fetched 2026-10-28
