# Hierarchical Stratified Sampling

Also called: Importance Stratified Sampling

## Overview
Hierarchical Stratified Sampling divides a camera ray into equally spaced bins, draws uniform samples from each, and adaptively refines regions that contributed to the final color using a probabilistic function [S1]. It builds on [Stratified Ray Sampling](3014-stratified-ray-sampling.md) by adding an importance-driven refinement layer to improve reconstruction quality and prevent overfitting [S1].

## How it works
**Inputs:** Camera ray geometry (origin, direction, $t_{min}$, $t_{max}$), a volume representation, and a target number of initial strata $N$.
**Outputs:** A sorted list of sample positions $t$ along the ray used to predict the pixel color [S1].

**Data Structures:**
1. **Ray Data Structure:** One instance per camera ray tracking segment locations where the importance value changes [S2].
2. **Sample List:** A sorted list of sample positions $t$ and their corresponding contributions [S2].

**Steps:**
1. **Initial Stratified Sampling:** Divide the camera ray into $N$ equally spaced bins. Draw one sample uniformly at random from each bin [S1].
2. **Contribution Analysis:** Render the scene using the initial samples to identify which samples contributed to the final color [S1].
3. **Adaptive Refinement:** Sample more points around the high-contribution regions using a probabilistic function to determine the expected color value along the ray [S1]. Maintain one data structure per ray and alternate between uniform strata sampling and adaptive sampling in subsequent progressive rendering passes [S2]. Adaptive sampling targets non-sampled regions between segments, within segments to detect splits, and non-sampled regions close to the begin and end of the ray [S2].
4. **Aggregation:** Combine the initial stratified samples and the refined adaptive samples to predict the pixel color [S1].

**Parameters:**
* $N$: Number of initial strata. Typical values are not specified in the sources [S1].
* Number of adaptive passes: In experiments, one uniform sampling pass followed by three adaptive sampling steps yielded extremely fast accuracy increases [S2].

**Edge Cases and Pitfalls:**
* **Overfitting:** Using stratified samples generally improves reconstructions by helping to prevent overfitting [S1].
* **View Dependency:** NeRFs may produce incorrect depth maps if they rely on all spatial points behind a reflecting surface rather than explaining view-dependent appearance directly [S1].
* **Precision:** The accuracy of the estimated integral $\tilde{k}(t)$ depends on the number of adaptive steps [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

def hierarchical_stratified_sampling(ray_min_t, ray_max_t, num_initial_strata=64, num_adaptive_passes=3):
    # Step 1: Divide the camera ray into equally spaced bins and draw a uniform sample from each bin [S1]
    bin_size = (ray_max_t - ray_min_t) / num_initial_strata
    samples = []
    for i in range(num_initial_strata):
        t = ray_min_t + i * bin_size + random.uniform(0, bin_size)
        samples.append(t)

    # Step 2-3: Maintain one data structure per camera ray [S2]
    # Alternate between uniform strata sampling and adaptive sampling in progressive passes [S2]
    for _ in range(num_adaptive_passes):
        # Identify which samples contributed to the final color [S1]
        # Sample more points around those regions using a probabilistic function to determine the expected color value along the ray [S1]
        # Adaptive sampling targets: non-sampled regions between segments, within segments to detect splits, and non-sampled regions close to the begin and end of the ray [S2]
        new_samples = []
        # The sources do not specify the exact probabilistic function or segment tracking algorithm [G]
        # Standard implementation would evaluate transmittance/opacity and insert samples based on importance gradients
        for t in samples:
            new_samples.append(t + random.uniform(-0.05, 0.05))
        samples.extend(new_samples)
        samples.sort()

    # Step 4: Aggregate samples to predict the pixel color [S1]
    # The exact aggregation weights are not specified in the sources [G]
    return samples
```

## Subtopics

### Stratified Adaptive Ray Sampling
This variant maintains one data structure per camera ray and uses stratified sampling by dividing the ray into $N$ equally-sized regions to obtain a sorted list of new samples [S2]. It alternates between uniform strata sampling and adaptive sampling in subsequent progressive rendering passes [S2]. Adaptive sampling targets non-sampled regions between segments, within segments to detect splits, and non-sampled regions close to the begin and end of the ray [S2]. In experiments, accuracy of $\tilde{k}(t)$ increased extremely fast when performing one uniform sampling, followed by three adaptive sampling steps in subsequent progressive rendering passes [S2].

## Operating characteristics
* **Time Complexity:** Not given in the sources. The method requires multiple progressive rendering passes, with experimental results showing fast convergence after one uniform pass and three adaptive passes [S2].
* **Memory Complexity:** Not given in the sources. Requires storing one data structure per camera ray and a sorted list of samples [S2].
* **Strengths:** Reduces overfitting and improves reconstruction quality [S1]. Allows for progressive refinement and fast accuracy increases in adaptive passes [S2].
* **Limitations:** More complex to implement than uniform or simple stratified sampling. Requires multiple passes for optimal results [S2].
* **When to Use:** When high-quality reconstruction is needed and computational resources allow for multiple sampling passes [S1, S2].

## Background
Hierarchical Stratified Sampling was introduced in the context of 3D geometric reconstruction and evaluation of Neural Radiance Fields (NeRFs) [S1]. It addresses the limitations of NeRFs in handling view-dependent appearance and incorrect depth maps by leveraging hierarchical sampling strategies [S1]. The method has been further developed in the context of view-dependent visibility optimization for Monte Carlo volume visualization [S2].

## Sources
[S1] Vision through Obstacles—3D Geometric Reconstruction and Evaluation of Neural Radiance Fields (NeRFs), https://doi.org/10.3390/rs16071188, fetched 2026-10-30
[S2] View‐Dependent Visibility Optimization for Monte Carlo Volume Visualization, https://doi.org/10.1111/cgf.70064, fetched 2026-11-05
