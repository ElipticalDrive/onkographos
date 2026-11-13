# Sharded Gradient Computation


## Overview
Sharded Gradient Computation computes the spatial gradients $(G_x, G_y, G_z)$ of a volume partitioned into shards across multiple devices (GPUs/CPU), specifically handling boundary conditions between shards for the z-dimension [S1, unofficial]. It builds on [Finite Difference Stencils (Central/Forward/Backward)](3858-finite-difference-stencils-central-forward-backward.md) by applying forward differences within local shards and explicitly bridging inter-shard boundaries for the primary axis of partitioning.

## How it works
**Inputs and Outputs:**
- Input: A volume $V$ partitioned into $N$ shards along the z-axis. Each shard $S_i$ is a 3D array with shape $(H_i, W, D)$, where $H_i$ is the height (z-dimension size) of shard $i$, $W$ is the width (y-dimension), and $D$ is the depth (x-dimension). Note: The source implementation uses a specific memory layout where the first dimension corresponds to the shard axis (z), the second to y, and the third to x.
- Output: Three gradient shards $G_x, G_y, G_z$, each matching the input volume structure.

**Data Structures:**
- `shards`: A list or array of device-specific arrays. `shards[i]` holds the data for the $i$-th shard.
- `xp`: A device-agnostic array manipulation module (e.g., `cupy` for GPU, `numpy` for CPU) that handles memory management and array operations on the current device context.

**Numbered Steps:**
1.  **Initialization:** Initialize output gradient shards $G_x, G_y, G_z$ with zeros, matching the shape of the input volume shards.
2.  **Iteration:** Iterate over each shard index $i$ from $0$ to $N-1$.
3.  **Device Context:** Set the active device context to the device holding shard $S_i$.
4.  **X-Gradient ($G_x$):** Compute the gradient along the x-axis (last dimension of the shard array) using forward differences:
    $$ G_{x,i}[:, :, :-1] = S_i[:, :, 1:] - S_i[:, :, :-1] $$
    This computes the difference between adjacent voxels along the x-axis. The last column of $G_{x,i}$ remains zero (or undefined, depending on boundary handling requirements, but the source sets it via slicing which leaves it as initialized zero).
5.  **Y-Gradient ($G_y$):** Compute the gradient along the y-axis (middle dimension of the shard array) using forward differences:
    $$ G_{y,i}[:, :-1, :] = S_i[:, 1:, :] - S_i[:, :-1, :] $$
    This computes the difference between adjacent voxels along the y-axis. The last row of $G_{y,i}$ remains zero.
6.  **Z-Gradient ($G_z$) Internal:** Compute the gradient along the z-axis (first dimension of the shard array) for internal rows. If the shard height $H_i > 1$:
    $$ G_{z,i}[:-1, :, :] = S_i[1:, :, :] - S_i[:-1, :, :] $$
    This computes the difference between adjacent voxels along the z-axis within the shard. The last row of $G_{z,i}$ is not yet computed.
7.  **Z-Gradient ($G_z$) Boundary:** Handle the boundary between shard $i$ and shard $i+1$. If $i + 1 < N$:
    a. Switch device context to the device holding shard $S_{i+1}$.
    b. Extract the first row (lowest z-index) of shard $S_{i+1}$:
       $$ \text{next\_first} = S_{i+1}[0, :, :] $$
    c. Switch device context back to the device holding shard $S_i$.
    d. Compute the last row of $G_{z,i}$ using the first row of the next shard:
       $$ G_{z,i}[-1, :, :] = \text{next\_first} - S_i[-1, :, :] $$
    This ensures continuity of the gradient across the shard boundary.
8.  **Final Shard Boundary:** If $i + 1 == N$ (last shard), the last row of $G_{z,i}$ remains zero (or requires a separate boundary condition not specified in the core algorithm, typically zero-padding or clamping).

**Formulas:**
The forward difference approximation for the partial derivatives is:
$$ G_x = V(x+1, y, z) - V(x, y, z) $$
$$ G_y = V(x, y+1, z) - V(x, y, z) $$
$$ G_z = V(x, y, z+1) - V(x, y, z) $$
Note: The source implementation does not divide by voxel spacing $h$. It assumes $h=1$ or returns unnormalized differences. If physical units are required, division by $h$ must be applied post-computation.

**Parameters:**
- `shard_count`: Number of shards along the z-axis. Can be "auto" or an integer [S1, unofficial].
- `device_ids`: List of device IDs for each shard.

**Edge Cases and Pitfalls:**
- **Single Voxel Shards:** If a shard has height $H_i = 1$, the internal z-gradient step is skipped. The boundary step must still handle the connection to the next shard if it exists.
- **Device Switching:** Switching device contexts (e.g., GPU to GPU) incurs overhead. Minimizing the number of switches is critical for performance.
- **Memory Coherence:** When accessing `next_first` from the next shard, ensure the data is fully computed and visible on the target device before switching back.
- **Boundary Conditions:** The algorithm leaves the gradient at the very top of the last shard and the very bottom of the first shard (if not handled by a previous shard) as zero. This is a form of zero-padding.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sharded_gradients(shards, xp):
    """
    Compute gradients for a volume sharded along the first dimension (z-axis).
    
    Args:
        shards: List of arrays, where shards[i] is the i-th shard.
        xp: Array manipulation module (e.g., cupy, numpy).
        
    Returns:
        gx, gy, gz: List of gradient shards, same structure as input.
    """
    n_shards = len(shards)
    if n_shards == 0:
        return [], [], []
        
    # Initialize output gradients with zeros
    gx = [xp.zeros_like(s) for s in shards]
    gy = [xp.zeros_like(s) for s in shards]
    gz = [xp.zeros_like(s) for s in shards]
    
    for i in range(n_shards):
        # Set device context to current shard's device
        # Note: In practice, this might involve setting the current device
        # or using context managers. Here we assume xp operations are device-aware
        # or we explicitly set the device.
        current_shard = shards[i]
        current_gx = gx[i]
        current_gy = gy[i]
        current_gz = gz[i]
        
        # Compute Gx (x-direction, last axis)
        # Forward difference: V[x+1] - V[x]
        # Shape: (H, W, D) -> (H, W, D-1) assigned to (H, W, D-1)
        if current_shard.shape[2] > 1:
            current_gx[:, :, :-1] = current_shard[:, :, 1:] - current_shard[:, :, :-1]
            
        # Compute Gy (y-direction, middle axis)
        # Forward difference: V[y+1] - V[y]
        # Shape: (H, W, D) -> (H, W-1, D) assigned to (H, W-1, D)
        if current_shard.shape[1] > 1:
            current_gy[:, :-1, :] = current_shard[:, 1:, :] - current_shard[:, :-1, :]
            
        # Compute Gz (z-direction, first axis)
        # Internal differences
        if current_shard.shape[0] > 1:
            current_gz[:-1, :, :] = current_shard[1:, :, :] - current_shard[:-1, :, :]
            
        # Handle boundary with next shard for Gz
        if i + 1 < n_shards:
            next_shard = shards[i + 1]
            # Extract first row of next shard
            # We need to ensure we are on the correct device to access next_shard
            # If next_shard is on a different device, we must copy or switch context
            # Assuming xp handles device placement or we switch explicitly
            next_first = next_shard[0, :, :]
            
            # Compute last row of current shard's Gz
            # Gz[-1] = V[z+1] - V[z] where V[z+1] is from next shard
            current_gz[-1, :, :] = next_first - current_shard[-1, :, :]
            
    return gx, gy, gz
```

## Subtopics

### [Brick-local Gradient Estimation](8810-brick-local-gradient-estimation.md)
Computes gradients within local bricks, potentially reducing inter-device communication by processing larger contiguous blocks. Choose when memory bandwidth is the bottleneck and bricks fit in cache.

### [Sobel-based Gradient Estimation](9967-sobel-based-gradient-estimation.md)
Uses Sobel filters for gradient estimation, providing noise reduction and smoother gradients. Choose when noise robustness is more important than sharp edge preservation.

### [Central Difference Stencils](9968-central-difference-stencils.md)
Uses central differences for higher accuracy ($O(h^2)$). Choose when precision is critical and boundary effects can be managed.

## Operating characteristics
- **Time Complexity:** $O(N \cdot H \cdot W \cdot D)$, where $N$ is the number of shards and $H, W, D$ are the dimensions of each shard. The complexity is linear with respect to the total number of voxels.
- **Memory Complexity:** $O(N \cdot H \cdot W \cdot D)$ for storing the input and output gradients. Additional memory is required for temporary arrays during device switching.
- **Measured Performance:** Performance depends heavily on the efficiency of device switching and memory bandwidth. In distributed GPU setups, minimizing the number of device switches is crucial. The source implementation uses `cupy` for GPU acceleration, which provides high throughput for array operations [S1, unofficial].
- **Strengths:** Scalable to large volumes by partitioning across multiple devices. Handles inter-shard boundaries explicitly.
- **Limitations:** Device switching overhead can be significant. The forward difference scheme is only first-order accurate ($O(h)$). Boundary conditions at the global volume boundaries are not explicitly defined beyond zero-padding.
- **When to Use:** Large-scale volume rendering or reconstruction tasks where the volume does not fit in a single device's memory.
- **When Not to Use:** Small volumes that fit in a single device, or when higher-order accuracy is required.

## Background
Sharded Gradient Computation is part of the TomoAnchor-V3 project, a tool for tomographic reconstruction [S1, unofficial]. It was developed to handle large volumes in distributed computing environments, specifically addressing the challenges of gradient computation across shard boundaries in a multi-GPU setup. The method is used in the context of iterative reconstruction algorithms, such as those based on ADMM (Alternating Direction Method of Multipliers), where gradient computation is a key step [S1, unofficial].

## Sources
[S1] alidarbandi/TomoAnchor-V3, https://github.com/alidarbandi/TomoAnchor-V3, fetched 2026-10-27 (unofficial)
