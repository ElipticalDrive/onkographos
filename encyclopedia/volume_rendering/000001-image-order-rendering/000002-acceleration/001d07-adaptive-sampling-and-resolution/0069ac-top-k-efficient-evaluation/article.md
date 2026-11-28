# Top-k Efficient Evaluation

Also called: Top-k MLP Evaluation, Top-k Pruning

## Overview
Top-k Efficient Evaluation is a sparse evaluation strategy for local radiance field representations (specifically Nerflets) that reduces computational cost by evaluating only the most influential neural network components for each volumetric sample [S1]. It solves the problem of excessive MLP evaluations in scenes with many local fields by filtering out distant or low-influence contributors [S1]. It differs from general adaptive sampling by operating at the component level (MLP selection) rather than the spatial sampling level, leveraging the spatial sparsity of basis functions [S1].

## How it works
**Inputs and Outputs:**
*   Input: A set of volumetric samples $\mathbf{x}_i$, a set of local fields (Nerflets) $\{f_j\}$ with associated influence weights $g_j(\mathbf{x}_i)$, and a pruning threshold $\tau$ and count $k$.
*   Output: The accumulated color and depth for each ray, computed using only the selected subset of fields.

**Data Structures:**
*   Influence Map: A sparse structure mapping each sample $\mathbf{x}_i$ to a list of candidate Nerflets $j$ where $g_j(\mathbf{x}_i) > \tau$.
*   Top-k List: For each sample, a sorted list of the $k$ Nerflets with the highest influence values $g_j(\mathbf{x}_i)$.

**Steps:**
1.  **Influence Calculation:** For each sample $\mathbf{x}_i$ along a ray, compute the influence score $g_j(\mathbf{x}_i)$ for all active Nerflets $j$. The influence is typically derived from the spatial support of the basis function (e.g., RBF) [S1].
2.  **Threshold Pruning:** Discard any Nerflet $j$ where $g_j(\mathbf{x}_i) \le \tau$. In free space, this may eliminate all Nerflets [S1].
3.  **Top-k Selection:** From the remaining candidates, select the $k$ Nerflets with the largest influence values $g_j(\mathbf{x}_i)$. Let this set be $S_i$.
4.  **Evaluation:** Evaluate the MLPs $f_j(\mathbf{x}_i)$ only for $j \in S_i$.
5.  **Accumulation:** Composite the results from the evaluated MLPs using standard volume rendering integration [S1].

**Parameters:**
*   $k$: The number of top influences to evaluate. Typical value: $k=16$ [S1].
*   $\tau$: The trivial threshold for influence. Values below this are truncated to 0 [S1].

**Edge Cases:**
*   **Free Space:** If no Nerflets exceed $\tau$, the ray passes through empty space without MLP evaluation [S1].
*   **Aggressive Pruning:** Reducing $k$ to 3 results in a negligible PSNR drop (~0.05) compared to $k=16$, allowing for further compute reduction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def top_k_evaluation(samples, nerflets, k=16, threshold=0.0):
    """
    samples: List of 3D points x_i
    nerflets: List of Nerflet objects, each having influence(x) and evaluate(x)
    k: Number of top influences to keep
    threshold: Minimum influence to consider
    """
    results = []
    
    for x in samples:
        # Step 1 & 2: Calculate influence and filter
        candidates = []
        for nerflet in nerflets:
            influence = nerflet.influence(x)
            if influence > threshold:
                candidates.append((influence, nerflet))
        
        # Step 3: Sort and select top-k
        # Sort descending by influence
        candidates.sort(key=lambda item: item[0], reverse=True)
        top_k_nerflets = [item[1] for item in candidates[:k]]
        
        # Step 4: Evaluate only selected MLPs
        # Note: In practice, this is done in a custom CUDA kernel for performance [S1]
        accumulated_color = 0.0
        accumulated_alpha = 0.0
        
        for nerflet in top_k_nerflets:
            # Evaluate MLP
            color, alpha = nerflet.evaluate(x)
            
            # Standard volume rendering step (simplified)
            # This assumes a specific compositing order; actual implementation 
            # may vary based on the broader rendering context [G]
            weight = alpha * (1.0 - accumulated_alpha)
            accumulated_color += weight * color
            accumulated_alpha += weight
            
        results.append(accumulated_color)
        
    return results
```

## Subtopics

### Interactive Visualization
A two-pass approach designed to amortize memory bandwidth by exploiting the spatial locality of high-influence Nerflets [S1].
1.  **Pass 1:** Determine which Nerflets have high influence in specific regions of the volumetric sample grid [S1].
2.  **Pass 2:** Evaluate small, spatially adjacent subgrids for a particular Nerflet. Since the RBF function has low spatial frequency, a Nerflet is likely high-influence for all samples within a small subgrid [S1]. This allows loading MLP parameters into shared memory once for multiple samples, improving performance over naive top-k evaluation, though it is slower than InstantNGP [S1].

## Operating characteristics
*   **Time Complexity:** Reduced from $O(N \cdot M)$ (where $N$ is samples and $M$ is total Nerflets) to $O(N \cdot k)$, where $k \ll M$ [S1].
*   **Memory:** Requires storing influence maps or computing influences on the fly. The two-pass interactive variant optimizes memory bandwidth usage [S1].
*   **Performance:** Provides substantial compute reduction with minimal quality loss (e.g., ~0.05 PSNR difference between $k=16$ and $k=3$) [S1].
*   **Limitations:** Not as fast as global hashing approaches like InstantNGP [S1]. Requires careful tuning of $k$ and threshold $\tau$ [S1].

## Background
Introduced in the "Nerflets" paper (2023) by [S1], this method addresses the computational bottleneck of evaluating many local radiance fields. It leverages the sparsity pattern of Nerflets, which have consistent local structure, to enable efficient pruning and evaluation [S1].

## Sources
[S1] Nerflets: Local Radiance Fields for Efficient Structure-Aware 3D Scene Representation from 2D Supervision, https://doi.org/10.1109/cvpr52729.2023.00800, fetched 2026-10-26
