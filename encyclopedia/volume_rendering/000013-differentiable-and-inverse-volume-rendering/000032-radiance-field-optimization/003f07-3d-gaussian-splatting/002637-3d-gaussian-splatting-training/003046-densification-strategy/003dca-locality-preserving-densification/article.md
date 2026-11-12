# Locality-preserving Densification

Also called: Z-ordering for Gaussians

## Overview
Locality-preserving Densification addresses the memory layout bottleneck in 3D Gaussian Splatting training, where adding new primitives to the end of parameter buffers causes spatially close Gaussians to be stored far apart in memory [S1, unofficial]. By reordering Gaussians based on their 3D spatial proximity using Z-ordering (Morton order), the method ensures that neighboring Gaussians in space are also adjacent in memory, thereby improving memory access coalescing [S1, unofficial]. This reduces warp divergence and cache misses, resulting in faster training for scenes with high Gaussian counts [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** The current set of $N$ Gaussians, each defined by position $\mu_k$, scaling $s_k$, rotation $q_k$, opacity $\alpha_k$, and spherical harmonics coefficients. The 3D bounding box of the scene or active region.
- **Output:** A permuted list of Gaussian parameters where spatial locality is preserved in memory layout.

**Data Structures:**
- **Gaussian Buffer:** A contiguous array of structures containing all Gaussian attributes.
- **Z-order Key:** A 64-bit integer derived from the 3D position of each Gaussian, used for sorting.

**Steps:**
1. **Trigger Check:** Determine if the current training iteration $t$ is a multiple of the application frequency $T_{freq}$ (typically 5000) [S1, unofficial]. If not, skip.
2. **Coordinate Normalization:** For each Gaussian $k$, normalize its 3D position $\mu_k = (x_k, y_k, z_k)$ into the range $[0, 1]$ relative to the scene bounding box or a defined active region [S1, unofficial].
   $$ x'_k = \frac{x_k - x_{min}}{x_{max} - x_{min}}, \quad y'_k = \frac{y_k - y_{min}}{y_{max} - y_{min}}, \quad z'_k = \frac{z_k - z_{min}}{z_{max} - z_{min}} $$
3. **Z-order Encoding:** Convert the normalized floating-point coordinates into a 64-bit Morton code. This involves interleaving the bits of the fixed-point representations of $x'_k, y'_k, z'_k$ [S1, unofficial].
   - Let $X, Y, Z$ be the integer representations of the normalized coordinates (e.g., 21 bits each for 64-bit total).
   - The Morton code $M_k$ is constructed by interleaving bits: $M_k = \text{interleave}(X, Y, Z)$.
4. **Sorting:** Sort the Gaussians based on their Morton codes $M_k$ in ascending order. This places spatially adjacent Gaussians next to each other in the buffer [S1, unofficial].
5. **Buffer Reordering:** Rearrange the Gaussian attributes (position, scale, rotation, opacity, SH coefficients) in the main parameter buffer according to the sorted order.

**Parameters:**
- **Application Frequency ($T_{freq}$):** 5000 iterations [S1, unofficial].
- **Computational Cost:** Approximately 4 ms per million Gaussians [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Atomic Contention:** This method performs effectively only when used with a per-Gaussian backward pass. If the original backward pass with numerous atomic operations is used, Z-ordering can heighten atomic contention [S1, unofficial].
- **Small Scene Overhead:** Repeated Z-ordering slows down training when the number of Gaussians is small due to a massive increase in atomic contention in the alpha blending backward pass (false sharing) [S1, unofficial].
- **Precision:** The Morton code generation must handle the precision of the normalized coordinates carefully to ensure consistent ordering [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_z_ordering(gaussians, iteration, freq=5000):
    """
    Reorders Gaussians in the buffer based on 3D spatial locality using Z-ordering.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - pos: (x, y, z) float32
                   - scale: (sx, sy, sz) float32
                   - rotation: (qw, qx, qy, qz) float32
                   - opacity: float32
                   - sh_coeffs: array of float32
        iteration: Current training iteration number.
        freq: Frequency of Z-ordering application (default 5000).
    
    Returns:
        None (modifies gaussians list in place).
    """
    # Step 1: Trigger Check
    if iteration % freq != 0:
        return

    # Step 2: Coordinate Normalization
    # Assume scene bounds are known or passed as argument. 
    # For simplicity, we assume normalized coordinates are available or can be computed.
    # In practice, bounds [x_min, x_max], [y_min, y_max], [z_min, z_max] are needed.
    # Here we assume a helper function get_normalized_coords exists or bounds are global.
    
    n = len(gaussians)
    morton_codes = []
    
    for g in gaussians:
        # Normalize position to [0, 1]
        # Note: In a real implementation, scene bounds must be tracked.
        # We assume pos is already normalized or bounds are accessible.
        x, y, z = g.pos
        # Placeholder for actual normalization logic if pos is not normalized
        # x_norm = (x - x_min) / (x_max - x_min)
        # y_norm = (y - y_min) / (y_max - y_min)
        # z_norm = (z - z_min) / (z_max - z_min)
        
        # Convert to fixed-point integers (e.g., 21 bits each for 64-bit Morton)
        # Assuming pos is in [0, 1], scale to 2^21
        bits = 21
        X = int(x * (1 << bits))
        Y = int(y * (1 << bits))
        Z = int(z * (1 << bits))
        
        # Interleave bits to create Morton code
        morton_code = interleave_bits(X, Y, Z, bits)
        morton_codes.append(morton_code)

    # Step 3: Sorting
    # Create indices and sort based on morton codes
    indices = list(range(n))
    indices.sort(key=lambda i: morton_codes[i])

    # Step 4: Buffer Reordering
    # Rearrange all attributes in the gaussians list
    sorted_gaussians = [gaussians[i] for i in indices]
    
    # Update the main buffer (in-place modification of the list)
    for i, g in enumerate(sorted_gaussians):
        gaussians[i] = g

def interleave_bits(x, y, z, bits):
    """
    Interleaves bits of x, y, z to form a Morton code.
    Assumes x, y, z are non-negative integers < 2^bits.
    """
    morton = 0
    for i in range(bits):
        morton |= ((x >> i) & 1) << (3 * i)
        morton |= ((y >> i) & 1) << (3 * i + 1)
        morton |= ((z >> i) & 1) << (3 * i + 2)
    return morton
```

## Operating characteristics
- **Time Complexity:** The sorting step dominates, typically $O(N \log N)$ where $N$ is the number of Gaussians. The Morton code generation is $O(N \cdot B)$ where $B$ is the number of bits.
- **Memory Complexity:** $O(N)$ for storing the Morton codes and the permuted buffer.
- **Measured Performance:** Computational cost is approximately 4 ms per million Gaussians [S1, unofficial]. It reduces warp divergence and cache misses, leading to faster training when scenes contain many Gaussians [S1, unofficial].
- **Scaling:** Benefits increase with the number of Gaussians. For small numbers of Gaussians, the overhead and potential false sharing may negate benefits [S1, unofficial].
- **Strengths:** Improves memory access coalescing, reduces warp divergence, and accelerates training for large-scale scenes.
- **Limitations:** Ineffective or detrimental if used with backward passes that rely heavily on atomic operations (original backward pass) [S1, unofficial]. Diminishing returns with frequent application [S1, unofficial].
- **When to Use:** When training scenes with high Gaussian counts (e.g., >1M) and using a per-Gaussian backward pass.
- **When Not to Use:** When the number of Gaussians is small, or when using the original backward pass with atomic operations.

## Background
Locality-preserving Densification, also known as Z-ordering for Gaussians, was introduced in the context of optimizing 3D Gaussian Splatting training. It addresses the memory layout issues identified in the original 3DGS implementation where densification leads to poor memory access patterns [S1, unofficial]. The method leverages Z-ordering (Morton order) to spatially sort Gaussians, a technique commonly used in spatial indexing [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
