# Bilateral Grid

Also called: Bilateral Space Grid

## Overview
The Bilateral Grid constructs a sparse 5D grid in spatial-luminance-chrominance space to facilitate efficient bilateral filtering via splatting and slicing [S1, unofficial]. It differs from standard bilateral filters by discretizing the feature space into a grid, allowing the filter to be implemented as a sequence of sparse matrix-vector multiplications, which is significantly faster for large kernels [S1, unofficial]. It builds on the Grid-based Implementation of Cross-Bilateral Filter [13144-grid-based-implementation-of-cross-bilateral-filter.md].

## How it works
**Inputs:** An input image $I$ (RGB), spatial sigma $\sigma_s$, luma sigma $\sigma_l$, and chroma sigma $\sigma_c$.
**Outputs:** A filtered image $I_{out}$, or a solution vector for the Bilateral Solver.

**Data Structures:**
1.  **Grid Coordinates:** A set of 5D coordinates for each pixel.
2.  **Splat Matrix ($S$):** A sparse matrix mapping pixel values to grid vertices.
3.  **Blur Matrices:** Sparse matrices representing 1D Gaussian-like blurring along each grid dimension.

**Steps:**

1.  **Color Conversion:** Convert the input RGB image $I$ to YUV color space.
    $$ I_{yuv} = \text{rgb2yuv}(I) $$
    Let $Y$ be the luma channel and $U, V$ be the chroma channels [S1, unofficial].

2.  **Coordinate Computation:** Compute 5D coordinates for each pixel $(x, y)$ in the image grid.
    *   Spatial coordinates:
        $$ x_{grid} = \frac{x}{\sigma_s}, \quad y_{grid} = \frac{y}{\sigma_s} $$
    *   Luma coordinate:
        $$ l_{grid} = \frac{Y}{\sigma_l} $$
    *   Chroma coordinates:
        $$ c1_{grid} = \frac{U}{\sigma_c}, \quad c2_{grid} = \frac{V}{\sigma_c} $$
    *   Stack into a 5D coordinate vector $p_i = [x_{grid}, y_{grid}, l_{grid}, c1_{grid}, c2_{grid}]$ for each pixel $i$ [S1, unofficial].

3.  **Grid Construction (Implicit):** The grid vertices are determined by the unique rounded coordinates of $p_i$. The grid is sparse, containing only vertices that are hit by at least one pixel.

4.  **Splat Matrix Construction ($S$):**
    *   Map each pixel $i$ to the nearest grid vertex $v_i$ by rounding its 5D coordinates.
    *   Construct a sparse matrix $S$ of size $N_v \times N_p$ (where $N_v$ is the number of vertices and $N_p$ is the number of pixels).
    *   $S_{v_i, i} = 1$ if vertex $v_i$ is the nearest neighbor to pixel $i$, otherwise 0.
    *   This is typically implemented as a Compressed Sparse Row (CSR) matrix [S1, unofficial].

5.  **Blur Matrix Construction:**
    *   For each of the 5 dimensions $d \in \{0, 1, 2, 3, 4\}$, construct a 1D blur matrix $B_d$.
    *   $B_d$ connects vertex $j$ to its neighbors $j \pm 1$ along dimension $d$.
    *   The weights are determined by a Gaussian kernel or similar smoothing function based on the distance in that dimension.
    *   The total blur operator $B$ is the product of these 1D matrices (or applied sequentially):
        $$ B = B_0 B_1 B_2 B_3 B_4 $$
    *   Valid neighbor indices are computed to ensure boundary conditions are handled correctly [S1, unofficial].

6.  **Filtering (Splat and Slice):**
    *   **Splat:** Project the input image values $x$ (flattened) onto the grid vertices:
        $$ y = S x $$
    *   **Blur:** Apply the blur operator to the grid values:
        $$ y_{blurred} = B y $$
    *   **Slice:** Project the blurred grid values back to the image pixels:
        $$ x_{out} = S^T y_{blurred} $$
    *   The result $x_{out}$ is reshaped back to the image dimensions [S1, unofficial].

**Parameters:**
*   $\sigma_s$: Spatial sigma for grid coordinate scaling. Default: 32 [S1, unofficial].
*   $\sigma_l$: Luma sigma for grid coordinate scaling. Default: 8 [S1, unofficial].
*   $\sigma_c$: Chroma sigma for grid coordinate scaling. Default: 8 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Coordinate rounding can introduce quantization artifacts if $\sigma_s$ is too small relative to image resolution.
*   **Boundaries:** Neighbor indices must be validated to avoid out-of-bounds access in the grid [S1, unofficial].
*   **Degenerate Input:** If the image has constant color, the chroma coordinates may be undefined or zero; standard YUV conversion handles this, but grid sparsity may increase.

## Subtopics

### Bilateral Solver
The Bilateral Solver uses the Bilateral Grid to solve for an output image that balances fidelity to the input and smoothness in bilateral space [S1, unofficial]. It formulates the filtering as a linear system solved via conjugate gradient optimization.

**Steps:**
1.  **Bistochastization:** Normalize the grid to improve conditioning. Compute diagonal matrices $D_n$ and $D_m$ such that the splat/slice operations are balanced [S1, unofficial].
2.  **Smoothness Matrix ($A_{smooth}$):** Construct the Laplacian-like matrix in bilateral space:
    $$ A_{smooth} = D_m - D_n S B S^T D_n $$
    (Note: The source describes this as $D_m - D_n \cdot \text{grid.blur}(D_n)$, implying the blur operator acts on the grid values scaled by $D_n$) [S1, unofficial].
3.  **Data Matrix ($A_{data}$):** Splat the weight image $w$ to get $w_{splat}$. Create a diagonal matrix from $w_{splat}$:
    $$ A_{data} = \text{diag}(w_{splat}) $$
4.  **Total Matrix ($A$):** Combine smoothness and data terms with a balance parameter $\lambda$:
    $$ A = \lambda A_{smooth} + A_{data} $$
5.  **Right-Hand Side ($b$):** Splat the weighted input image $x \cdot w$:
    $$ b = S (x \cdot w) $$
6.  **Preconditioner ($M$):** Construct a Jacobi preconditioner using the diagonal of $A$:
    $$ A_{diag} = \max(\text{diag}(A), A_{diag\_min}) $$
    $$ M = \text{diag}(1 / A_{diag}) $$
7.  **Initial Guess ($y_0$):**
    $$ y_0 = \frac{S(x \cdot w)}{w_{splat} + \epsilon} $$
8.  **Solve:** For each color channel, solve $A y = b$ using Preconditioned Conjugate Gradient (PCG) with preconditioner $M$, initial guess $y_0$, tolerance $cg\_tol$, and max iterations $cg\_maxiter$ [S1, unofficial].
9.  **Slice:** Project the solution $y_{hat}$ back to image space:
    $$ x_{out} = S^T y_{hat} $$

**Parameters:**
*   $\lambda$: Strength of the smoothness term. Default: 128 [S1, unofficial].
*   $A_{diag\_min}$: Minimum value for the diagonal of $A$ in the preconditioner. Default: $10^{-5}$ [S1, unofficial].
*   $cg\_tol$: Tolerance for PCG convergence. Default: $10^{-5}$ [S1, unofficial].
*   $cg\_maxiter$: Maximum number of PCG iterations. Default: 25 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The splat and slice operations are $O(N_p)$ where $N_p$ is the number of pixels. The blur operation depends on the grid size $N_v$ and the number of non-zero entries in the blur matrices, which is proportional to $N_v$. For the Bilateral Solver, each PCG iteration is $O(N_v)$, so the total time is $O(k \cdot N_v)$ where $k$ is the number of iterations.
*   **Memory Complexity:** $O(N_v + N_p)$ for storing the grid vertices, splat matrix, and blur matrices.
*   **Performance:** Significantly faster than direct bilateral filtering for large kernels because the kernel size is effectively decoupled from the spatial resolution. The grid size is determined by the sigmas, not the image size.
*   **Strengths:** Efficient implementation of bilateral filtering; enables differentiable operations (Bilateral Solver); handles large kernels efficiently.
*   **Limitations:** Quantization artifacts if grid resolution is too low; requires careful tuning of sigmas; the Bilateral Solver requires solving a large linear system, which can be memory-intensive for very high-resolution images.
*   **When to use:** When real-time or near-real-time bilateral filtering is required, or when differentiable filtering is needed for optimization tasks.
*   **When not to use:** For very small kernels where direct filtering is faster; when high precision is required and grid quantization artifacts are unacceptable.

## Background
The Bilateral Grid was introduced as a method for efficient bilateral filtering by [S1, unofficial]. It is implemented in the `weiyithu/NerfingMVS` repository, which uses it for image processing tasks within the context of NeRF and MVS pipelines [S1, unofficial]. The method leverages the separability of the bilateral filter in a higher-dimensional space to reduce computational cost.

## Sources
[S1] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
