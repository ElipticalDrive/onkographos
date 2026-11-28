# Multi-Scale Entropy for View Quality

Also called: Multi resolution entropy, Wavelet entropy

## Overview
Multi-Scale Entropy for View Quality is a reference-free metric that quantifies the visual information content of a rendered volume image by computing the Shannon entropy of wavelet coefficients across multiple resolution levels [S1]. It addresses the limitation of standard histogram-based entropy, which ignores spatial pixel correlations, by capturing structural details and local correlations inherent in volume rendering [S1]. Unlike its sibling [Image Entropy (H)](15295-reference-free-image-quality-metrics.md), which relies solely on pixel value histograms, this method uses wavelet transforms to measure information at different scales, making it more sensitive to the structural complexity of the rendered view [S1].

## How it works
**Inputs:** A rendered RGB image $X$ of size $W \times H$, encoded with 8 bits per color channel [S1].
**Outputs:** A scalar value $H_W(X)$ representing the multi-scale entropy of the image [S1].

**Data Structures:**
*   **Image $X$:** A 2D array of pixel values. Each pixel has three components $(R, G, B)$, where each component is an integer in $[0, 255]$ [G].
*   **Wavelet Coefficients:** The result of applying the Haar wavelet transform to each color channel separately [S1].

**Parameters:**
*   **Wavelet Type:** Haar wavelet transform [S1].
*   **Decomposition Levels ($L$):** Typically 4 levels are sufficient [S1].
*   **Color Channels:** Red, Green, Blue (processed independently) [S1].

**Steps:**
1.  **Decomposition:** Apply the Haar wavelet transform to each of the three color channels (R, G, B) of the input image $X$ independently [S1]. This produces a set of wavelet coefficients for each level $l$ and channel $c \in \{R, G, B\}$ [S1]. Let $w_{l,k}^{(c)}$ denote the $k$-th wavelet coefficient at level $l$ for channel $c$ [G].
2.  **Iteration:** For each decomposition level $l$ from $1$ to $L$:
    a.  For each color channel $c \in \{R, G, B\}$:
        i.  Compute the histogram of the wavelet coefficients $w_{l,k}^{(c)}$ for all $k$ at this level [G].
        ii. Calculate the probability $p(w_{l,k}^{(c)})$ for each unique coefficient value $k$ as the relative frequency of that value in the histogram [S1].
        iii. Compute the local entropy contribution $h_{RGB}(w_{l,k}^{(c)})$ for each unique value using the formula:
        $$ h_{RGB}(w_{l,k}^{(c)}) = -p(w_{l,k}^{(c)}) \log_2 p(w_{l,k}^{(c)}) $$
        Note: If $p(w_{l,k}^{(c)}) = 0$, the term is defined as 0 [S1].
        iv. Sum the contributions for all unique coefficient values $k$ at level $l$ for channel $c$:
        $$ H_l^{(c)} = \sum_{k} h_{RGB}(w_{l,k}^{(c)}) $$
    b.  Sum the entropy contributions across all three channels for level $l$:
        $$ H_l = H_l^{(R)} + H_l^{(G)} + H_l^{(B)} $$
3.  **Aggregation:** Sum the entropy contributions across all $L$ levels to obtain the final Multi-Scale Entropy:
    $$ H_W(X) = \sum_{l=1}^{L} H_l $$
    Or equivalently, summing over all levels and all coefficients directly:
    $$ H_W(X) = - \sum_{l=1}^{L} \sum_{k=0}^{N_l} h_{RGB}(w_{l,k}) $$
    where $L$ is the number of wavelet levels, $N_l$ is the number of coefficients at level $l$, $w_{l,k}$ are wavelet coefficients, and $h_{RGB}(w_{l,k}) = -p(w_{l,k}) \log p(w_{l,k})$ with $p(w_{l,k})$ being the relative frequency of coefficient value $k$ in channel RGB at level $l$ [S1].

**Edge Cases and Pitfalls:**
*   **Zero Probability:** The term $0 \log 0$ is defined as 0 to ensure continuity [S1].
*   **Resolution:** The method assumes the image has sufficient resolution to support $L$ levels of decomposition. If the image is too small, fewer levels may be used, or the method may fail [G].
*   **Color Independence:** The metric treats color channels independently, as $h_{RGB}$ means the entropy is measured separately for each RGB channel [S1].

## Subtopics

### [Adaptive Best View Selection](17096-adaptive-best-view-selection.md)
An algorithm that uses Multi-Scale Entropy to efficiently find the optimal viewpoint for a volume model by restricting the search to a bounding sphere and using adaptive subdivision, avoiding the cost of exhaustive search.

### Viewpoint Entropy Evaluation Function
A metric to evaluate the quality of a viewpoint in volume rendering by calculating the information-theoretic entropy of the resulting 2D projective image [S2]. Higher entropy indicates more abundant visual information and better viewpoint quality [S2].

## Operating characteristics
*   **Time Complexity:** A brute-force evaluation of a dense set of views takes roughly half a minute, which is too slow for interactive previsualization [S1]. The computational cost is multiplied by the number of views evaluated [S1].
*   **Memory Complexity:** Requires storage for the image and the wavelet coefficients at each level [G].
*   **Strengths:** Captures structural information and pixel correlations that standard entropy misses [S1]. It correlates well with the amount of visual information revealed by illumination in volume rendering [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When evaluating the structural complexity of volume rendering views without a reference image, particularly for selecting representative views [S1].
*   **When not to use:** Not given in the sources.

## Background
The concept of multi-resolution entropy was introduced by Starck et al. [S1]. It was applied to volume rendering in the paper "Representative Views and Paths for Volume Models" [S1]. The authors found that standard Shannon entropy, which relies on pixel histograms, is insensitive to pixel correlation [S1]. By using wavelet coefficients, the metric better analyzes the structures given a transfer function (TF) in volume rendering [S1]. A user study cited in [S1] showed that Multi-Scale Entropy was a good approximation to the amount of information revealed by illumination. The method is suitable for volume rendering analysis because it reasonably analyzes the structures provided by the transfer function [S1].

## Sources
[S1] Representative Views and Paths for Volume Models, https://doi.org/10.1007/978-3-540-85412-8_10, fetched 2026-10-26
[S2] Viewpoint Selection Using Hybrid Simplex Search and Particle Swarm Optimization for Volume Rendering, https://doi.org/10.5815/ijigsp.2012.09.03, fetched 2026-10-26
