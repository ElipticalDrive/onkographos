# Differentiable Primitive Assembly

Also called: DPA-Net Primitive Assembly, Quadric Primitive Assembly

## Overview
Differentiable Primitive Assembly (DPA) models 3D shapes as an explicit assembly of learnable quadric primitives, utilizing differentiable set-theoretic operations (intersection and union) to compute occupancy. Unlike implicit neural representations, DPA constructs geometry from discrete primitives, enabling compact shape abstraction and test-time adaptation. It distinguishes itself from sibling methods like Gaussian Splatting by using algebraic primitive intersections rather than rasterization, and from Point-NeRF by using structured convex hulls rather than point-based density fields [S1].

## How it works
**Inputs:** A set of sparse input images (or fused image features $f$), camera poses, and query points $\mathbf{x}_i$ sampled along rays.
**Outputs:** Rendered color $C_b$, object mask $M_c$, and a compact set of primitive parameters defining the 3D shape.

### Data Structures
1.  **Primitive Parameters Matrix ($\mathbf{P}$):** Shape $\mathbb{R}^{P \times 7}$. Each row $p$ contains coefficients $(a_p, b_p, c_p, d_p, e_p, f_p, g_p)$ for a quadric primitive. The first three parameters are constrained to be non-negative to ensure convexity [S1].
2.  **Selection Matrix ($\mathbf{T}$):** Shape $\mathbb{R}^{P \times C}$. Ideally binary, but takes floating-point values in $[0, 1]$ during training. Column $c$ selects a subset of primitives whose intersection forms the $c$-th convex [S1].
3.  **Weight Vector ($\mathbf{w}$):** Shape $\mathbb{R}^{C}$. Learnable weights for the soft union approximation, with elements close to 1 [S1].

### Formulas
**Quadric Primitive Equation:**
$$ |a_p|x^2 + |b_p|y^2 + |c_p|z^2 + d_p x + e_p y + f_p z + g_p = 0 $$
where $(x, y, z)$ is a query point [S1].

**Distance Matrix ($\mathbf{D}$):**
Define query matrix $\mathbf{Q}$ where the $i$-th row is $(x_i^2, y_i^2, z_i^2, x_i, y_i, z_i, 1)$.
$$ \mathbf{D} = \mathbf{Q}\mathbf{P}^\top $$
$\mathbf{D}_{i,p} < 0$ if point $i$ is inside primitive $p$, and $\mathbf{D}_{i,p} > 0$ if outside [S1].

**Convex Occupancy ($\mathbf{O}$):**
$$ \mathbf{O} = \text{ReLU}(\mathbf{D})\mathbf{T} $$
$\mathbf{O}_{i,c} = 0$ if point $i$ is inside convex $c$, and $\mathbf{O}_{i,c} > 0$ if outside [S1].

**Exact Occupancy ($\mathbf{a}^*$):**
Computed via min-pooling (union of convexes):
$$ \mathbf{a}^*_i = \min_{1 \leq c \leq C} \mathbf{O}_{i,c} $$
$\mathbf{a}^*_i = 0$ if inside the shape, $> 0$ if outside [S1].

**Soft Occupancy ($\mathbf{a}^+$):**
Used for gradient flow during pretraining:
$$ \mathbf{a}^+_i = \mathcal{L}\left( \sum_{c=1}^{C} w_c \mathcal{L}(1 - \mathbf{O}_{i,c}) \right) $$
where $\mathcal{L}(\cdot)$ clips input to $[0, 1]$. $\mathbf{a}^+_i \approx 1$ if inside, $< 1$ if outside [S1].

**Rendering:**
Color $C_b$ and mask $M_c$ for a pixel are computed by accumulating along ray points $i=1 \dots R$:
$$ C_b = \sum_{i=1}^{R} t_i \alpha_i c_{b,i}, \quad M_c = \sum_{i \in \Omega} t_i \alpha_i $$
where $t_i = \prod_{k=1}^{i-1} (1 - \alpha_k)$ is transmittance, and $\alpha_i$ is opacity derived from occupancy [S1].

### Parameters
*   $P = 4,096$: Number of primitives [S1].
*   $C = 256$: Number of convexes [S1].
*   Optimizer: Adam, learning rate $0.0001$ [S1].
*   Batch size: 4 object instances [S1].

### Steps and Pseudocode

1.  **Initialize** $\mathbf{P} \in \mathbb{R}^{P \times 7}$, $\mathbf{T} \in \mathbb{R}^{P \times C}$, $\mathbf{w} \in \mathbb{R}^{C}$.
2.  **Sample Query Points:** Generate $N$ query points $\mathbf{x}_i$ along camera rays.
3.  **Compute Distances:**
    *   Construct $\mathbf{Q}$ from $\mathbf{x}_i$.
    *   Compute $\mathbf{D} = \mathbf{Q}\mathbf{P}^\top$.
4.  **Compute Convex Occupancy:**
    *   $\mathbf{O} = \text{ReLU}(\mathbf{D})\mathbf{T}$.
5.  **Compute Opacity:**
    *   Depending on training phase, compute $\alpha_i$ from $\mathbf{a}^*$ or $\mathbf{a}^+$ (see Training Phases below).
6.  **Render:**
    *   Compute $C_b$ and $M_c$ using volume rendering equation.
7.  **Compute Loss:**
    *   $\mathcal{L} = \mathcal{L}_{ph} + \mathcal{L}_T + \mathcal{L}_w$ (Pretraining) or variant (TTA).
8.  **Update:**
    *   Backpropagate and update parameters.

**Pseudocode for Occupancy and Rendering:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def compute_quadric_distances(query_points, P_matrix):
    """
    query_points: (N, 3) tensor of (x, y, z)
    P_matrix: (P, 7) tensor of primitive parameters (a, b, c, d, e, f, g)
    Returns: D (N, P)
    """
    # Q: (N, 7) row is (x^2, y^2, z^2, x, y, z, 1)
    x, y, z = query_points[:, 0], query_points[:, 1], query_points[:, 2]
    Q = torch.stack([x**2, y**2, z**2, x, y, z, torch.ones_like(x)], dim=1)
    # D = Q @ P^T
    D = torch.matmul(Q, P_matrix.T)
    return D

def compute_convex_occupancy(D, T):
    """
    D: (N, P) distance matrix
    T: (P, C) selection matrix
    Returns: O (N, C) convex occupancy
    """
    # O = ReLU(D) @ T
    O = F.relu(D) @ T
    return O

def compute_exact_occupancy(O):
    """
    O: (N, C) convex occupancy
    Returns: a_star (N,) exact occupancy
    """
    # a_star = min(O, dim=1)
    a_star, _ = torch.min(O, dim=1)
    return a_star

def compute_soft_occupancy(O, w):
    """
    O: (N, C) convex occupancy
    w: (C,) learnable weights
    Returns: a_plus (N,) soft occupancy
    """
    # inner = L(1 - O)
    inner = torch.clamp(1 - O, min=0, max=1)
    # weighted sum
    weighted_sum = inner @ w
    # outer L
    a_plus = torch.clamp(weighted_sum, min=0, max=1)
    return a_plus

def render_ray(points_along_ray, alphas, colors):
    """
    points_along_ray: (R,) not strictly needed for alpha/color accumulation if ordered
    alphas: (R,) opacity values
    colors: (R, 3) color values
    Returns: C_b (3,), M_c (1,)
    """
    # t_i = product(1 - alpha_k) for k < i
    # Standard volume rendering accumulation
    transmittance = torch.cumprod(1 - alphas, dim=0)
    # Shift transmittance: t_1 = 1, t_i = product_{k=1}^{i-1} (1-alpha_k)
    # torch.cumprod gives product up to i. We need product up to i-1.
    # Prepend 1.0
    transmittance_shifted = F.pad(transmittance, (1, 0), value=1.0)[:-1]
    
    contribution = transmittance_shifted.unsqueeze(1) * alphas.unsqueeze(1) * colors
    C_b = torch.sum(contribution, dim=0)
    
    # Mask accumulation (only for points inside object, typically alpha > 0 implies inside)
    # M_c = sum(t_i * alpha_i)
    M_c = torch.sum(transmittance_shifted * alphas)
    
    return C_b, M_c
```

### Training Phases and Losses

**Pretraining:**
*   **Opacity:** $\alpha = \mathbf{a}^+$ [S1].
*   **Loss:** $\mathcal{L} = \mathcal{L}_{ph} + \mathcal{L}_T + \mathcal{L}_w$ [S1].
*   $\mathcal{L}_{ph} = \frac{1}{B} \sum_{j=1}^B ||\hat{C}_j - C_j||_2^2 + \frac{1}{B} \sum_{j=1}^B ||\hat{M}_j - M_j||_2^2$ [S1].
*   $\mathcal{L}_T = \sum_{i,j} \max(-T_{i,j}, 0) + \max(T_{i,j} - 1, 0)$ (Keeps $T \in [0, 1]$) [S1].
*   $\mathcal{L}_w = \sum_j |w_j - 1|$ [S1].

**Test-Time Adaptation (TTA) Phase 1:**
*   **Input:** Test images as Ground Truth.
*   **Opacity:** $\alpha = \mathbf{a}^+$ [S1].
*   **Loss:** $\mathcal{L}_{ph} + \mathcal{L}_T + \mathcal{L}_w$ [S1].
*   **Parameters:** Network, $\mathbf{T}$, $\mathbf{w}$ [S1].

**Test-Time Adaptation (TTA) Phase 2:**
*   **Opacity:** $\alpha = \exp(-10\mathbf{a}^*)$ [S1].
*   **Loss:** $\mathcal{L}_{ph} + \mathcal{L}_T$ ($\mathcal{L}_w$ omitted) [S1].
*   **Parameters:** Network, $\mathbf{T}$ [S1].
*   **Sampling:** Sample 40,960 points surrounding the shape. Check membership in $\Omega$ (inside shape) using $\mathbf{a}^*_i < 0.01$ [S1].

**Test-Time Adaptation (TTA) Phase 3:**
*   **Selection Matrix:** Binarize $\mathbf{T}$ with threshold 0.01 and freeze [S1].
*   **Opacity:** $\alpha = \exp(-10\mathbf{a}^*)$ [S1].
*   **Loss:** $\mathcal{L}_{ph} + \mathcal{L}_{over}$ [S1].
*   **Overlapping Loss:** $\mathcal{L}_{over} = \sum_{i \in \Omega} \max(h_i, 1.9)$, where $h_i = \sum_{c=1}^C \exp(-10\mathbf{O}_{i,c})$ [S1].
*   **Parameters:** Network [S1].
*   **Primitive Dropout:** Applied every 400 iterations. For each primitive $p$, set row $p$ of $\mathbf{T}$ to zero, compute new occupancy $\mathbf{b}^*$, and calculate variation $v = K \sum_{i=1}^P \mathbb{I}(\text{XOR}(\mathbf{b}^*_i < 0.1, \mathbf{a}^*_i < 0.1))$. If $v \approx 0$, keep row zero; else restore [S1].

### Edge Cases and Pitfalls
*   **Complex Topologies:** Random pixel sampling struggles with thin structures or holes. Use silhouette-aware sampling: 256 random pixels + 1,000 pixels around contours (obtained via morphological closing on mask + Gaussian noise) [S1].
*   **Redundant Primitives:** Use primitive dropout to remove primitives that do not significantly alter the shape [S1].
*   **Overlap:** Excessive overlap between convexes is penalized via $\mathcal{L}_{over}$ in Phase 3 [S1].

## Operating characteristics
*   **Complexity:** Time complexity depends on the number of query points $N$, primitives $P$, and convexes $C$. Matrix multiplications $\mathbf{Q}\mathbf{P}^\top$ ($O(NP)$) and $\mathbf{D}\mathbf{T}$ ($O(NPC)$) dominate. Memory is $O(NP + PC)$ for intermediate matrices [S1].
*   **Performance:** Trained on 4 NVIDIA Tesla V100 GPUs with batch size 4 [S1].
*   **Strengths:** Compact representation (up to 29% primitive reduction via dropout), differentiable assembly allows end-to-end training from image-space losses only, no 3D supervision required [S1].
*   **Limitations:** Requires test-time adaptation for novel shapes deviating from training data; complex topologies require specific sampling strategies [S1].

## Background
DPA-Net was introduced in "DPA-Net: Structured 3D Abstraction from Sparse Views via Differentiable Primitive Assembly" (2024) [S1]. It builds on CAPRI-Net [55] and differentiable CSG operations [6, 55]. It addresses the limitations of monolithic MLPs by using explicit quadric primitives, enabling structured 3D abstraction from sparse views.

## Sources
[S1] DPA-Net: Structured 3D Abstraction from Sparse Views via Differentiable Primitive Assembly, https://arxiv.org/abs/2404.00875v3, fetched 2026-10-26
