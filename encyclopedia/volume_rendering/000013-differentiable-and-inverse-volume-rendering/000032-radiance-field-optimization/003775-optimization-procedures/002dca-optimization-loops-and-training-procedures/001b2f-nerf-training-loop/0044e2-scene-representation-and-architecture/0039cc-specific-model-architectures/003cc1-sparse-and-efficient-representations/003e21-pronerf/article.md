# ProNeRF

Also called: ProNeRF-12, ProNeRF-32, ProNeRF-8, Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields

## Overview
ProNeRF is a few-point volumetric rendering method that uses a Projection-Aware Sampling (PAS) network to predict sparse 3D sampling points and opacity adjustments, allowing high-quality rendering with significantly fewer samples than standard NeRF [S1]. It achieves faster inference than vanilla NeRF with higher quality than HyperReel [S1]. Unlike generalizable radiance fields that store all training input images for epipolar feature projection, ProNeRF uses a fixed subset of reference views, eliminating the need for nearest-neighbor projection among all available training views [S1].

## How it works
ProNeRF performs volumetric rendering using a small number of samples $N_s$ (e.g., 8, 12, or 32) guided by a PAS network. The method consists of a PAS network (comprising MLP heads $F_{\theta_c}$ and $F_{\theta_f}$) and a shading network $F_{\theta_s}$ [S1].

### Inputs and Outputs
*   **Input:** Ray parameters $\mathbf{r} = (\mathbf{r}_o, \mathbf{r}_d)$ (origin and direction), a set of $N_n$ neighboring reference view colors $\{\mathbf{c}^k\}_{k=1}^{N_n}$ from a fixed pool of $N_t$ training views [S1].
*   **Output:** Rendered pixel color $\hat{\mathbf{c}}(\mathbf{r})$ [S1].

### Data Structures and Network Layout
1.  **Plücker Ray-Point Encoding:** The ray is encoded into a vector $\mathbf{r}_{pr}$ of size 48 using $N_{pr}=48$ points evenly spaced between near $t_n$ and far $t_f$ bounds [S1]:
    $$
    \mathbf{r}_{pr} = [\mathbf{r}_d, \mathbf{r}_o + \mathbf{r}_d \odot \mathbf{t}_{nf}, (\mathbf{r}_o + \mathbf{r}_d \odot \mathbf{t}_{nf}) \odot \mathbf{r}_d]
    $$
    where $\mathbf{t}_{nf}$ is a vector of $N_{pr}$ elements, $\odot$ is the Hadamard product, and $[\cdot, \cdot]$ is concatenation [S1].
2.  **Coarse MLP ($F_{\theta_c}$):** Consists of 6 fully connected layers with 256 neurons and ELU activations [S1]. It takes $\mathbf{r}_{pr}$ as input and outputs:
    *   Coarse sampling distances $\mathbf{T}' = \{t'_1, \dots, t'_{N_s}\}$ [S1].
    *   Opacity shifts $\Delta_t$ and scales $B_t$ [S1].
    *   Auxiliary coarse color $\hat{\mathbf{c}}_c$ [S1].
3.  **Color-to-Ray Projections:** For each coarse point $p_i = \mathbf{r}(t'_i)$, lines are traced to the camera centers of $N_n$ neighboring reference views. The colors at these intersections are collected into a projection vector $\mathbf{f}_{pi} = [\mathbf{c}^1_{pi}, \dots, \mathbf{c}^{N_n}_{pi}]$ [S1].
4.  **Fine MLP ($F_{\theta_f}$):** Consists of 6 fully connected layers with 256 neurons and ELU activations [S1]. It takes the encoded ray $\mathbf{r}'_{pr}$ and the aggregated projections $\{\mathbf{f}_{p1}, \dots, \mathbf{f}_{pN_s}\}$ as input. It outputs:
    *   Refinement weights $\Delta_T$ [S1].
    *   Auxiliary weights $W \in \mathbb{R}^{N_s}$ (softmax) and $M \in \mathbb{R}^{N_n}$ (sigmoid) [S1].
    *   Auxiliary fine color $\hat{\mathbf{c}}_f$ [S1].
5.  **Shading Network ($F_{\theta_s}$):** Consists of 8 fully connected layers with 256 neurons [S1]. It takes the refined 3D points $\mathbf{x}_i = \mathbf{r}_o + \mathbf{r}_d \odot T_i$ as input and outputs density $\sigma_i$ and color $\mathbf{c}_i$ [S1].

### Numbered Steps
1.  **Encode Ray:** Compute $\mathbf{r}_{pr}$ from $\mathbf{r}_o, \mathbf{r}_d$ and $\mathbf{t}_{nf}$ [S1].
2.  **Coarse Sampling:** Query $F_{\theta_c}(\mathbf{r}_{pr})$ to obtain $\mathbf{T}'$, $\Delta_t$, $B_t$, and $\hat{\mathbf{c}}_c$ [S1].
3.  **Projection:** For each $t'_i \in \mathbf{T}'$, compute $p_i = \mathbf{r}(t'_i)$. Trace rays from $p_i$ to the $N_n$ reference camera centers. Interpolate colors from the reference images to get $\mathbf{f}_{pi}$ [S1].
4.  **Refinement:** Query $F_{\theta_f}([\mathbf{r}'_{pr}, \mathbf{f}_{p1}, \dots, \mathbf{f}_{pN_s}])$ to obtain $\Delta_T$, $W$, $M$, and $\hat{\mathbf{c}}_f$ [S1].
5.  **Compute Refined Distances:** Linearly interpolate between consecutive elements of the expanded set $\dot{\mathbf{T}} = \{t_n, t'_1, \dots, t'_{N_s}, t_f\}$ using $\Delta_T$:
    $$
    T_i = \frac{1}{2} \left( (\dot{T}_i + \dot{T}_{i+1}) + \Delta_{T_i}(\dot{T}_{i+2} - \dot{T}_i) \right), \quad i=1,\dots,N_s
    $$
    where $\dot{T}$ indices are mapped appropriately to the $N_s$ intervals [S1].
6.  **Adjust Opacity:** Compute adjusted opacities $\hat{\alpha}_i$ using the original opacities $\alpha_i$ (derived from $\sigma_i$ and distances in $T$), shifts $\Delta_t$, and scales $B_t$:
    $$
    \hat{\alpha}_i = 1 - \exp(-(\sigma_i + \Delta_{t_i})(t_{i+1} - t_i) \cdot B_{t_i})
    $$
    [S1].
7.  **Render Color:** Compute final color using alpha compositing with adjusted opacities:
    $$
    \hat{\mathbf{c}}(\mathbf{r}) = \sum_{i=1}^{N_s} \left( \prod_{j=1}^{i-1} (1 - \hat{\alpha}_j) \right) \hat{\alpha}_i \mathbf{c}_i
    $$
    [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pronerf_render(ray_origin, ray_direction, ref_view_colors, ref_cameras):
    # 1. Encode Ray
    t_nf = linspace(t_n, t_f, N_pr)
    r_pr = concatenate([ray_direction, 
                        ray_origin + ray_direction * t_nf, 
                        (ray_origin + ray_direction * t_nf) * ray_direction])
    
    # 2. Coarse Sampling
    T_prime, delta_t, B_t, c_hat_c = F_theta_c(r_pr)
    
    # 3. Projections
    projections = []
    for t_prime in T_prime:
        p = ray_origin + ray_direction * t_prime
        view_colors = []
        for k in range(N_n):
            # Trace line from p to ref_cameras[k].center
            # Interpolate color from ref_view_colors[k] at intersection
            view_colors.append(interpolate_color(ref_view_colors[k], p, ref_cameras[k]))
        projections.append(view_colors)
        
    # 4. Refinement
    # Note: r'_pr is derived from r_pr and T_prime context, assumed available
    delta_T, W, M, c_hat_f = F_theta_f(concatenate([r_pr, projections]))
    
    # 5. Refined Distances
    T_dot = [t_n] + list(T_prime) + [t_f]
    T = []
    for i in range(N_s):
        # Linear interpolation with refinement weight
        t_i = 0.5 * (T_dot[i] + T_dot[i+1] + delta_T[i] * (T_dot[i+2] - T_dot[i]))
        T.append(t_i)
        
    # 6. Shading and Opacity Adjustment
    alphas = []
    colors = []
    for t in T:
        x = ray_origin + ray_direction * t
        sigma, c = F_theta_s(x)
        colors.append(c)
        # Compute base alpha
        delta_t_dist = T[(T.index(t)+1) % len(T)] - t if (T.index(t)+1) < len(T) else t_f - t
        # Note: In practice, T is sorted. Assuming T is sorted for delta calculation.
        # If T is not sorted, sort it or use adjacent indices carefully.
        # The formula uses t_{i+1} - t_i.
        # For the last point, delta is typically handled or ignored in sum.
        # Assuming standard NeRF sampling logic for delta.
        alpha = 1 - exp(-sigma * delta_t_dist)
        # Adjust opacity
        adj_alpha = 1 - exp(-(sigma + delta_t[i]) * delta_t_dist * B_t[i])
        alphas.append(adj_alpha)
        
    # 7. Compositing
    T_transmittance = 1.0
    c_final = zeros(3)
    for i in range(N_s):
        c_final += T_transmittance * alphas[i] * colors[i]
        T_transmittance *= (1 - alphas[i])
        
    return c_final, c_hat_c, c_hat_f
```

### Parameters
*   **$N_s$:** Number of samples. Typical values: 8, 12, 32 [S1].
*   **$N_{pr}$:** Number of Plücker points. Value: 48 [S1].
*   **$N$:** Maximum exploration samples. Value: 64 [S1].
*   **$N_n$:** Number of neighboring reference views. Significantly smaller than total training views $N_t$ [S1].
*   **Learning Rate:** $5 \times 10^{-4}$, exponentially decayed for 700,000 iterations [S1].
*   **Batch Size:** 4,096 rays [S1].
*   **$\lambda$:** Auxiliary loss weight. 1 for first 60% of training, 0 afterward [S1].
*   **Auxiliary Loss Duration:** First $4 \times 10^5$ iterations [S1].

### Edge Cases and Pitfalls
*   **Pruning:** ProNeRF-8 prune adds an MLP head $F_{\theta_m}$ (same complexity as $F_{\theta_c}$) to predict error between $\hat{\mathbf{c}}_f$ and $\hat{\mathbf{c}}$. If error is low, use $\hat{\mathbf{c}}_f$ (AVR); otherwise, run $F_{\theta_s}$ [S1]. This adds 3.3 MB memory overhead [S1].
*   **Auxiliary Loss:** Removing auxiliary loss $l_a$ causes ~1 dB PSNR drop [S1].
*   **Plücker Encoding:** Removing Plücker ray-point encoding causes ~0.5 dB PSNR drop [S1].

## Subtopics

### ProNeRF-8
8 samples. 26.78 dB PSNR on LLFF. >20x faster than NeRF [S1].

### ProNeRF-12
12 samples. 27.15 dB PSNR on LLFF. ~15x faster than NeRF [S1].

### ProNeRF-32
32 samples. 31.92 dB PSNR on Blender. 3.2 FPS after pruning [S1].

### ProNeRF-8 prune
Pruned version of ProNeRF-8. 23% faster than standard ProNeRF-8. 26.54 dB PSNR [S1].

## Operating characteristics
*   **Speed:** 15-23× faster than vanilla NeRF [S1]. ProNeRF-8 is >20x faster [S1]. ProNeRF-8 prune is 23% faster than ProNeRF-8 [S1].
*   **Quality:** 0.65 dB higher PSNR than NeRF on LLFF [S1]. 0.95 dB higher PSNR than HyperReel [S1].
*   **Memory:** Similar to NeRF. ProNeRF-8 prune adds 3.3 MB overhead [S1]. ProNeRF-32 uses 6.3 MB memory [S1].
*   **Strengths:** High quality with few samples; consistent memory footprint via fixed reference views; fast inference [S1].
*   **Limitations:** Requires training with auxiliary losses; pruning adds slight complexity [S1].
*   **When to use:** When rendering speed is critical but quality must exceed sampler-based methods like HyperReel, and memory is constrained [S1].
*   **When not to use:** When generalizable rendering across many scenes without per-scene training is required (ProNeRF is not generalizable like IBRNet) [S1].

## Background
ProNeRF was proposed in 2024 [S1]. It builds on the differentiable approximation of continuous radiance fields [Parent]. It addresses the quality degradation in few-point sampling methods [S1]. It is evaluated on LLFF and Blender datasets [S1].

## Sources
[S1] ProNeRF: Learning Efficient Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields, https://doi.org/10.1109/access.2024.3390753, fetched 2026-10-30
