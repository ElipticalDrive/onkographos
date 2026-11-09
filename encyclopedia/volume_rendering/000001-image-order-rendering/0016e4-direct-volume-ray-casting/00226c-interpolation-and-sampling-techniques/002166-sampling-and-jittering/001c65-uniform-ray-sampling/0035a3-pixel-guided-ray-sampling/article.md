# Pixel-guided ray sampling
Also called: Color-guided sampling, Pc sampling

## Overview
Pixel-guided ray sampling is a non-uniform sampling strategy that calculates a prior probability distribution based on the color variation (standard deviation) within pixel neighborhoods to identify high-information regions [S1]. It solves the problem of inefficient ray casting in uniform sampling by concentrating computational resources on pixels with sharp color differences, which typically correspond to rich texture details or sharp density variations in the scene [S1].

## How it works
**Inputs:**
- Source view image $I_i$ with dimensions $H_i \times W_i$ [S1].
- Pixel coordinates $(u, v)$ where $u \in [0, H_i]$ and $v \in [0, W_i]$ [S1].
- Neighborhood size parameter $n$ (default 3) [S1].
- Threshold scaling factor $\alpha = 0.01$ [S1].

**Outputs:**
- A set of sampled rays $r_i(u, v)$ for the image, where the probability of selecting a ray at $(u, v)$ is proportional to the computed probability density $P'_c(u, v)$ [S1].

**Data Structures:**
- The probability map $P_c$ is a 2D array of size $H_i \times W_i$ storing the raw standard deviation values [S1].
- The normalized probability map $P'_c$ is a 2D array of size $H_i \times W_i$ storing the final sampling weights [S1].

**Steps:**
1. For each pixel $(u, v)$ in image $I_i$, compute the mean color $\bar{c}$ of the $n \times n$ neighborhood centered at $(u, v)$ [S1].
2. Compute $P_c(u, v)$ as the standard deviation of colors in that neighborhood [S1].
3. Find the maximum value $\max(P_c(u, v))$ across the image [S1].
4. Calculate threshold $s = 0.01 \times \text{mean}(P_c)$, where $\text{mean}(P_c)$ is the average of all $P_c$ values across the image [S1].
5. Apply normalization: $P'_c(u, v) = \text{clamp}(P_c(u, v), s, \max(P_c(u, v))) / \max(P_c(u, v))$ [S1].
6. Sample rays $r_i(u, v)$ from pixel coordinates $(u, v)$ according to the probability distribution $P'_c(u, v)$ [S1].

**Formulas:**
- Raw probability calculation: $P_c(u, v) = \text{std}(c(u, v)) = \sqrt{\frac{1}{n^2} \sum_{x', y'} [c(x', y') - \bar{c}]^2}$, where $c(u,v)$ is the pixel color, $n$ is the neighborhood size (default 3), $x' \in [u - \frac{n-1}{2}, u + \frac{n-1}{2}]$, $y' \in [v - \frac{n-1}{2}, v + \frac{n-1}{2}]$, and $\bar{c}$ is the mean color of the $n \times n$ neighborhood [S1].
- Computational implementation using variance identity: $\text{std}(c(u, v)) = \sqrt{E(c(x', y')^2) - E(c(x', y'))^2}$ [S1]. Note: The source text contains a typo in the quote 'E(c(x', y')^2)' repeated, but the standard identity implies subtracting the square of the mean [S1].
- Normalization to prevent underfitting: $P'_c(u, v) = \frac{\text{clamp}(P_c(u, v), s, \max(P_c(u, v)))}{\max(P_c(u, v))}$, where $s = 0.01 \times \text{mean}(P_c)$ [S1]. Values below $s$ are clamped to $s$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixel_guided_ray_sampling(image: list, n: int = 3, alpha: float = 0.01) -> list:
    """
    Computes the normalized probability distribution for pixel-guided ray sampling.
    
    Args:
        image: 2D list of pixel colors (H, W, C).
        n: Neighborhood size (default 3).
        alpha: Threshold scaling factor (default 0.01).
        
    Returns:
        P_prime_c: Normalized probability distribution (H, W).
    """
    H = len(image)
    W = len(image[0])
    C = len(image[0][0])
    half_n = (n - 1) / 2.0
    
    mean_color = [[0.0] * C for _ in range(H)]
    mean_sq_color = [[0.0] * C for _ in range(H)]
    
    for u in range(H):
        for v in range(W):
            u_start = int(max(0, u - half_n))
            u_end = int(min(H, u + half_n + 1))
            v_start = int(max(0, v - half_n))
            v_end = int(min(W, v + half_n + 1))
            
            count = (u_end - u_start) * (v_end - v_start)
            sum_c = [0.0] * C
            sum_sq = [0.0] * C
            
            for i in range(u_start, u_end):
                for j in range(v_start, v_end):
                    for k in range(C):
                        val = image[i][j][k]
                        sum_c[k] += val
                        sum_sq[k] += val * val
            
            for k in range(C):
                mean_color[u][v][k] = sum_c[k] / count
                mean_sq_color[u][v][k] = sum_sq[k] / count
                
    Pc = [[0.0] * W for _ in range(H)]
    for u in range(H):
        for v in range(W):
            var_sum = 0.0
            for k in range(C):
                var = mean_sq_color[u][v][k] - mean_color[u][v][k] ** 2
                var_sum += max(var, 0.0)
            Pc[u][v] = var_sum ** 0.5
            
    max_Pc = max(max(row) for row in Pc)
    mean_Pc = sum(sum(row) for row in Pc) / (H * W)
    s = alpha * mean_Pc
    
    P_prime_c = [[0.0] * W for _ in range(H)]
    for u in range(H):
        for v in range(W):
            val = Pc[u][v]
            if val < s: val = s
            if val > max_Pc: val = max_Pc
            P_prime_c[u][v] = val / max_Pc
            
    return P_prime_c
```

**Parameters:**
- $n$: The size of the pixel neighborhood for computing standard deviation. Default value: 3 [S1].
- $\alpha$: The scaling factor for the lower bound threshold $s$. Default value: 0.01 [S1].

**Edge Cases and Pitfalls:**
- **Boundary Handling:** Not given in the sources. Implementers typically clamp indices to image bounds [G].
- **Zero Variance:** If a pixel and its neighborhood have identical colors, $P_c(u, v) = 0$. The clamping step ensures this value is raised to $s$, preventing zero probability [S1].
- **Precision:** The variance identity $E(c^2) - E(c)^2$ can suffer from catastrophic cancellation if the color values are large and the variance is small. Using `float64` for intermediate calculations is recommended [G].

## Subtopics

### Depth-guided ray sampling
A ray sampling strategy that uses the standard deviation of depth values in pixel neighborhoods to guide ray sampling, targeting areas with abrupt depth changes [S1]. It obtains the depth probability volume $P_i(u, v)$ from an ENeRF coarse-to-fine approach, computes the expected depth $D(u, v) = \sum_{i=1}^{N_d} P_i(u, v) \cdot L_i(u, v)$ where $L_i$ are depth planes sampled from $[L_{min}, L_{max}]$ and $N_d$ is the number of sampled depth planes [S1]. It then computes $P_d(u, v) = \text{std}(D(u, v)) = \sqrt{\frac{1}{n^2} \sum_{x', y'} [D(x', y') - \bar{D}]^2}$ over an $n \times n$ neighborhood, applies the same normalization as pixel-guided sampling to get $P'_d(u, v)$, and fuses it with the color-guided distribution via $P_{samp}(u, v) = \beta P'_c(u, v) + (1 - \beta)P'_d(u, v)$ during training [S1].

## Operating characteristics
**Time Complexity:** $O(H \cdot W \cdot n^2)$ for computing the probability map, where $H$ and $W$ are image dimensions and $n$ is the neighborhood size. Sampling rays is $O(H \cdot W)$ assuming efficient sampling from the distribution [G].
**Memory Complexity:** $O(H \cdot W)$ to store the probability maps $P_c$ and $P'_c$ [G].
**Strengths:** Focuses computational resources on pixels with sharp color differences, which correspond to rich texture details or sharp density variations in the scene [S1].
**Limitations:** Requires an initial pass to compute statistics [S1].
**When to use:** When training radiance fields or volume renderers where computational efficiency is critical and the scene has significant texture variation [S1].
**When not to use:** Not given in the sources.

## Background
Pixel-guided ray sampling was introduced in the context of efficient ray sampling for radiance fields reconstruction [S1]. It is based on the intuition that pixels with identical colors to their neighborhood contain less information, while those with substantial color differences encompass more scene details [S1]. The method replaces the uniform distribution of sampled rays during training with a probability density function derived from color variation [S1].

## Sources
[S1] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
