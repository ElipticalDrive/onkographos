# Tiled Matrix Multiplication with Shared Memory


## Overview
A GPU kernel that optimizes matrix multiplication $C = A \times B$ by tiling data into block-private shared memory to reduce global memory bandwidth pressure [S1, unofficial]. It differs from the naive kernel by loading each tile once into shared memory and reusing it for $TILE\_WIDTH$ multiplications, whereas the naive kernel accesses global memory for every element in the inner loop [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Matrix $A$ of size $M \times N$ in global memory.
- Matrix $B$ of size $N \times P$ in global memory.
- Matrix $C$ of size $M \times P$ in global memory (output).
- `TILE_WIDTH`: The dimension of the square tile loaded into shared memory. Default value is 32 [S1, unofficial].

**Data Structures:**
- `__shared__ float As[TILE_WIDTH][TILE_WIDTH]`: Block-private array storing a tile of matrix $A$ [S1, unofficial].
- `__shared__ float Bs[TILE_WIDTH][TILE_WIDTH]`: Block-private array storing a tile of matrix $B$ [S1, unofficial].

**Algorithm:**
1. Calculate global row and column indices for the output matrix $C$ using block and thread indices:
   $$row = blockIdx.y \times TILE\_WIDTH + ty$$
   $$col = blockIdx.x \times TILE\_WIDTH + tx$$
   where $tx = threadIdx.x$ and $ty = threadIdx.y$ [S1, unofficial].

2. Initialize accumulator:
   $$accumulate = 0.0f$$ [S1, unofficial].

3. Calculate total number of tiles along the reduction dimension $N$:
   $$total\_tiles = \lceil N / TILE\_WIDTH \rceil = (N + TILE\_WIDTH - 1) / TILE\_WIDTH$$
   using integer division rounding up [S1, unofficial].

4. Loop over tiles:
   $$for \ (int \ tile\_idx = 0; \ tile\_idx < total\_tiles; \ tile\_idx++)$$ [S1, unofficial].

5. Load tile of $A$ into shared memory:
   Each thread $(tx, ty)$ loads one element from global memory $A$ into shared memory $As$. The global index for $A$ is $(row, tile\_idx \times TILE\_WIDTH + tx)$.
   - If $row < M$ and $tile\_idx \times TILE\_WIDTH + tx < N$, load $A[row][tile\_idx \times TILE\_WIDTH + tx]$ into $As[ty][tx]$.
   - Otherwise, write $0.0f$ to $As[ty][tx]$ to prevent garbage values from corrupting the dot product accumulation [S1, unofficial].

6. Load tile of $B$ into shared memory:
   Each thread $(tx, ty)$ loads one element from global memory $B$ into shared memory $Bs$. The global index for $B$ is $(tile\_idx \times TILE\_WIDTH + ty, col)$.
   - If $tile\_idx \times TILE\_WIDTH + ty < N$ and $col < P$, load $B[tile\_idx \times TILE\_WIDTH + ty][col]$ into $Bs[ty][tx]$.
   - Otherwise, write $0.0f$ to $Bs[ty][tx]$ [S1, unofficial].

7. Synchronize threads within the block to ensure all shared memory loads are complete:
   $$\_\_syncthreads()$$ [S1, unofficial].

8. Compute partial dot product for the current tile:
   Loop $k$ from $0$ to $TILE\_WIDTH - 1$:
   $$accumulate += As[ty][k] \times Bs[k][tx]$$ [S1, unofficial].

9. Synchronize threads again before loading the next tile:
   $$\_\_syncthreads()$$ [S1, unofficial].

10. Write final result to global memory:
    If $row < M$ and $col < P$, write $accumulate$ to $d\_c[row \times P + col]$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumes TILE_WIDTH is a compile-time constant or passed as a parameter
# Assumes A, B, C are 1D arrays representing MxN, NxP, MxP matrices in row-major order
# Assumes global memory pointers d_A, d_B, d_C

__shared__ float As[TILE_WIDTH][TILE_WIDTH]
__shared__ float Bs[TILE_WIDTH][TILE_WIDTH]

# Thread indices
tx = threadIdx.x
ty = threadIdx.y

# Block indices
bx = blockIdx.x
by = blockIdx.y

# Global row and column for this thread's output element
row = by * TILE_WIDTH + ty
col = bx * TILE_WIDTH + tx

# Accumulator for the dot product
accumulate = 0.0f

# Total number of tiles along the reduction dimension N
total_tiles = (N + TILE_WIDTH - 1) // TILE_WIDTH

for tile_idx in range(total_tiles):
    # Load tile of A
    a_col = tile_idx * TILE_WIDTH + tx
    if row < M and a_col < N:
        As[ty][tx] = d_A[row * N + a_col]
    else:
        As[ty][tx] = 0.0f

    # Load tile of B
    b_row = tile_idx * TILE_WIDTH + ty
    if b_row < N and col < P:
        Bs[ty][tx] = d_B[b_row * P + col]
    else:
        Bs[ty][tx] = 0.0f

    # Synchronize to ensure all threads have loaded their data
    __syncthreads()

    # Compute partial dot product for the current tile
    for k in range(TILE_WIDTH):
        accumulate += As[ty][k] * Bs[k][tx]

    # Synchronize before loading the next tile
    __syncthreads()

# Write result to global memory
if row < M and col < P:
    d_C[row * P + col] = accumulate
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** If matrix dimensions $M$, $N$, or $P$ are not multiples of `TILE_WIDTH`, threads accessing out-of-bounds global memory indices must write `0.0f` to shared memory to prevent garbage values from corrupting the dot product accumulation [S1, unofficial].
- **Precision:** The use of `float` (single precision) is standard, but accumulation order can affect precision in large tiles. Not given in the sources.
- **Synchronization:** Failure to call `__syncthreads()` after loading shared memory and before computing the dot product, or after computing and before loading the next tile, will result in race conditions and incorrect results [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(M \times P \times N)$, same as naive matrix multiplication, but with reduced memory access overhead.
- **Memory Complexity:** $O(TILE\_WIDTH^2)$ per block for shared memory, plus $O(M \times N + N \times P + M \times P)$ for global memory.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales with the number of blocks and threads, limited by shared memory capacity and occupancy. Larger `TILE_WIDTH` reduces global memory accesses but increases shared memory usage and may reduce occupancy.
- **Strengths:** Significantly reduces global memory bandwidth pressure by reusing data in shared memory [S1, unofficial].
- **Limitations:** Requires shared memory, which is limited on GPUs. Not suitable for very large tiles or matrices that exceed shared memory capacity. Boundary handling adds conditional logic.
- **When to Use:** When matrix dimensions are large enough to benefit from shared memory reuse and fit within the tile size. When global memory bandwidth is a bottleneck.
- **When Not to Use:** When matrices are small, or when shared memory is constrained. When dimensions are not multiples of `TILE_WIDTH` and boundary handling overhead is significant.

## Background
- **Origin:** The tiled matrix multiplication with shared memory is a standard optimization technique in GPU programming, often attributed to early CUDA programming guides and textbooks. The specific implementation details and code structure are found in [S1, unofficial].
- **History:** Developed as part of the broader effort to optimize GPU performance by leveraging the memory hierarchy.
- **Usage:** Widely used in scientific computing, machine learning, and graphics applications where matrix multiplication is a core operation.

## Sources
[S1] wyh1309/hpc_foundational_demos, https://github.com/wyh1309/hpc_foundational_demos, fetched 2026-10-26 (unofficial)
