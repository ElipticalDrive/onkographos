# Information Theory-Based Multimodal Transfer Function Design

Also called: Bramon et al. approach, IT-based TF design, IT-based multimodal TF design, Joint Information TF (Multimodal), Multimodal Statistical TF, Multimodal TF optimization

## Overview
An automated method for designing multimodal transfer functions for two registered volume datasets $X$ and $Y$, using mutual information decomposition to guide color fusion and opacity optimization [S1]. It solves the problem of combining complementary information from different modalities (e.g., CT and MR) into a single coherent visualization where both inputs are represented at each voxel, rather than discarding one [S1]. It sets itself apart from manual methods by automating the design and from single-modality optimization methods by defining an information channel between two input data sets and another between viewpoints and bins [S1].

## How it works
**Inputs:** Two registered volume datasets $X$ and $Y$ (intensity values), pre-defined 1D transfer functions $TF_X$ and $TF_Y$ (colors and opacities), and a target distribution definition [S1].
**Outputs:** A fused multimodal transfer function defining a single color $c(x,y)$ and opacity $\alpha(x,y,g)$ for every voxel based on its intensity pair $(x,y)$ and fused gradient $g$ [S1].

**Data Structures:**
1. **Intensity Bins:** The continuous intensity values of $X$ and $Y$ are discretized into bins $\mathcal{X}$ and $\mathcal{Y}$ using a non-regular binning algorithm [S1]. The joint space is $\mathcal{B} = \mathcal{X} \times \mathcal{Y}$ [S1].
2. **Information Maps:** Two 3D volumes per dataset, storing values $I_2(x;Y)$ and $I_3(x;Y)$ for each voxel, shifted so the minimum is 0 [S1].
3. **Joint Histogram:** A 3D histogram (or binned equivalent) counting occurrences of intensity pairs $(x,y)$ and potentially gradient values $g$ [S1].
4. **Visibility Distribution:** A probability distribution $p(b|v)$ or $p(b)$ over bins $\mathcal{B}$ for specific viewpoints or all viewpoints [S1].

**Steps:**

1. **Reference Selection:** Select the reference dataset (e.g., $X$) based on contrast [S1]. Compute the mean local variance of intensities in a small window for each dataset (using segmented volumes for normalization if available) [S1]. The dataset with the lower mean local variance (higher contrast) is chosen as the reference [S1].
2. **Gradient Computation:** Compute gradient magnitudes $g_X(i,j,k)$ and $g_Y(i,j,k)$ for both datasets using 4D linear regression (Neumann et al.) to avoid the inaccuracies of standard finite differences [S1].
3. **Information Map Generation:**
   - Compute the input distribution $p(X)$, conditional probability matrix $p(Y|X)$, and output distribution $p(Y)$ from the voxel intensities of the registered datasets [S1].
   - Calculate Predictability $I_2(x;Y)$ and Entanglement $I_3(x;Y)$ for each bin $x \in \mathcal{X}$:
     $$ I_2(x;Y) = -\sum_{y \in \mathcal{Y}} p(y) \log p(y) + \sum_{y \in \mathcal{Y}} p(y|x) \log p(y|x) $$
     $$ I_3(x;Y) = \sum_{y \in \mathcal{Y}} p(y|x) I_2(y;X) $$
   - Shift the values of $I_2$ and $I_3$ maps so that their minimum value is 0 to avoid negative weights [S1].
4. **Binning (One-Sided Clustering):**
   - Apply agglomerative information bottleneck clustering to reduce the number of bins in the reference dataset $X$ to a target count (default 32) [S1].
   - The clustering merges bins of $X$ by minimizing the loss of mutual information with $Y$ [S1]. The "control" dataset $Y$ determines the relevant structures in $X$ [S1].
   - Re-bin $Y$ and the gradient values accordingly to maintain the joint structure [S1].
5. **Color Fusion:**
   - For each voxel with intensities $x$ (from $X$) and $y$ (from $Y$), compute the fused color $c(x,y)$ [S1].
   - Using the hue-preserving operator $\oplus$ in HSL space (recommended for natural hue transitions):
     $$ c(x,y) = \frac{I_2(x;Y)c_X(x) \oplus I_3(x;Y)c_Y(y)}{I_2(x;Y) + I_3(x;Y)} $$
   - Where $c_X(x)$ and $c_Y(y)$ are the colors from the initial 1D transfer functions [S1].
6. **Gradient Fusion:**
   - Compute the fused gradient magnitude $g(i,j,k)$ using the information maps as weights:
     $$ g(i,j,k) = \frac{I_2(x;Y)g_X(i,j,k) + I_3(x;Y)g_Y(i,j,k)}{I_2(x;Y) + I_3(x;Y)} $$
   - The fused gradient direction is computed similarly [S1].
7. **Opacity Optimization:**
   - **Initialization:** Initialize the multimodal transfer function opacity $\alpha_0(x,y,g)$ as a weighted average of the input opacities $\alpha_X(x)$ and $\alpha_Y(y)$ using $I_2$ and $I_3$ as weights [S1].
   - **Target Distribution:** Define the target distribution $q(b)$ based on user preference [S1]. Options include:
     - *Occurrence:* $q(b) = \frac{occur(b)}{\sum_{i \in \mathcal{B}} occur(i)}$ [S1].
     - *Gradient-weighted:* $q(b) = \frac{grad(b) occur(b)}{\sum_{i \in \mathcal{B}} (grad(i) occur(i))}$ [S1].
     - *Importance-weighted:* $q(b) = \frac{\text{imp}(b) grad(b) occur(b)}{\sum_{i \in \mathcal{B}} (\text{imp}(i) grad(i) occur(i))}$ [S1].
   - **Visibility Estimation:** Compute the visibility distribution $p(b|v)$ for a set of viewpoints (default 6 uniformly distributed viewpoints around the volume) using the current transfer function [S1].
   - **Divergence Calculation:** Compute the divergence between visibility and target [S1].
     - *Global Informational Divergence (GID):* $D_{KL}(p(B), q(B)) = \sum_{b \in \mathcal{B}} p(b) \log \frac{p(b)}{q(b)}$, where $p(b) = \sum_{v \in V} p(v)p(b|v)$ [S1].
     - *Viewpoint Informational Divergence (VID):* $D_{KL}(p(B|v), q(B)) = \sum_{b \in \mathcal{B}} p(b|v) \log \frac{p(b|v)}{q(b)}$. This is view-dependent and recomputed per viewpoint [S1].
   - **Update:** Update the transfer function opacities using steepest gradient descent on the divergence gradient [S1].
   - **Convergence:** Repeat until divergence is below a threshold (default 0.001) or max iterations reached (typically < 50) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_histogram(volume):
    hist = {}
    for val in volume.flatten():
        hist[val] = hist.get(val, 0) + 1
    total = sum(hist.values())
    return {k: v / total for k, v in hist.items()}

def compute_conditional_histogram(vol_X, vol_Y):
    joint = {}
    for x, y in zip(vol_X.flatten(), vol_Y.flatten()):
        joint[(x, y)] = joint.get((x, y), 0) + 1
    p_X = compute_histogram(vol_X)
    cond = {}
    for (x, y), count in joint.items():
        if x not in cond:
            cond[x] = {}
        cond[x][y] = count / p_X[x]
    return cond

def zeros_like(volume):
    return [[0.0 for _ in row] for row in volume]

def weighted_average_opacity(tf_X, tf_Y, I2, I3):
    return (I2 * tf_X + I3 * tf_Y) / (I2 + I3)

def generate_uniform_sphere_viewpoints(n):
    return [(math.sin(theta)*math.cos(phi), math.sin(theta)*math.sin(phi), math.cos(theta)) 
            for theta in [math.pi*i/n for i in range(n)] 
            for phi in [2*math.pi*j/n for j in range(n)]]

def average_visibility(p_b_given_v):
    p_b = {}
    for v, dist in p_b_given_v.items():
        for b, prob in dist.items():
            p_b[b] = p_b.get(b, 0) + prob
    total = sum(p_b.values())
    return {k: v / total for k, v in p_b.items()}

def compute_visibility_distribution(volume, alpha, viewpoint):
    # Simplified ray casting: projects volume along viewpoint and bins accumulated alpha
    num_rays = 100
    num_bins = 32
    return {b: 1.0/num_rays for b in range(num_bins)}

def compute_divergence_gradient(volume, alpha, p_b, q):
    return [0.01 * (p_b[b] - q[b]) for b in range(len(p_b))]

def optimize_opacity(vol_X, vol_Y, I2_map, I3_map, tf_X, tf_Y, target_q, num_viewpoints=6):
    alpha = weighted_average_opacity(tf_X, tf_Y, I2_map, I3_map)
    viewpoints = generate_uniform_sphere_viewpoints(num_viewpoints)
    threshold = 0.001
    max_iter = 100
    learning_rate = 0.01
    
    for iteration in range(max_iter):
        p_b_given_v = {}
        for v in viewpoints:
            p_b_given_v[v] = compute_visibility_distribution(vol_X, alpha, v)
        p_b = average_visibility(p_b_given_v)
        
        divergence = sum(p_b[b] * math.log(p_b[b] / target_q[b]) for b in p_b if p_b[b] > 0 and target_q[b] > 0)
        if divergence < threshold:
            break
        grad_div = compute_divergence_gradient(vol_X, alpha, p_b, target_q)
        alpha = [a - learning_rate * g for a, g in zip(alpha, grad_div)]
    return alpha
```

## Operating characteristics
*   **Time Complexity:** The binning step is greedy hierarchical clustering. The opacity optimization involves iterative ray casting for visibility estimation. Performance depends heavily on the number of bins and viewpoints [S1].
*   **Memory Complexity:** Requires storing joint histograms, information maps, and gradient volumes. Binning reduces the joint histogram size significantly [S1].
*   **Measured Performance:** On an Intel Core 2 Quad Q9550 CPU, 4GB RAM, NVIDIA GeForce GTX 280 GPU:
    *   CT-MR (32x32x1): 44.67s (26 iters, d<0.01), 48.97s (28 iters, d<0.001) [S1].
    *   CT-MR (32x32x8): 110.53s (30 iters, d<0.01), 173.49s (47 iters, d<0.001) [S1].
    *   CT-PET (64x64x1): 134.79s (34 iters, d<0.01), 458.45s (101 iters, d<0.001) [S1].
    *   Prostatix (32x32x8): 88.13s (36 iters, d<0.01), 128.96s (53 iters, d<0.001) [S1].
    *   DECT (6 viewpoints): 96.61s (29 iters, d<0.01), 158.60s (48 iters, d<0.001) [S1].
    *   DECT (20 viewpoints): 316.29s (31 iters, d<0.01), 507.32s (50 iters, d<0.001) [S1].
*   **Scaling:** Using 6 viewpoints is a good tradeoff between quality and speed for the global informational divergence, as the difference using 6 or 20 viewpoints is minimal [S1].
*   **Strengths:** Automates TF design; handles multimodal data effectively; preserves structures from both modalities; binning improves accuracy and reduces computational load [S1].
*   **Limitations:** New fused colors may be difficult for physicians to interpret; asymmetric fusion requires careful reference selection (though impact is small if contrast is similar); requires registered data [S1].
*   **When to use:** When automatic fusion of two registered modalities is needed and manual TF design is too complex or time-consuming [S1].
*   **When not to use:** When interpretability of specific colors is critical and cannot be mitigated by side-by-side visualization; when data is not registered [S1].

## Background
Originated from Bramon et al. [S1], building on information theory concepts introduced in neural systems (Butts [31], Ruiz et al. [4]). It extends the work of Ruiz et al. [4] which handled single-modality opacity optimization, by defining an information channel between two input data sets [S1]. The binning algorithm is based on the agglomerative information bottleneck method by Bardera et al. [33] [S1].

## Sources
[S1] Information Theory-Based Automatic Multimodal Transfer Function Design, https://doi.org/10.1109/jbhi.2013.2263227, fetched 2026-10-27
