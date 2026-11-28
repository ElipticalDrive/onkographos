# Pixel Region and Depth Guided Ray Sampling


## Overview
Pixel Region and Depth Guided Ray Sampling is a ray sampling strategy that increases sampling probability in pixel areas exhibiting significant color and depth variation, reducing redundant rays in flat regions to accelerate network convergence and improve geometry perception [S1]. Unlike conventional NeRF which uses uniform distribution, this method samples varying ray densities based on depth boundaries and pixel regions to simultaneously accelerate convergence and improve geometry perception [S1]. It builds on [Depth-Guided Sampling](7490-depth-guided-sampling.md) by explicitly incorporating color variation alongside depth estimates.

## How it works
**Inputs:**
*   $N$ training source images $I_i$ with height $H_i$ and width $W_i$ [S1].
*   A neural radiance field network $F_\Theta$ parameterized by $\Theta$, mapping 3D position $p$ and viewing direction $d$ to volume density $\sigma$ and emitted color $c$ [S1].
*   Estimated depth information for each pixel, obtained from the ENeRF cost volume [S1].
*   True pixel colors $c_i^k$ [S1].

**Outputs:**
*   A set of sampled rays $\mathbb{R}_i$ with non-uniform density per image [S1].
*   Rendered pixel colors $V_\Theta(r_i^k)$ [S1].

**Data Structures:**
*   **Sampling Probability Map:** A 2D array of size $H_i \times W_i$ storing the probability $P(u, v)$ for each pixel $(u, v)$ to be sampled [S1].
*   **Cost Volume:** A 3D tensor used to derive depth estimates, as utilized in the ENeRF framework [S1].

**Steps:**
1.  **Depth Estimation:** Obtain rough scene depth using the depth estimation module from the ENeRF framework, leveraging the cost volume to obtain depth information for each pixel $(u, v)$ [S1].
2.  **Probability Calculation:** Calculate the sampling probability for each pixel $(u, v)$ based on its color value and the estimated depth information [S1].
3.  **Ray Sampling:** Sample additional rays in pixel areas exhibiting substantial color variation from the source view images [S1]. Similarly, guided by the depth-based strategy, sample more rays in regions where geometric depth varies markedly [S1]. This replaces the uniform distribution $r_i(u, v) \sim U(I_i)$ [S1].
4.  **Volume Rendering:** Perform volume rendering using the network to output colors and densities along the sampled rays to render the final pixel color [S1].
5.  **Optimization:** Optimize the network parameters $\Theta$ by minimizing the volume rendering loss [S1].

**Formulas:**
The optimization goal minimizes the volume rendering loss [S1]:
$$ \min_{\Theta} \mathcal{L} = \frac{1}{NM} \sum_{r_i^k \in \mathbb{R}_i} \|V_{\Theta}(r_i^k) - c_i^k\|_2^2 $$
where:
*   $c_i^k$ is the true color of pixels [S1].
*   $M$ is the number of sampled rays per image [S1].
*   $N$ is the number of trained images [S1].
*   $r_i^k$ is the $k$-th ray from the $i$-th training image [S1].
*   $V_{\Theta}$ is the volume rendering function [S1].

Uniform ray sampling assumes rays follow a uniform distribution, which this method replaces [S1]:
$$ r_i(u, v) \sim U(I_i), \quad u \in [0, H_i], \quad v \in [0, W_i] $$
where $H_i$ and $W_i$ are the height and width of image $I_i$ [S1].

**Parameters:**
*   $N$: The number of trained images (source views) [S1].
*   $M$: The number of sampled rays per image [S1].
*   $\Theta$: The network parameters optimized by minimizing the color error [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the ENeRF cost volume for depth estimation [S1].
*   **Boundaries:** Flat regions with no variation receive fewer rays to reduce redundancy, requiring the remaining samples to maintain global scene coherence [S1].
*   **Degenerate Input:** If the cost volume is ambiguous, the depth-guided component may provide misleading probabilities [S1].

## Subtopics

### Superpixel-based Efficient Sampling
A sampling method for learning neural fields from large input images that optimizes pixel-level ray sampling by segmenting the error map into superpixels and dynamically updating their errors to increase ray sampling in areas with higher rendering errors [S2]. It segments the input image into various superpixels using the SLIC algorithm, treating superpixels as the smallest processing unit [S2]. It sets different sampling probabilities for various superpixels based on their rendering errors and continuously updates them during training [S2]. Unlike quadtree or fixed-shape patch sampling methods which may cause fragmented image information, superpixels reduce redundancy while considering local information [S2]. Unlike color-guided ray sampling which does not simultaneously consider rendering errors, SES uses rendering errors to guide sampling [S2].

## Operating characteristics
*   **Time and Memory Complexity:** The method reduces redundant rays, accelerating network convergence and reducing training time [S1]. It achieves the same convergence effect in minutes through fine-tuning reconstructions from given views [S1].
*   **Measured Performance:** Applied to the ENeRF framework, it achieves state-of-the-art performance on both view synthesis and training time [S1].
*   **Scaling:** The strategy directly transfers to existing NeRF models because ray sampling is ubiquitous in NeRF [S1].
*   **Strengths:** Accelerates per-scene optimization, improves efficiency, and better perceives spatial geometry, especially for rich-texture regions [S1].
*   **Limitations:** Current acceleration methods predominantly focus on static scenes [S1].
*   **When to use:** When faster per-scene optimization is desired and the scene contains rich textures or significant depth variations [S1].
*   **When not:** When uniform coverage is strictly required or when depth estimation is unreliable [S1].

## Background
The method was introduced in the paper "Efficient ray sampling for radiance fields reconstruction" [S1]. It addresses the inefficiency of uniform ray sampling in NeRF, which fails to capture the non-uniform distribution of color differences and contextual information in images [S1]. The authors hypothesize that training with fewer, more informative rays can accelerate scene fitting [S1]. The method is applied to the ENeRF framework [15] to demonstrate its effectiveness [S1].

## Sources
[S1] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
[S2] Superpixel-based Efficient Sampling for Learning Neural Fields from Large Input, https://doi.org/10.1145/3664647.3681299, fetched 2026-11-08
