# Locality-preserving Densification

Also called: Z-ordering for Gaussians

## Overview
Locality-preserving densification addresses the memory access throttling caused by standard Gaussian densification, where new primitives are appended to the end of parameter buffers, causing spatially close Gaussians to be stored far apart in memory [S1, unofficial]. By reordering the Gaussian parameter buffers based on 3D spatial proximity using Z-ordering (Morton order), this method ensures that neighboring Gaussians in space are also adjacent in memory, thereby improving memory access coalescing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of $N$ 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$ and the current parameter buffer layout [S1, unofficial].
*   **Output:** A permuted parameter buffer where the indices of the Gaussians are reordered such that spatial locality is preserved [S1, unofficial].

**Data Structures:**
*   **Gaussian Buffer:** An array of structures containing $\mu_k, \Sigma_k, \alpha_k, \mathbf{c}_k$ [S1, unofficial]. The method operates on the indices of this array [G].
*   **Z-order Key:** A 64-bit integer derived from the 3D coordinates of $\mu_k$ [G].

**Steps:**
1.  **Trigger Check:** Determine if the current training iteration $t$ is a multiple of the Z-ordering frequency $T_{\text{zorder}}$. If not, skip [S1, unofficial].
2.  **Coordinate Normalization:** Normalize the mean positions $\mu_k$ of all active Gaussians to a canonical range (e.g., $[0, 1]$) suitable for Morton coding. Let $\mu'_k$ be the normalized position [G].
3.  **Z-order Encoding:** For each Gaussian $g_k$, compute its Morton code $m_k$ from $\mu'_k$. The Morton code interleaves the bits of the x, y, and z coordinates [G].
    $$ m_k = \text{MortonEncode}(\mu'_k.x, \mu'_k.y, \mu'_k.z) $$
4.  **Sorting:** Create a list of indices $I = [0, 1, \dots, N-1]$ and sort $I$ based on the ascending order of $m_k$ [G].
5.  **Buffer Reordering:** Rearrange the elements in the Gaussian parameter buffer according to the sorted index list $I$. This ensures that Gaussians with similar Morton codes (spatially close) are adjacent in memory [S1, unofficial].
6.  **Backward Pass Compatibility:** Ensure the rendering pipeline uses the per-Gaussian backward pass, as the original backward pass with numerous atomic operations causes high atomic contention when combined with Z-ordering [S1, unofficial].

**Parameters:**
*   **Z-ordering Frequency ($T_{\text{zorder}}$):** 5000 iterations [S1, unofficial].
*   **Z-ordering Computational Cost:** Approximately 4 ms per million Gaussians [S1, unofficial].

**Pitfalls:**
*   **Atomic Contention:** Z-ordering performs effectively *only* when used with the per-Gaussian backward pass. Using it with the original backward pass heightens atomic contention due to the interaction between the reordered memory layout and the atomic accumulation operations [S1, unofficial].
*   **Small Scene Overhead:** Repeated Z-ordering slows down training when the number of Gaussians is small. This is due to a massive increase in atomic contention in the alpha blending backward pass, leading to false sharing [S1, unofficial].
*   **Precision:** Normalization range must be consistent across iterations to ensure stable Morton codes [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_z_ordering(gaussians, iteration, t_zorder=5000, scene_min=None, scene_max=None):
    """
    Reorders the gaussian buffer based on 3D spatial locality using Z-ordering.
    
    Args:
        gaussians: List of Gaussian objects with attribute 'mean' (vec3).
        iteration: Current training iteration number.
        t_zorder: Frequency of Z-ordering application.
        scene_min, scene_max: Bounding box of the scene for normalization [G].
        
    Returns:
        None (modifies gaussians in place).
    """
    # Step 1: Trigger Check
    if iteration % t_zorder != 0:
        return

    n = len(gaussians)
    if n == 0:
        return

    # Step 2 & 3: Compute Morton Codes
    morton_codes = []
    for g in gaussians:
        # [G] Normalize to [0, 1] using scene bounds
        x = (g.mean.x - scene_min.x) / (scene_max.x - scene_min.x)
        y = (g.mean.y - scene_min.y) / (scene_max.y - scene_min.y)
        z = (g.mean.z - scene_min.z) / (scene_max.z - scene_min.z)
        morton_codes.append(morton_encode(x, y, z))

    # Step 4: Sort Indices
    indexed_codes = list(enumerate(morton_codes))
    indexed_codes.sort(key=lambda x: x[1])
    sorted_indices = [idx for idx, _ in indexed_codes]

    # Step 5: Buffer Reordering
    gaussians[:] = [gaussians[i] for i in sorted_indices]

def morton_encode(x, y, z):
    """
    Interleaves bits of x, y, z to form a Morton code.
    Assumes inputs are scaled to integers in [0, 2^21 - 1].
    """
    scale = 1 << 21
    ix = int(x * scale) & ((1 << 21) - 1)
    iy = int(y * scale) & ((1 << 21) - 1)
    iz = int(z * scale) & ((1 << 21) - 1)
    
    m = 0
    for i in range(21):
        m |= ((ix >> i) & 1) << (3 * i)
        m |= ((iy >> i) & 1) << (3 * i + 1)
        m |= ((iz >> i) & 1) << (3 * i + 2)
    return m
```

## Operating characteristics
*   **Time Complexity:** The Z-ordering step itself is $O(N \log N)$ due to sorting, where $N$ is the number of Gaussians [G]. The computational cost is approximately 4 ms per million Gaussians [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the Morton codes and sorted indices [G]. No additional memory is required for the Gaussians themselves [G].
*   **Measured Performance:** Reduces warp divergence and cache misses, resulting in faster training when scenes contain many Gaussians [S1, unofficial]. It outperforms load-balanced approaches which can cause uncoalesced memory accesses that become a bigger bottleneck as primitives increase [S1, unofficial].
*   **Strengths:** Significantly improves memory access patterns during densification, leading to faster training times for large-scale scenes [S1, unofficial].
*   **Limitations:** Adds overhead for small scenes (fewer Gaussians) due to false sharing and atomic contention in the backward pass. Requires the per-Gaussian backward pass to be effective [S1, unofficial].
*   **When to Use:** Use when training scenes with a large number of Gaussians (e.g., >1M) and densification is active. Do not use if the number of Gaussians is small or if the per-Gaussian backward pass is not implemented [S1, unofficial].

## Background
Locality-preserving densification was introduced in the "Faster-GS" paper, which analyzes and improves Gaussian Splatting optimization [S1, unofficial]. The authors observed that memory layout became a throttling factor during densification, as new Gaussians added to the end of buffers caused spatially close primitives to be far apart in memory [S1, unofficial]. By applying Z-ordering every 5000 iterations alongside other optimization passes, the resulting framework achieved up to 5.2× faster training on Deep Blending scenes compared to the original 3DGS implementation [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
