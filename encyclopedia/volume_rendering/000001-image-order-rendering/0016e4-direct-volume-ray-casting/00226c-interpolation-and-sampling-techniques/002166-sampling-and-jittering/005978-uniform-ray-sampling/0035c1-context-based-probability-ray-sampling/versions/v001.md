# Context-based Probability Ray Sampling

Also called: Context-based probability distribution sampling, Probability-based sampling function, context-based ray allocation, probability sampling

## Overview
Context-based Probability Ray Sampling is a non-uniform ray sampling strategy that allocates more rays to image regions with high visual complexity (edges, texture) and fewer rays to trivial regions (uniform backgrounds) [S1]. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by using local image statistics to guide ray density, and from [Pixel-guided ray sampling](7551-pixel-guided-ray-sampling.md) by relying on pre-computed spatial variance rather than learned or iterative error metrics.

## How it works
**Inputs:**
- Input image $I_i$ with height $H_i$ and width $W_i$.
- Color channel data $c(x, y)$ for each pixel.
- Threshold parameter $s$.

**Outputs:**
- A set of rays $r_i(u, v)$ sampled from pixel coordinates $(u, v)$ according to a probability distribution.

**Data Structures:**
- A 2D probability map $g'(u, v)$ of size $H_i \times W_i$, normalized to $[0, 1]$.
- A cumulative distribution function (CDF) derived from $g'(u, v)$ for sampling.

**Steps:**
1. **Compute Local Mean Color:** For each pixel $(u, v)$, calculate the mean color $\bar{c}$ of the $3 \times 3$ patch centered at $(u, v)$. The patch includes pixels $(x, y)$ where $x \in \{u-1, u, u+1\}$ and $y \in \{v-1, v, v+1\}$ [S1].
   $$ \bar{c} = \frac{1}{9} \sum_{x,y} c(x, y) $$
   *Note: Boundary handling for pixels near image edges is not specified in the source [G]. A common implementation is zero-padding or reflection padding.*

2. **Compute Standard Deviation (Context Metric):** Calculate the standard deviation $g(u, v)$ of the color values within the same $3 \times 3$ patch [S1].
   $$ g(u, v) = \sqrt{\frac{1}{9} \sum_{x,y} [c(x, y) - \bar{c}]^2} $$
   This metric represents the local color variance. High values indicate edges or texture; low values indicate uniform areas [S1].

3. **Normalize and Clamp:** Normalize $g(u, v)$ to the interval $[0, 1]$ and apply a lower-bound clamp to prevent trivial areas from receiving zero rays [S1].
   $$ g'(u, v) = \frac{\text{clamp}(s, \max(g))}{\max(g)} $$
   Where $\text{clamp}(s, \max(g))$ replaces any value in $g(u, v)$ less than $s$ with $s$. The result is then divided by the global maximum of $g$ [S1].

4. **Sample Rays:** Generate rays $r_i(u, v)$ for pixel $(u, v)$ by sampling from the distribution $g'(u, v)$ [S1]. In practice, this involves constructing a CDF from $g'$ and using inverse transform sampling or stratified sampling to select ray origins or densities [G].

**Parameters:**
- **Threshold $s$:** Prevents division by zero and ensures minimum ray allocation. Typically $s = 0.01 \times \text{mean}(g(u, v))$ [S1].
- **Patch Size:** The spatial neighborhood scale. The source primarily uses $3 \times 3$ but tests $5 \times 5, 7 \times 7, 9 \times 9, 11 \times 11$ [S1].
- **Context Metric:** The source uses Standard Deviation. Alternatives tested include Entropy and Variance [S1].

**Edge Cases and Pitfalls:**
- **Boundary Pixels:** The formula requires neighbors outside the image for edge pixels. The source does not specify padding; implementers must choose a strategy (e.g., zero, replicate, or ignore) [G].
- **Uniform Regions:** Without clamping $s$, uniform regions (e.g., white walls) would have $g(u, v) \approx 0$, leading to no rays being sampled and potential training instability [S1].
- **Metric Choice:** Standard Deviation is sensitive to outliers. Entropy and Variance were tested as alternatives but Standard Deviation was selected for the final method [S1].

## Operating characteristics
- **Time Complexity:** $O(H \times W \times K^2)$ to compute the probability map, where $K$ is the patch size (typically 3). Sampling is $O(N)$ where $N$ is the number of rays.
- **Memory Complexity:** $O(H \times W)$ to store the probability map $g'(u, v)$ [S1].
- **Performance:** Reduces training time by sampling fewer rays in trivial areas while maintaining accuracy in complex regions [S1]. Evaluated on Plenoxels, it significantly improves performance over methods without probability sampling [S1].
- **Strengths:** Simple to implement; leverages existing image data; reduces computational load in low-information areas.
- **Limitations:** Requires access to the input image context; may not adapt to 3D geometry changes that do not affect 2D image variance; boundary handling is ambiguous.

## Background
Originated in the paper "Fast Learning Radiance Fields by Shooting Much Fewer Rays" (2023) [S1]. The method was evaluated as an ablation study on the Plenoxels radiance field method to demonstrate the benefit of context-aware ray allocation [S1]. It addresses the inefficiency of uniform sampling in neural radiance fields by recognizing that not all pixels contribute equally to the reconstruction of complex 3D structures [S1].

## Sources
[S1] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
