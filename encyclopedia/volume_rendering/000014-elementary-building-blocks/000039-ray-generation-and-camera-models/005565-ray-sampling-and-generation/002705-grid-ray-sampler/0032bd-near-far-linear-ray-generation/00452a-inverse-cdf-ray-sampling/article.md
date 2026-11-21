# Inverse CDF Ray Sampling


## Overview
Inverse CDF Ray Sampling is a CUDA kernel that samples points along rays using inverse CDF sampling based on probability distributions (probs) associated with depth intervals (hits), supporting optional fixed step size [S1, unofficial]. It maps uniform random noise to depth intervals using the cumulative distribution function derived from per-hit probabilities [S1, unofficial]. It builds on [Near-Far Linear Ray Generation](6841-near-far-linear-ray-generation.md) by replacing uniform spacing with probability-weighted stratification.

## How it works
### Inputs and Outputs
**Inputs:**
* `probs`: Probability distribution for each hit along the ray. Shape: `[batch_size, num_rays, max_hits]` [S1, unofficial].
* `steps`: Target number of samples per ray. Shape: `[batch_size, num_rays]` [S1, unofficial].
* `min_depth`: Minimum depth of the current hit interval. Shape: `[batch_size, num_rays, max_hits]` [S1, unofficial].
* `max_depth`: Maximum depth of the current hit interval. Shape: `[batch_size, num_rays, max_hits]` [S1, unofficial].
* `pts_idx`: Index of the primitive/vertex associated with each hit. Shape: `[batch_size, num_rays, max_hits]` [S1, unofficial].
* `fixed_step_size`: Optional scalar. If $> 0$, overrides dynamic step calculation for uniform spacing within bins. If $-1$, uses dynamic calculation [S1, unofficial].

**Outputs:**
* `sampled_idx`: Primitive indices for each sample. Shape: `[batch_size, num_rays, max_samples_per_ray]` [S1, unofficial].
* `sampled_dists`: Distance from the start of the current bin to the sample. Shape: `[batch_size, num_rays, max_samples_per_ray]` [S1, unofficial].
* `sampled_depth`: Absolute depth of the sample. Shape: `[batch_size, num_rays, max_samples_per_ray]` [S1, unofficial].

### Data Structures
The method processes data in chunks to manage memory [S1, unofficial].
* **Chunk Size ($G$):** 200 [S1, unofficial].
* **Batch Chunk Size:** $4 \times G = 800$ rays processed per CUDA kernel launch to avoid Out-Of-Memory (OOM) errors [S1, unofficial].
* **Padding:** Inputs are padded to multiples of $G$. Let $H$ be the padded ray count [S1, unofficial].
* **Reshaping:** Data is viewed as `(G, -1, P)` for 3D tensors and `(G, -1)` for 2D tensors, where $P$ is the point dimension (e.g., 3 for depth/idx) [S1, unofficial].

### Algorithm Steps
1. Set batch chunk size $G = 200$ [S1, unofficial].
2. Determine ray count $N$ and point dimension $P$ [S1, unofficial].
3. Pad inputs to size $H$ (multiple of $G$) [S1, unofficial].
4. Reshape inputs to `(G, -1, P)` for data and `(G, -1)` for steps [S1, unofficial].
5. Calculate `max_steps = steps.ceil().long().max() + P` [S1, unofficial].
6. Generate noise tensor of shape `[H, max_steps]` [S1, unofficial].
7. If `deterministic`, set noise to $0.5$; else, sample uniform $u \in [0, 1)$ and clamp to $[0.001, 0.999]$ [S1, unofficial].
8. Process in chunks of $4 \times G$ to avoid OOM [S1, unofficial].
9. Call CUDA extension `_ext.inverse_cdf_sampling` with `pts_idx`, `min_depth`, `max_depth`, `noise`, `probs`, `steps`, and `fixed_step_size` [S1, unofficial].
10. Concatenate results from chunks [S1, unofficial].
11. Reshape outputs to `(H, -1)` [S1, unofficial].
12. Truncate to $N$ rays if padded [S1, unofficial].
13. Trim outputs to `max_len` (max valid samples per ray) [S1, unofficial].
14. Mark outputs as non-differentiable [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def pad_1d(arr, pad_right):
    return arr + [0] * pad_right

def pad_2d(arr, pad_right):
    return [row + [0] * pad_right for row in arr]

def inverse_cdf_sampling(probs, steps, min_depth, max_depth, pts_idx, fixed_step_size=-1, deterministic=False):
    batch_size, num_rays, max_hits = len(probs), len(probs[0]), len(probs[0][0])
    G = 200
    H = ((num_rays + G - 1) // G) * G
    P = 3

    probs_pad = pad_2d(probs, H - num_rays)
    steps_pad = pad_1d(steps, H - num_rays)
    min_depth_pad = pad_2d(min_depth, H - num_rays)
    max_depth_pad = pad_2d(max_depth, H - num_rays)
    pts_idx_pad = pad_2d(pts_idx, H - num_rays)

    max_steps_val = math.ceil(max(steps_pad)) + P
    noise = [[0.5 if deterministic else max(0.001, min(0.999, u)) for u in [0.0] * max_steps_val] for _ in range(H)]
    if not deterministic:
        for i in range(H):
            for j in range(max_steps_val):
                noise[i][j] = max(0.001, min(0.999, random.random()))

    results = []
    chunk_size = 4 * G

    for start in range(0, H, chunk_size):
        end = min(start + chunk_size, H)
        chunk_len = end - start
        chunk_samples = cuda_inverse_cdf_kernel(
            probs_pad[start:end], steps_pad[start:end],
            min_depth_pad[start:end], max_depth_pad[start:end],
            pts_idx_pad[start:end], noise[start:end],
            fixed_step_size, max_steps_val, start
        )
        results.append(chunk_samples)

    all_samples = [item for sublist in results for item in sublist]
    all_samples = all_samples[:num_rays]
    final_samples = [ray[:max(steps[i])] for i, ray in enumerate(all_samples)]
    return final_samples

def cuda_inverse_cdf_kernel(probs, steps, min_depth, max_depth, pts_idx, noise, fixed_step_size, max_steps, K_offset):
    batch_size, num_rays, max_hits = len(probs), len(probs[0]), len(probs[0][0])
    output_idx = []
    output_dist = []
    output_depth = []

    for k in range(num_rays):
        s = 0
        z_low = min_depth[k][0]
        curr_bin = 0
        curr_min_cdf = 0.0
        curr_max_cdf = probs[k][0]
        curr_min_depth = min_depth[k][0]
        curr_max_depth = max_depth[k][0]
        done = False
        step_size = fixed_step_size if fixed_step_size > 0 else 1.0 / steps[k]

        for curr_step in range(int(steps[k])):
            if done:
                break
            curr_cdf = (float(curr_step) + noise[k][curr_step]) * step_size

            while curr_cdf > curr_max_cdf:
                output_idx.append(pts_idx[k][curr_bin])
                output_dist.append(curr_max_depth - z_low)
                output_depth.append((curr_max_depth + z_low) * 0.5)
                curr_bin += 1
                s += 1
                if curr_bin >= max_hits or pts_idx[k][curr_bin] == -1:
                    done = True
                    break
                curr_min_depth = min_depth[k][curr_bin]
                curr_max_depth = max_depth[k][curr_bin]
                curr_min_cdf = curr_max_cdf
                curr_max_cdf += probs[k][curr_bin]
                z_low = curr_min_depth
            if done:
                break

            u = (curr_cdf - curr_min_cdf) / (curr_max_cdf - curr_min_cdf)
            z = curr_min_depth + u * (curr_max_depth - curr_min_depth)
            output_idx.append(pts_idx[k][curr_bin])
            output_dist.append(z - z_low)
            output_depth.append((z + z_low) * 0.5)
            z_low = z
            s += 1

            while z_low < curr_max_depth and not done:
                output_idx.append(pts_idx[k][curr_bin])
                output_dist.append(curr_max_depth - z_low)
                output_depth.append((curr_max_depth + z_low) * 0.5)
                curr_bin += 1
                s += 1
                if curr_bin >= max_hits or pts_idx[k][curr_bin] == -1:
                    break
                curr_min_depth = min_depth[k][curr_bin]
                curr_max_depth = max_depth[k][curr_bin]
                z_low = curr_min_depth

        output_idx.append([idx + K_offset for idx in output_idx[-s:]])
        output_dist.append(output_dist[-s:])
        output_depth.append(output_depth[-s:])

    return output_idx, output_dist, output_depth
```

### Parameters
* `fixed_step_size`: If $> 0$, overrides dynamic step calculation. If $-1$, uses dynamic calculation [S1, unofficial].
* `probs`: Shape `[b, num_rays, max_hits]` [S1, unofficial].
* `steps`: Shape `[b, num_rays]` [S1, unofficial].

### Pitfalls
* **Noise Clamping:** Noise must be clamped to $[0.001, 0.999]$ to prevent division by zero or invalid CDF lookups at the exact boundaries [S1, unofficial].
* **Padding:** Inputs must be padded to multiples of $G=200$ to align with the kernel's block structure [S1, unofficial].
* **Invalid Hits:** Check for `pts_idx == -1` to handle rays with fewer hits than `max_hits` [S1, unofficial].

## Operating characteristics
* **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the number of samples per ray. The bin-skipping loop adds overhead proportional to the number of hits, but typically $S \ll \text{max\_hits}$ [G].
* **Memory Complexity:** $O(N \cdot S)$ for outputs. Intermediate padding adds $O(G \cdot S)$ overhead [G].
* **Performance:** Chunking by $4 \times G$ (800 rays) balances GPU occupancy and memory usage. The method is efficient on CUDA hardware due to parallel ray processing [S1, unofficial].
* **Strengths:** Provides stratified sampling with arbitrary probability distributions, reducing variance compared to uniform sampling [G].
* **Limitations:** Requires pre-computed probabilities and hit intervals. Less flexible than pure random sampling for highly dynamic scenes where probabilities change rapidly [G].

## Background
Inverse CDF sampling is a standard technique in Monte Carlo integration, applied to volume rendering to sample along rays according to the estimated density or opacity distribution. In the context of Neural Volume Rendering (NSVF), it is used to efficiently sample points within the reconstructed volume based on the learned probability distribution of surface intersections [S1, unofficial].

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S2] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
