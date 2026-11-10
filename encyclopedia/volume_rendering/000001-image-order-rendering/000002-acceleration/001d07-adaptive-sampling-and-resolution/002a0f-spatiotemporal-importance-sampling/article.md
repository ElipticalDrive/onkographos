# Spatiotemporal Importance Sampling

Also called: Spatiotemporal importance sampling strategy

## Overview
Spatiotemporal importance sampling is a strategy that assigns higher sampling probabilities to tissue pixels occluded by tools or exhibiting extensive motion ranges, and lower probabilities to stationary tissues, to improve convergence and efficiency [S1]. It addresses the challenge of tool occlusion and stationary tissue redundancy, which exacerbate learning complexity and reduce efficiency compared to uniform sampling [S1].

## How it works
**Inputs:**
- A sequence of $T$ frames, indexed $i=1 \dots T$.
- For each frame $i$, an image $I_i$ and a binary tool mask $M_i$ identifying occluded regions [S1].
- Hyper-parameters: $\alpha$ (lower-bound for weights), $\beta$ (balance factor), $n$ (temporal window radius) [S1].

**Outputs:**
- A set of sampling weight maps $\{W_i\}_{i=1}^T$, where each $W_i$ represents the sampling probability for each pixel/ray at time step $i$ [S1].

**Data Structures:**
- $I_i$: Image tensor for frame $i$.
- $M_i$: Binary mask tensor for frame $i$, same spatial dimensions as $I_i$.
- $W_i$: Weight map tensor for frame $i$, same spatial dimensions as $I_i$.
- $\Omega_i$: Occlusion frequency scaling tensor for frame $i$.

**Steps:**
1. Calculate the temporal difference metric for each pixel to identify regions with extensive motion. This involves comparing the current frame's masked image against neighboring frames within a temporal window $[i-n, i+n]$ [S1].
2. Compute the occlusion frequency scaling factor $\Omega_i$ for each frame based on the total occlusion count across the sequence [S1].
3. Combine the temporal difference metric and the occlusion scaling factor to generate the final sampling weight map $W_i$ [S1].

**Formulas:**
The sampling weight map $W_i$ is calculated as [S1]:

$$W_i = \min\left( \max_{\substack{i-n < j \\ < i+n}} \frac{\| I_i \odot M_i - I_j \odot M_j \|_1}{3}, \alpha \right) \odot \Omega_i$$

where:
- $\odot$ denotes element-wise multiplication.
- $\| \cdot \|_1$ denotes the L1 norm (sum of absolute differences) between the masked images of frame $i$ and frame $j$.
- The term $\max_{\substack{i-n < j \\ < i+n}}$ computes the maximum L1 difference between the current frame $i$ and any frame $j$ within the temporal window of radius $n$ [S1].
- $\alpha$ is a lower-bound parameter to avoid zero weight among unchanged pixels [S1].
- $\Omega_i$ is the occlusion frequency scaling factor, defined as [S1]:

$$\Omega_i = \beta \left( \frac{T M_i}{\sum_{k=1}^T M_k} \right)$$

where:
- $T$ is the total number of frames.
- $\sum_{k=1}^T M_k$ is the sum of the binary masks across all frames, representing the total occlusion count per pixel over the entire sequence.
- $\beta$ is a hyper-parameter for balancing augmentation among frequently occluded areas and time-variant areas [S1].

**Parameters:**
- $\alpha$: Lower-bound for the weight map to prevent zero weights for static pixels. Value not given in the sources.
- $\beta$: Hyper-parameter balancing augmentation between frequently occluded areas and time-variant areas. Value not given in the sources.
- $n$: Radius of the temporal window for comparing frames. Value not given in the sources.

**Edge Cases and Pitfalls:**
- If a pixel is never occluded, $M_i$ is zero for all $i$. The term $\frac{T M_i}{\sum_{k=1}^T M_k}$ becomes $0/0$ if no pixels are ever occluded, or $0$ if some are. The sources do not specify a guard for the case where $\sum_{k=1}^T M_k = 0$ [S1].
- The division by 3 in the L1 norm term suggests the images might be normalized to $[0,1]$ or $[0,255]$, but the exact scaling factor's origin is not explained beyond the formula [S1].

## Operating characteristics
Not given in the sources.

## Background
The method was introduced in the context of efficient deformable tissue reconstruction via orthogonal neural planes for surgical scenes [S1]. It draws inspiration from EndoNeRF [S1]. It is designed to handle the specific challenges of robotic surgery scenes, where tool occlusion hides tissues and stationary tissues provide redundant training data [S1].

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
