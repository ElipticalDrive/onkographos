# Loss Scheduling for Candidate Head

Also called: candidate scheduling, v scheduling

## Overview
Loss scheduling for the candidate head is a training strategy that gradually reduces the influence of an image-dependent 'candidate head' ($\hat{C}_i$) during optimization, transitioning to a shared representation ($\hat{C}$) to stabilize pose estimation for hard-pose images [S1, unofficial]. It prevents the shared representation from being distracted by wrong supervision introduced by hard-pose images in the early stages, allowing the model to assimilate these images into the shared representation once the shared model is accurate enough [S1, unofficial]. This method is distinct from sibling methods like [InstantSplat](6136-instantsplat.md) and [pixelSplat](6464-pixelsplat.md) by explicitly decoupling the training influence of per-image heads via a cosine schedule rather than relying solely on joint pose optimization or fixed head structures.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training progress $l \in [0, 1]$; hyperparameters $u$ (start of transition) and $v$ (end of transition); rendered color from shared head $\hat{C}$; rendered color from candidate head $\hat{C}_i$; ground truth image $I$.
*   **Outputs:** Scheduled loss value $\mathcal{L}$.

**Data Structures:**
*   $l$: Scalar float representing normalized training progress (0 to 1).
*   $u, v$: Scalar floats in $[0, 1]$ defining the schedule boundaries.
*   $\mathcal{L}_{rgb}(\cdot)$: Photometric loss function (e.g., $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$).

**Parameters:**
*   $u$: Start of transition phase. Typical value: $0.1$ [S1, unofficial].
*   $v$: End of transition phase. Typical value: $0.5$ [S1, unofficial].

**Steps:**
1.  Calculate the scheduling weight $w_{u,v}(l)$ based on current training progress $l$ and hyperparameters $u, v$.
2.  Compute the photometric loss for the shared head: $\mathcal{L}_{shared} = \mathcal{L}_{rgb}(\hat{C} | C)$.
3.  Compute the photometric loss for the candidate head: $\mathcal{L}_{candidate} = \mathcal{L}_{rgb}(\hat{C}_i | C)$.
4.  Combine losses using the schedule: $\mathcal{L} = w_{u,v} \mathcal{L}_{shared} + (1 - w_{u,v}) \mathcal{L}_{candidate}$.

**Formulas:**
The scheduling weight $w_{u,v}(l)$ is defined as:
$$
w_{u,v}(l) = 
\begin{cases} 
0 & \text{if } l < u \\
1 - \frac{\cos(\pi(l - u)/(v - u))}{2} & \text{if } u \le l < v \\
1 & \text{if } l \ge v
\end{cases}
$$
The total loss is:
$$
\mathcal{L} = w_{u,v} \mathcal{L}_{rgb}(\hat{C} | C) + (1 - w_{u,v}) \mathcal{L}_{rgb}(\hat{C}_i | C)
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_loss_schedule(l, u, v, loss_shared, loss_candidate):
    """
    Computes the scheduled loss for candidate head training.
    
    Args:
        l (float): Training progress in [0, 1].
        u (float): Start of transition phase.
        v (float): End of transition phase.
        loss_shared (float): Photometric loss from shared head.
        loss_candidate (float): Photometric loss from candidate head.
        
    Returns:
        float: Scheduled loss value.
    """
    if l < u:
        w = 0.0
    elif l < v:
        # Cosine interpolation from 0 to 1
        # Formula: 1 - cos(pi * (l - u) / (v - u)) / 2
        w = 1.0 - (math.cos(math.pi * (l - u) / (v - u)) / 2.0)
    else:
        w = 1.0
        
    # Weighted combination
    # When w=0, only candidate head contributes.
    # When w=1, only shared head contributes.
    scheduled_loss = w * loss_shared + (1.0 - w) * loss_candidate
    
    return scheduled_loss
```

**Edge Cases and Pitfalls:**
*   If $u \ge v$, the transition interval is invalid or empty. The sources assume $u < v$ [S1, unofficial].
*   Precision issues in the cosine calculation near boundaries $u$ and $v$ are mitigated by the piecewise definition ensuring continuity ($w=0$ at $l=u$, $w=1$ at $l=v$).
*   The candidate head size (embedding $\ell_i$) should be adjusted based on the challenge of the image poses, acting as a regularizer for 'slack' variables [S1, unofficial].

## Subtopics

### Feature-Surrogate Bundle Adjustment
This variant replaces the photometric loss $\mathcal{L}_{rgb}$ with a feature-based loss $\mathcal{L}_{feat}$ during the early training stage ($l < u$) to stabilize training against appearance changes (weather/time) [S1, unofficial]. It uses deep features from a Vision Transformer (ViT) as a surrogate for RGB values, which are less sensitive to appearance changes [S1, unofficial].

**Steps:**
1.  Render features $\hat{F}_i(r)$ using volume rendering with feature field $f(t)$, shared density $\sigma(t)$, candidate feature $f_i(t)$, and candidate density $\sigma_i(t)$.
2.  Compute feature loss: $\mathcal{L}_{feat}(r) = \| F_i(r) - \hat{F}_i(r) \|_2^2$, where $F_i(r)$ is the 2D deep feature from a pretrained model (e.g., DINO) [S1, unofficial].
3.  Replace $\mathcal{L}_{rgb}(\hat{C}_i | C_i)$ with $\mathcal{L}_{feat}(\hat{F}_i | F_i)$ for $l < u$ [S1, unofficial].
4.  Optimize static color $\hat{C}_i^{(a)}(r)$ jointly with transient color $\hat{C}_i^{(\tau)}(r)$ using volume rendering with appearance-dependent color $c_i^{(a)}(t)$ and $\sigma(t)$ [S1, unofficial].

**Formulas:**
*   Feature field: $f(t) = \Pi_{\theta_2}(z(t))$, where $z(t)$ is positional encoding. No viewing direction is used for $f(t)$ [S1, unofficial].
*   Candidate head feature/density: $[f_i(t), \sigma_i(t)] = \Pi_{\theta_3}(z(t), \ell_i^{(c)})$, where $\Pi_{\theta_3}$ is the candidate head MLP and $\ell_i^{(c)}$ is the candidate embedding [S1, unofficial].
*   Appearance-dependent static color: $c_i^{(a)}(t) = \Pi_{\theta_4}(f(t), \gamma_d(d), \ell_i^{(a)})$, where $d$ is viewing direction and $\ell_i^{(a)}$ is appearance embedding [S1, unofficial].

**When to choose:** Use this variant when training on unconstrained images with diverse appearances due to different weather or times, where RGB values of corresponding pixels are not guaranteed to be the same [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds minimal overhead to the training loop as it only involves computing a scalar weight and combining two pre-computed loss values. The computational cost is dominated by the rendering of the candidate and shared heads.
*   **Memory Complexity:** Requires storing parameters for both the shared head and the candidate head. The size of the candidate embedding $\ell_i$ can be adjusted based on pose difficulty [S1, unofficial].
*   **Performance:** Enables high-quality NeRF model training with perfect camera poses even for hard-pose images that would otherwise distract the shared representation [S1, unofficial]. It allows the model to converge where methods like BARF fail [S1, unofficial].
*   **Strengths:** Stabilizes pose estimation for hard-pose images; prevents shared representation corruption; robust to appearance changes when combined with feature surrogates.
*   **Limitations:** Requires careful tuning of hyperparameters $u$ and $v$; relies on the candidate head architecture being sufficiently expressive.
*   **When to use:** When training pose-free, feed-forward models on unconstrained image sets with varying poses and appearances.
*   **When not to use:** When camera poses are known and fixed, or when the scene has consistent appearance across all views.

## Background
The method originates from UP-NeRF (Unconstrained Pose-Prior-Free Neural Radiance Fields) [S1, unofficial]. It addresses the challenge of training NeRFs without camera pose priors on unconstrained image collections, where hard-pose images can introduce wrong supervision. The approach builds on the principle that the shared representation should not be distracted by hard-pose images initially, allowing the candidate head to handle them until the shared representation is robust enough [S1, unofficial]. The scheduling parameters $u$ and $v$ are set to 0.1 and 0.5 respectively, aligning with coarse-to-fine training strategies [S1, unofficial].

## Sources
[S1] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
