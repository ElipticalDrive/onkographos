# Sharded Gradient Computation


## Overview
Sharded Gradient Computation computes the spatial gradients ($g_x, g_y, g_z$) of a volumetric dataset partitioned into shards across multiple devices (GPUs or CPUs), specifically handling boundary conditions between shards in the z-dimension [S1, unofficial]. It differs from standard gradient methods by distributing the memory and computation load of large volumes across a cluster, requiring explicit inter-shard communication for gradients that cross shard boundaries [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume $V$ partitioned into $N$ shards along the z-dimension. Each shard $i$ contains a 3D array $A_i$ of shape $(H_i, W, D)$, where $H_i$ is the height (z-dimension size) of the shard, $W$ is the width (y-dimension), and $D$ is the depth (x-dimension). Note: The source code uses `array[..., :-1]` for x-gradient, implying the last dimension is x, middle is y, first is z. Let's align with the source's indexing: `array[z, y, x]`.
- Output: Three gradient shards $G_x, G_y, G_z$, each matching the input volume structure (same number of shards and local shapes).

**Data Structures:**
- `shards`: A list of arrays, where `shards[i]` is the data for shard $i$.
- `xp`: The array library (e.g., NumPy, CuPy) used for computation.
- `device`: The current device context (GPU/CPU).

**Steps:**
1. Initialize output gradient shards $G_x, G_y, G_z$ with zeros, matching the shape of each input shard.
2. Iterate over each shard index $i$ from $0$ to $N-1$:
   a. Let $A$ be `shards[i]`.
   b. Compute $G_x$ for the current shard using forward differences along the last dimension (x-axis):
      $$ G_x[i, :, :, :-1] = A[:, :, 1:] - A[:, :, :-1] $$
      (Note: The source uses `gx_arr[..., :-1] = array[..., 1:] - array[..., :-1]`. This implies the gradient is defined on the edges or shifted. Standard forward difference for derivative at node $k$ is $V_{k+1} - V_k$. The source assigns to `:-1`, suggesting the gradient array might be one element smaller in the x-dimension, or it's a specific implementation detail. We follow the source exactly.)
   c. Compute $G_y$ for the current shard using forward differences along the middle dimension (y-axis):
      $$ G_y[i, :, :-1, :] = A[:, 1:, :] - A[:, :-1, :] $$
   d. Compute $G_z$ for the internal rows of the current shard using forward differences along the first dimension (z-axis):
      If $A.shape[0] > 1$:
      $$ G_z[i, :-1, :, :] = A[1:, :, :] - A[:-1, :, :] $$
   e. Handle the boundary between shard $i$ and shard $i+1$ for the $G_z$ gradient:
      If $i + 1 < N$:
      i. Switch device context to the device holding shard $i+1$.
      ii. Extract the first row (z-index 0) of shard $i+1$: $A_{next\_first} = \text{shards}[i+1][0, :, :]$.
      iii. Compute the last row of the current shard's $G_z$ using the first row of the next shard:
          $$ G_z[i, -1, :, :] = A_{next\_first} - A[-1, :, :] $$
      iv. Switch device context back to the current shard's device.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sharded_gradients(shards, xp, devices):
    """
    shards: list of 3D arrays [z, y, x]
    xp: array library (e.g., numpy, cupy)
    devices: list of device contexts corresponding to each shard
    """
    num_shards = len(shards)
    # Initialize output gradient shards with zeros
    gx_shards = [xp.zeros_like(s) for s in shards]
    gy_shards = [xp.zeros_like(s) for s in shards]
    gz_shards = [xp.zeros_like(s) for s in shards]

    for i in range(num_shards):
        A = shards[i]
        gx = gx_shards[i]
        gy = gy_shards[i]
        gz = gz_shards[i]

        # Compute gx (x-direction) using forward differences on the last dimension
        # Source: gx_arr[..., :-1] = array[..., 1:] - array[..., :-1]
        # This reduces the x-dimension size by 1.
        if A.shape[2] > 1:
            gx[..., :-1] = A[..., 1:] - A[..., :-1]

        # Compute gy (y-direction) using forward differences on the middle dimension
        # Source: gy_arr[:, :-1, :] = array[:, 1:, :] - array[:, :-1, :]
        # This reduces the y-dimension size by 1.
        if A.shape[1] > 1:
            gy[:, :-1, :] = A[:, 1:, :] - A[:, :-1, :]

        # Compute gz (z-direction) for internal rows
        # Source: gz_arr[:-1, :, :] = array[1:, :, :] - array[:-1, :, :]
        # This reduces the z-dimension size by 1 for internal rows.
        if A.shape[0] > 1:
            gz[:-1, :, :] = A[1:, :, :] - A[:-1, :, :]

        # Handle boundary between current shard and next shard for gz
        if i + 1 < num_shards:
            # Switch device context to the next shard's device
            with devices[i + 1]:
                # Extract the first row (z=0) of the next shard
                next_first = shards[i + 1][0, :, :]

            # Compute the last row of the current shard's gz gradient
            # Source: gz_arr[-1, :, :] = next_first - array[-1, :, :]
            # Note: This requires accessing the next shard's data. 
            # If the next shard is on a different device, we must have copied 
            # next_first to the current device's memory or use a unified memory system.
            # The source implies `next_first` is available on the current device 
            # after switching context or via unified memory.
            # We assume `next_first` is on the current device for the subtraction.
            gz[-1, :, :] = next_first - A[-1, :, :]

    return gx_shards, gy_shards, gz_shards
```

**Parameters:**
- Shard count: Configurable via `config.zsharded_admm.shard_count` [S1, unofficial].
- Device context: Managed via `devices` list [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Inter-shard communication may introduce precision issues if data types differ or if synchronization is not handled correctly [G].
- Boundaries: The method assumes a contiguous volume. If shards are not contiguous in z, the gradient calculation is invalid [G].
- Degenerate Input: If a shard has size 1 in any dimension, the corresponding gradient component for that dimension will be all zeros (or undefined if not handled) [G].

## Subtopics

### [Neumann Gradient](3342-neumann-gradient.md)
Uses zero-gradient boundary conditions. Choose when smooth boundaries are expected.

### [Forward Difference Gradient](3362-forward-difference-gradient.md)
Simple first-order approximation. Choose for speed and simplicity.

### [4th-Order Finite Difference Gradient](4889-4th-order-finite-difference-gradient.md)
Higher accuracy. Choose when precision is critical.

### [Gradient Selection](5889-gradient-selection.md)
Selects optimal gradient method. Choose for adaptive accuracy.

### [Gauss-Seidel with Successive Over-Relaxation (SOR)](6904-gauss-seidel-with-successive-over-relaxation-sor.md)
Iterative solver. Choose for solving Poisson equations.

### [Brick-local Gradient Estimation](8810-brick-local-gradient-estimation.md)
Local brick-based computation. Choose for hierarchical data.

### [Sobel-based Gradient Estimation](9967-sobel-based-gradient-estimation.md)
Edge detection. Choose for image processing.

### [Central Difference Stencils](9968-central-difference-stencils.md)
Second-order accuracy. Choose for balanced accuracy and cost.

## Operating characteristics
- Time Complexity: $O(N)$ where $N$ is the total number of voxels, with a constant factor for inter-shard communication [S1, unofficial].
- Memory Complexity: $O(N)$ for storing the volume and gradients, plus overhead for shard management [S1, unofficial].
- Measured Performance: Not given in the sources.
- Scaling: Scales with the number of shards and devices [S1, unofficial].
- Strengths: Enables processing of volumes larger than a single device's memory [S1, unofficial].
- Limitations: Requires inter-shard communication, which can be a bottleneck [S1, unofficial].
- When to use: When the volume is too large for a single device [S1, unofficial].
- When not to use: When the volume fits in a single device, as the overhead may not be justified [G].

## Background
Origin: Introduced in the TomoAnchor-V3 project by alidarbandi [S1, unofficial].
History: Developed to support large-scale tomographic reconstruction on GPU clusters [S1, unofficial].
Usage: Used in tomographic reconstruction pipelines [S1, unofficial].

## Sources
[S1] alidarbandi/TomoAnchor-V3, https://github.com/alidarbandi/TomoAnchor-V3, fetched 2026-10-27 (unofficial)
