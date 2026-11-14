# Differentiable Primitive Assembly

Also called: DPA-Net Primitive Assembly, Quadric Primitive Assembly

## Overview
Differentiable Primitive Assembly (DPA) models 3D shapes as an assembly of quadric primitives using differentiable intersection and union operations, trained with image-space losses only without 3D supervision [S1]. It sets itself apart from standard NeRF by replacing the continuous density field with a discrete, structured representation of convex hulls formed by quadric surfaces, allowing for explicit shape abstraction and test-time adaptation [S1].

## How it works
**Inputs:** Fused image features $f$ (from a backbone like PixelNeRF [S1]), query points $\mathbf{x}_i$ sampled along camera rays, and ground truth images/masks.
**Outputs:** Primitive parameters, selection matrix, and rendered color/mask.

**Data Structures:**
1.  **Primitive Parameters Matrix $\mathbf{P}$**: Shape $\mathbb{R}^{P \times 7}$. Each row $p$ contains $(a_p, b_p, c_p, d_p, e_p, f_p, g_p)$. The first three parameters are enforced to be non-negative to ensure convexity [S1].
2.  **Selection Matrix $\mathbf{T}$**: Shape $\mathbb{R}^{P \times C}$. Ideally binary, but takes floating-point values in $[0, 1]$ during training. Each column $c$ selects a subset of primitives whose intersection forms one of the $C$ convexes [S1].
3.  **Weight Vector $\mathbf{w}$**: Shape $\mathbb{R}^{C}$. Learnable weights for the soft union operation [S1].

**Parameters:**
*   $P = 4,096$: Number of primitives [S1].
*   $C = 256$: Number of convexes [S1].
*   Optimizer: Adam, learning rate $0.0001$ [S1].
*   Batch size: 4 object instances [S1].

**Steps:**

1.  **Compute Query Matrix $\mathbf{Q}$**: For $N$ query points, construct matrix $\mathbf{Q} \in \mathbb{R}^{N \times 7}$ where the $i$-th row is $(x_i^2, y_i^2, z_i^2, x_i, y_i, z_i, 1)$ [S1].
2.  **Compute Distance Matrix $\mathbf{D}$**: Calculate $\mathbf{D} = \mathbf{Q}\mathbf{P}^\top$. The entry $D_{i,p}$ is the approximate signed distance between point $i$ and primitive $p$. $D_{i,p} < 0$ if inside, $> 0$ if outside [S1].
3.  **Compute Convex Occupancy $\mathbf{O}$**: Calculate $\mathbf{O} = \text{ReLU}(\mathbf{D})\mathbf{T}$. $O_{i,c} = 0$ if point $i$ is inside convex $c$, $> 0$ if outside [S1].
4.  **Compute Exact Occupancy $\mathbf{a}^*$**: Apply min-pooling across convexes:
    $$ a^*_i = \min_{1 \leq c \leq C} O_{i,c} $$
    $a^*_i = 0$ if inside the shape, $> 0$ if outside [S1].
5.  **Compute Soft Occupancy $\mathbf{a}^+$**: For gradient flow during pretraining:
    $$ a^+_i = \mathcal{L}\left( \sum_{c=1}^{C} w_c \mathcal{L}(1 - O_{i,c}) \right) $$
    where $\mathcal{L}(\cdot)$ clips input to $[0, 1]$. $a^+_i \approx 1$ if inside, $< 1$ if outside [S1].
6.  **Differentiable Rendering**:
    *   Opacity $\alpha_i$:
        *   Phase 1: $\alpha_i = a^+_i$ [S1].
        *   Phase 2 & 3: $\alpha_i = \exp(-10 a^*_i)$ [S1].
    *   Transmittance $t_i = \prod_{k=1}^{i-1} (1 - \alpha_k)$ [S1].
    *   Color: $C_b = \sum_{i=1}^{R} t_i \alpha_i c_{b,i}$ [S1].
    *   Mask: $M_c = \sum_{i \in \Omega} t_i \alpha_i$ [S1].
7.  **Loss Calculation**:
    *   Photo/Mask Loss $\mathcal{L}_{ph}$:
        $$ \mathcal{L}_{ph} = \frac{1}{B} \sum_{j=1}^{B} ||\hat{C}_j - C_j||_2^2 + \frac{1}{B} \sum_{j=1}^{B} ||\hat{M}_j - M_j||_2^2 $$
        where $B$ is the number of pixels in the batch [S1].
    *   Selection Matrix Loss $\mathcal{L}_T$:
        $$ \mathcal{L}_T = \sum_{i,j} \max(-T_{i,j}, 0) + \max(T_{i,j} - 1, 0) $$
        Keeps $T$ entries in $[0, 1]$ [S1].
    *   Weight Loss $\mathcal{L}_w$:
        $$ \mathcal{L}_w = \sum_j |w_j - 1| $$
        Ensures weights are close to 1 [S1].
    *   Overlapping Loss $\mathcal{L}_{over}$ (Phase 3 only):
        $$ h_i = \sum_{c=1}^{C} \exp(-10 O_{i,c}) $$
        $$ \mathcal{L}_{over} = \sum_{i \in \Omega} \max(h_i, 1.9) $$
        where $\Omega$ is the set of points inside the shape ($a^*_i < 0.01$) [S1].

**Training Phases:**
1.  **Pretraining**: Minimize $\mathcal{L} = \mathcal{L}_{ph} + \mathcal{L}_T + \mathcal{L}_w$. Opacity $\alpha = a^+$. Parameters: network, $\mathbf{T}$, $\mathbf{w}$ [S1].
2.  **Test-Time Adaptation Phase 1**: Use test images as GT. Loss: $\mathcal{L}_{ph} + \mathcal{L}_T + \mathcal{L}_w$. Opacity: $\alpha = a^+$. Parameters: network, $\mathbf{T}$, $\mathbf{w}$ [S1].
3.  **Test-Time Adaptation Phase 2**: Use exact occupancy. Loss: $\mathcal{L}_{ph} + \mathcal{L}_T$. Opacity: $\alpha = \exp(-10 a^*)$. Parameters: network, $\mathbf{T}$ [S1].
4.  **Test-Time Adaptation Phase 3**: Binarize $\mathbf{T}$ with threshold 0.01 and freeze it. Loss: $\mathcal{L}_{ph} + \mathcal{L}_{over}$. Opacity: $\alpha = \exp(-10 a^*)$. Parameters: network [S1].

**Edge Cases and Pitfalls:**
*   **Complex Topologies**: Standard random pixel sampling struggles with thin structures or holes. Use silhouette-aware sampling: 256 random pixels + 1,000 pixels around contours (obtained via morphological closing on mask + Gaussian noise) [S1].
*   **Redundant Primitives**: Apply primitive dropout every 400 iterations in Phase 3. For each primitive $p$, set row $p$ of $\mathbf{T}$ to zero, compute new occupancy $b^*$, and quantify variation $v = K \sum_{i=1}^{P} \mathbb{1}(\text{XOR}(b^*_i < 0.1, a^*_i < 0.1))$. If $v \approx 0$, keep row zero; else restore. $K=40,960$ points sampled around the shape [S1].
*   **Overlap**: The overlapping loss $\mathcal{L}_{over}$ penalizes points inside multiple convexes. Sample 40,960 points surrounding the shape and check membership in $\Omega$ using $a^*_i < 0.01$ [S1].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Trained on 4 NVIDIA Tesla V100 GPUs [S1].
*   **Strengths**: Explicit shape abstraction, test-time adaptation for instance-specific refinement, handles sparse views [S1].
*   **Limitations**: Struggles with complex topologies without silhouette-aware sampling; redundant primitives require dropout [S1].
*   **When to use**: When explicit 3D shape abstraction is required from sparse views without 3D supervision [S1].
*   **When not**: When continuous density fields are sufficient or when dense multi-view inputs are available [S1].

## Background
DPA-Net was introduced in 2024 by the authors of "DPA-Net: Structured 3D Abstraction from Sparse Views via Differentiable Primitive Assembly" [S1]. It builds on CAPRI-Net [S1] and adapts differentiable CSG operations from [6, 55] for 3D shape abstraction. It is used for instance-specific refinement during inference via multi-phase test-time adaptation [S1].

## Sources
[S1] DPA-Net: Structured 3D Abstraction from Sparse Views via Differentiable Primitive Assembly, https://arxiv.org/abs/2404.00875v3, fetched 2026-10-26
