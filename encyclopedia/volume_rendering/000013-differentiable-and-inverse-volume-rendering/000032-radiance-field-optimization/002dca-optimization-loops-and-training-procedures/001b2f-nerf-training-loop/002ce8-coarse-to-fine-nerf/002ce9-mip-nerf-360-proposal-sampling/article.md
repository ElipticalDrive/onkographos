# Mip-NeRF 360 Proposal Sampling

Also called: Modified Resampling, Proposal Sampling, midpoint resampling

## Overview
Mip-NeRF 360 Proposal Sampling is a hierarchical sampling strategy used during the training of neural radiance fields to allocate samples efficiently in space. It solves the problem of aliasing and inefficient sampling by generating a second, finer set of sample intervals based on the opacity distribution of a coarse network [S1]. It sets itself apart from standard hierarchical sampling by using a "modified resampling" procedure that employs midpoints of adjacent samples as interval boundaries, which qualitatively reduces aliasing artifacts compared to using sampled points directly as endpoints [S1].

## How it works
The method consists of two main phases: the generation of proposal intervals via modified resampling, and the configuration of the proposal networks.

### Inputs and Outputs
*   **Input:** A set of $n$ sorted sample positions $\{t_1, t_2, \dots, t_n\}$ drawn from a coarse histogram, along with the coarse histogram's bin boundaries.
*   **Output:** A set of $n-1$ fine intervals defined by their endpoints, which are used to sample a fine network.

### Data Structures
The proposal networks use a Hash Grid (NGP) structure.
*   **Hierarchy:** Truncated at maximum grid sizes of 512 and 2048 [S2].
*   **Channels:** Single channel of features is used for proposal networks, as they only predict density, not color [S2].
*   **MLP:** A view-dependent bottleneck of size 256 processed by a 3-layer MLP is used for the final round, though proposal rounds may use simpler structures [S2].

### Steps
1.  **Sample from Coarse Histogram:** Draw $n$ sorted values from the coarse histogram weights [S1]. Let these sorted values be $t_1, t_2, \dots, t_n$.
2.  **Define Fine Intervals via Midpoints:** Use the midpoints of adjacent sample pairs as the endpoints for the new set of $n-1$ fine intervals [S1]. For adjacent samples $t_i$ and $t_{i+1}$, the midpoint is $m_i = \frac{t_i + t_{i+1}}{2}$.
3.  **Handle Boundaries:** Reflect the first sample $t_1$ around the first endpoint and the last sample $t_n$ around the last endpoint to handle boundary conditions [S1]. This ensures the first and last fine intervals are properly bounded by the scene extent.
4.  **Proposal Rounds:** Perform two rounds of proposal sampling to bound scene geometry and recursively generate new sample intervals [S2].
5.  **Final Rendering:** Perform one final NeRF round to render the final set of intervals into an image [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_resampling(coarse_samples, first_endpoint, last_endpoint):
    """
    Generates fine intervals from coarse samples using midpoint resampling.
    
    Args:
        coarse_samples: List of n sorted sample positions from coarse histogram.
        first_endpoint: The start of the first coarse bin.
        last_endpoint: The end of the last coarse bin.
        
    Returns:
        List of n-1 fine intervals, where each interval is (start, end).
    """
    n = len(coarse_samples)
    fine_intervals = []
    
    # Step 1 & 2: Use midpoints of adjacent sample pairs as endpoints
    # We need n-1 intervals, so we compute n-1 midpoints
    midpoints = []
    for i in range(n - 1):
        t_i = coarse_samples[i]
        t_next = coarse_samples[i+1]
        midpoint = (t_i + t_next) / 2.0
        midpoints.append(midpoint)
        
    # Step 3: Handle boundaries by reflecting first and last samples
    # Reflect first sample around first endpoint
    reflected_first = 2 * first_endpoint - coarse_samples[0]
    
    # Reflect last sample around last endpoint
    reflected_last = 2 * last_endpoint - coarse_samples[-1]
    
    # Construct intervals
    # The first interval starts at the reflected first sample and ends at the first midpoint
    fine_intervals.append((reflected_first, midpoints[0]))
    
    # Intermediate intervals start at previous midpoint and end at current midpoint
    for i in range(1, len(midpoints)):
        fine_intervals.append((midpoints[i-1], midpoints[i]))
        
    # The last interval starts at the last midpoint and ends at the reflected last sample
    fine_intervals.append((midpoints[-1], reflected_last))
    
    return fine_intervals

def proposal_sampling_pipeline(coarse_histogram, num_samples=64):
    """
    Executes the two-round proposal sampling process.
    """
    # Round 1: Coarse sampling
    # Sample n values from coarse histogram weights
    coarse_samples = sample_from_histogram(coarse_histogram, num_samples)
    coarse_samples.sort()
    
    # Generate fine intervals for Round 2
    # Assuming first_endpoint and last_endpoint are known from scene bounds
    first_endpoint = coarse_histogram.bins[0].start
    last_endpoint = coarse_histogram.bins[-1].end
    
    fine_intervals_round1 = modified_resampling(coarse_samples, first_endpoint, last_endpoint)
    
    # Round 2: Recursive proposal
    # Sample from the fine intervals generated in Round 1 to create Round 2 intervals
    # This step is described as "recursively generate new sample intervals" [S2]
    # Implementation detail: Sample from the distribution defined by fine_intervals_round1
    round2_samples = sample_from_intervals(fine_intervals_round1, num_samples)
    round2_samples.sort()
    
    # Generate final intervals for the fine network
    final_intervals = modified_resampling(round2_samples, first_endpoint, last_endpoint)
    
    # Final Round: Render using the final intervals
    # Render the final set of intervals into an image [S2]
    image = render_neRF(final_intervals)
    
    return image
```

### Parameters
*   **Number of samples ($n$):** Not explicitly fixed in the notes, but typical values in NeRF literature are 64 or 128. The sources mention sampling $n$ values [S1].
*   **Grid Sizes:** Proposal NGP hierarchy truncated at maximum grid sizes of 512 and 2048 [S2].
*   **Channels:** Single channel for proposal NGP [S2].

### Edge Cases and Pitfalls
*   **Aliasing:** Using sampled points directly as endpoints can "erode" the coarse histogram because samples are unlikely to span the extent of each coarse bin, leading to aliasing [S1]. The midpoint resampling method mitigates this [S1].
*   **Boundary Conditions:** Without reflecting samples, the first and last intervals may not be properly bounded, leading to artifacts at the scene edges [S1].

## Operating characteristics
*   **Time Complexity:** The resampling step is $O(n \log n)$ due to sorting, where $n$ is the number of samples. The subsequent rendering is proportional to the number of fine intervals.
*   **Memory Complexity:** The proposal networks use truncated hash grids (max sizes 512 and 2048) and single-channel features, reducing memory usage compared to full-resolution NGP [S2].
*   **Strengths:** Qualitatively reduces aliasing artifacts [S1]. Efficiently bounds scene geometry through recursive proposal sampling [S2].
*   **Limitations:** Requires two rounds of proposal sampling before the final render, increasing training time per iteration [S2].
*   **When to use:** When training unbounded, anti-aliased neural radiance fields where aliasing is a concern [S1].

## Background
Mip-NeRF 360 Proposal Sampling was introduced in the Mip-NeRF 360 paper [S1]. It was later adopted and adapted in Zip-NeRF, which uses two rounds of proposal sampling to bound scene geometry and recursively generate new sample intervals [S2]. The method addresses the limitations of standard hierarchical sampling by modifying the resampling procedure to reduce aliasing [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
