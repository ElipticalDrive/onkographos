# Inverse CDF Ray Sampling


## Overview
Inverse CDF Ray Sampling generates sample points along a ray by mapping uniform random noise to depth intervals using the cumulative distribution function (CDF) derived from per-hit probabilities. It solves the problem of stratified sampling within non-uniform depth bins defined by ray-triangle or ray-volume intersections, ensuring that samples are distributed according to the likelihood of each interval while maintaining stratification to reduce variance. It differs from simple uniform sampling by weighting samples toward intervals with higher probability mass, and from pure stratified sampling by allowing arbitrary probability distributions rather than uniform bin weights.

## How it works
### Inputs and Outputs
**Inputs:**
*   `probs`: Probability distribution for each hit along the ray. Shape: `[batch_size, num_rays, max_hits]`. Each row sums to 1.0 (or is normalized).
*   `steps`: Target number of samples per ray. Shape: `[batch_size, num_rays]`.
*   `min_depth`: Minimum depth of the current hit interval. Shape: `[batch_size, num_rays, max_hits]`.
*   `max_depth`: Maximum depth of the current hit interval. Shape: `[batch_size, num_rays, max_hits]`.
*   `pts_idx`: Index of the primitive/vertex associated with each hit. Shape: `[batch_size, num_rays, max_hits]`.
*   `fixed_step_size`: Optional scalar. If $> 0$, overrides dynamic step calculation for uniform spacing within bins. If $-1$, uses dynamic calculation.

**Outputs:**
*   `sampled_idx`: Primitive indices for each sample. Shape: `[batch_size, num_rays, max_samples_per_ray]`.
*   `sampled_dists`: Distance from the start of the current bin to the sample. Shape: `[batch_size, num_rays, max_samples_per_ray]`.
*   `sampled_depth`: Absolute depth of the sample. Shape: `[batch_size, num_rays, max_samples_per_ray]`.

### Data Structures
The method processes data in chunks to manage memory.
*   **Chunk Size ($G$):** 200.
*   **Batch Chunk Size:** $4 \times G = 800$ rays processed per CUDA kernel launch to avoid Out-Of-Memory (OOM) errors.
*   **Padding:** Inputs are padded to multiples of $G$. Let $H$ be the padded ray count.
*   **Reshaping:** Data is viewed as `(G, -1, P)` for 3D tensors and `(G, -1)` for 2D tensors, where $P$ is the point dimension (e.g., 3 for depth/idx).

### Algorithm Steps
1.  **Initialization:**
    *   Set $G = 200$.
    *   Determine ray count $N$ and point dimension $P$.
    *   Pad inputs to size $H$ (multiple of $G$).
    *   Calculate `max_steps` as $\lceil \max(\text{steps}) \rceil + P$. This defines the output length dimension.
    *   Generate a noise tensor of shape `[H, max_steps]`.
    *   If `deterministic` is true, set noise to $0.5$. Otherwise, sample uniform $u \in [0, 1)$ and clamp to $[0.001, 0.999]$ to prevent edge cases in division [S1, unofficial].

2.  **Chunking:**
    *   Iterate through rays in chunks of $4 \times G$.

3.  **Per-Chunk Processing (CUDA Kernel Logic):**
    For each ray $k$ in the chunk:
    *   Let $K$ be the global index offset for the chunk.
    *   Let $s$ be the sample index counter, initialized to 0.
    *   Let $z_{low}$ be the lower bound of the current bin, initialized to `min_depth` of the first hit.
    *   Let $curr\_bin$ be the current hit index, initialized to 0.
    *   Let $curr\_min\_cdf$ be the cumulative probability at the start of the current bin, initialized to 0.
    *   Let $curr\_max\_cdf$ be the cumulative probability at the end of the current bin, initialized to `probs[k, 0]`.
    *   Let $curr\_min\_depth$ be `min_depth[k, 0]`.
    *   Let $curr\_max\_depth$ be `max_depth[k, 0]`.
    *   Let $done$ be false.

    **Loop over target steps:**
    For $curr\_step$ from 0 to `total_steps` - 1:
    *   Calculate the target CDF value:
        $$ curr\_cdf = (curr\_step + noise[K + curr\_step]) \times step\_size $$
        Where $step\_size = 1.0 / \text{steps}[k]$ if `fixed_step_size` $\le 0$, else $step\_size = \text{fixed\_step\_size}$.
        *Note: The source implies `step_size` scales the noise addition to the integer step index. In standard inverse CDF, this maps the uniform noise to the [0,1] probability space.*

    *   **Skip Empty/Invalid Bins:**
        While $curr\_cdf > curr\_max\_cdf$:
        *   If $curr\_bin \ge max\_hits$ or `pts_idx[k, curr_bin] == -1`:
            *   Set $done = true$. Break.
        *   Increment $curr\_bin$.
        *   Update bounds:
            $$ curr\_min\_depth = \text{min\_depth}[k, curr\_bin] $$
            $$ curr\_max\_depth = \text{max\_depth}[k, curr\_bin] $$
            $$ curr\_min\_cdf = curr\_max\_cdf $$
            $$ curr\_max\_cdf += \text{probs}[k, curr\_bin] $$
            $$ z_{low} = curr\_min\_depth $$
        *   If $done$, break outer loop.

    *   **Sample Within Current Bin:**
        If not $done$:
        *   Calculate normalized position within the bin:
            $$ u = \frac{curr\_cdf - curr\_min\_cdf}{curr\_max\_cdf - curr\_min\_cdf} $$
        *   Calculate depth:
            $$ z = curr\_min\_depth + u \times (curr\_max\_depth - curr\_min\_depth) $$
        *   Store sample:
            $$ \text{sampled\_idx}[K + s] = \text{pts\_idx}[k, curr\_bin] $$
            $$ \text{sampled\_dists}[K + s] = z - z_{low} $$
            $$ \text{sampled\_depth}[K + s] = \frac{z + z_{low}}{2} $$
        *   Update $z_{low} = z$.
        *   Increment $s$.

    *   **Handle Remaining Bins (if step size is fixed or to fill quota):**
        While $z_{low} < curr\_max\_depth$ AND NOT $done$:
        *   This loop handles cases where multiple samples might fall into the remaining part of the current bin or subsequent bins if the step logic doesn't cover all hits. However, the primary logic above iterates by *step*. The "Handle remaining bins" section in the source suggests a fallback or specific handling for fixed step sizes or boundary conditions.
        *   Specifically, if `fixed_step_size` is used, the logic might differ. The source notes: "If `fixed_step_size` > 0.0, overrides the calculated step size for uniform spacing within bins."
        *   The loop:
            $$ \text{sampled\_idx}[K + s] = \text{pts\_idx}[k, curr\_bin] $$
            $$ \text{sampled\_dists}[K + s] = curr\_max\_depth - z_{low} $$
            $$ \text{sampled\_depth}[K + s] = \frac{curr\_max\_depth + z_{low}}{2} $$
            Increment $curr\_bin$ and $s$.
            If $curr\_bin \ge max\_hits$ or `pts_idx[k, curr_bin] == -1`, break.
            Update $curr\_min\_depth, curr\_max\_depth, z_{low}$.

4.  **Post-Processing:**
    *   Concatenate results from all chunks.
    *   Reshape outputs to `[H, -1]`.
    *   Truncate to $N$ rays (remove padding).
    *   Trim outputs to `max_len` (the maximum valid samples per ray, usually `max_steps`).
    *   Mark outputs as non-differentiable.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_cdf_sampling(probs, steps, min_depth, max_depth, pts_idx, fixed_step_size=-1, deterministic=False):
    """
    Implements Inverse CDF Ray Sampling.
    """
    batch_size, num_rays, max_hits = probs.shape
    G = 200
    H = ((num_rays + G - 1) // G) * G  # Pad to multiple of G
    
    # Pad inputs
    probs_pad = pad(probs, (0, 0, 0, H - num_rays))
    steps_pad = pad(steps, (0, H - num_rays))
    min_depth_pad = pad(min_depth, (0, 0, 0, H - num_rays))
    max_depth_pad = pad(max_depth, (0, 0, 0, H - num_rays))
    pts_idx_pad = pad(pts_idx, (0, 0, 0, H - num_rays))
    
    # Calculate max samples needed
    max_steps_val = int(torch.ceil(steps_pad).max().item())
    max_samples = max_steps_val + 3  # P=3 buffer
    
    # Generate noise
    noise = torch.rand(H, max_samples)
    if deterministic:
        noise.fill_(0.5)
    else:
        noise = torch.clamp(noise, 0.001, 0.999)
        
    results = []
    chunk_size = 4 * G
    
    for start in range(0, H, chunk_size):
        end = min(start + chunk_size, H)
        chunk_len = end - start
        
        # Extract chunk data
        c_probs = probs_pad[start:end]
        c_steps = steps_pad[start:end]
        c_min_depth = min_depth_pad[start:end]
        c_max_depth = max_depth_pad[start:end]
        c_pts_idx = pts_idx_pad[start:end]
        c_noise = noise[start:end]
        
        # Call CUDA kernel (simulated logic for description)
        # The actual implementation is in C++/CUDA
        chunk_samples = cuda_inverse_cdf_kernel(
            c_probs, c_steps, c_min_depth, c_max_depth, c_pts_idx, 
            c_noise, fixed_step_size, max_samples
        )
        results.append(chunk_samples)
        
    # Concatenate and reshape
    all_samples = torch.cat(results, dim=1) # Shape: [H, max_samples]
    all_samples = all_samples[:, :num_rays, :] # Truncate padding
    
    # Trim to actual max steps per ray
    final_samples = []
    for i in range(num_rays):
        num_s = int(steps[i].item())
        final_samples.append(all_samples[i, :num_s])
        
    return final_samples

def cuda_inverse_cdf_kernel(probs, steps, min_depth, max_depth, pts_idx, noise, fixed_step_size, max_samples):
    """
    Simulates the CUDA kernel logic.
    """
    batch_size, num_rays, max_hits = probs.shape
    output = torch.zeros(batch_size, num_rays, max_samples)
    output_idx = torch.zeros_like(output).long()
    output_dist = torch.zeros_like(output)
    
    for k in range(num_rays):
        s = 0
        z_low = min_depth[k, 0]
        curr_bin = 0
        curr_min_cdf = 0.0
        curr_max_cdf = probs[k, 0]
        curr_min_depth = min_depth[k, 0]
        curr_max_depth = max_depth[k, 0]
        done = False
        
        step_size = fixed_step_size if fixed_step_size > 0 else 1.0 / steps[k]
        
        for curr_step in range(int(steps[k].item())):
            if done:
                break
                
            curr_cdf = (curr_step + noise[k, curr_step]) * step_size
            
            # Skip bins where curr_cdf exceeds current bin's max CDF
            while curr_cdf > curr_max_cdf:
                curr_bin += 1
                if curr_bin >= max_hits or pts_idx[k, curr_bin] == -1:
                    done = True
                    break
                curr_min_depth = min_depth[k, curr_bin]
                curr_max_depth = max_depth[k, curr_bin]
                curr_min_cdf = curr_max_cdf
                curr_max_cdf += probs[k, curr_bin]
                z_low = curr_min_depth
                
            if done:
                break
                
            # Sample within bin
            u = (curr_cdf - curr_min_cdf) / (curr_max_cdf - curr_min_cdf)
            z = curr_min_depth + u * (curr_max_depth - curr_min_depth)
            
            output[k, curr_step, 0] = z # Depth
            output_dist[k, curr_step, 0] = z - z_low
            output_idx[k, curr_step, 0] = pts_idx[k, curr_bin]
            
            z_low = z
            s += 1
            
    return output, output_idx, output_dist
```

### Parameters
*   `fixed_step_size`: If $> 0$, overrides dynamic step calculation. If $-1$, uses $1/\text{steps}$ [S1, unofficial].
*   `probs`: Shape `[b, num_rays, max_hits]` [S1, unofficial].
*   `steps`: Shape `[b, num_rays]` [S1, unofficial].

### Pitfalls
*   **Noise Clamping:** Noise must be clamped to $[0.001, 0.999]$ to prevent division by zero or invalid CDF lookups at the exact boundaries [S1, unofficial].
*   **Padding:** Inputs must be padded to multiples of $G=200$ to align with the kernel's block structure.
*   **Invalid Hits:** Check for `pts_idx == -1` to handle rays with fewer hits than `max_hits`.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the number of samples per ray. The bin-skipping loop adds overhead proportional to the number of hits, but typically $S \ll \text{max\_hits}$.
*   **Memory Complexity:** $O(N \cdot S)$ for outputs. Intermediate padding adds $O(G \cdot S)$ overhead.
*   **Performance:** Chunking by $4 \times G$ (800 rays) balances GPU occupancy and memory usage. The method is efficient on CUDA hardware due to parallel ray processing.
*   **Strengths:** Provides stratified sampling with arbitrary probability distributions, reducing variance compared to uniform sampling.
*   **Limitations:** Requires pre-computed probabilities and hit intervals. Less flexible than pure random sampling for highly dynamic scenes where probabilities change rapidly.

## Background
Inverse CDF sampling is a standard technique in Monte Carlo integration, applied to volume rendering to sample along rays according to the estimated density or opacity distribution. In the context of Neural Volume Rendering (NSVF), it is used to efficiently sample points within the reconstructed volume based on the learned probability distribution of surface intersections [S1, unofficial].

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
