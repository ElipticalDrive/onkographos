# PlaneTV Regularization

Also called: PlaneTV, Total Variation on Planes

## Overview
PlaneTV Regularization computes the total variation of spatial planes to encourage smoothness in feature fields [S1, unofficial]. It operates on multi-dimensional tensors by summing differences along specified spatial dimensions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A tensor $t$ with batch size $B$, channels $C$, height $H$, and width $W$ [S1, unofficial].
- Output: A scalar loss value representing the regularized TV [S1, unofficial].

**Data Structures:**
- The tensor is treated as a collection of spatial planes [S1, unofficial]. For a grid length of 3, spatial grids are indices $[0, 1, 2]$ [S1, unofficial]. For a grid length not equal to 3 (4D case), spatial grids are indices $[0, 1, 3]$ [S1, unofficial].

**Formula:**
The loss is computed as:
$$ L = 2 \times \left( \frac{\sum (t_{i+1} - t_i)^2}{count_h} + \frac{\sum (t_{j+1} - t_j)^2}{count_w} \right) $$

Where:
- $t_{i+1} - t_i$ represents the difference along the height dimension (rows) [S1, unofficial].
- $t_{j+1} - t_j$ represents the difference along the width dimension (columns) [S1, unofficial].
- $count_h = B \times C \times (H-1) \times W$ [S1, unofficial].
- $count_w = B \times C \times H \times (W-1)$ [S1, unofficial].

**Steps:**
1. Compute the squared differences along the height dimension: $\sum (t_{i+1} - t_i)^2$ [S1, unofficial].
2. Compute the squared differences along the width dimension: $\sum (t_{j+1} - t_j)^2$ [S1, unofficial].
3. Normalize the height sum by $count_h$ [S1, unofficial].
4. Normalize the width sum by $count_w$ [S1, unofficial].
5. Sum the normalized values and multiply by 2 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_plane_tv(t: list, B: int, C: int, H: int, W: int) -> float:
    sum_h = 0.0
    for b in range(B):
        for c in range(C):
            for h in range(H - 1):
                for w in range(W):
                    diff = t[b][c][h+1][w] - t[b][c][h][w]
                    sum_h += diff * diff
    count_h = B * C * (H - 1) * W

    sum_w = 0.0
    for b in range(B):
        for c in range(C):
            for h in range(H):
                for w in range(W - 1):
                    diff = t[b][c][h][w+1] - t[b][c][h][w]
                    sum_w += diff * diff
    count_w = B * C * H * (W - 1)

    term_h = sum_h / count_h if count_h > 0 else 0.0
    term_w = sum_w / count_w if count_w > 0 else 0.0

    return 2.0 * (term_h + term_w)
```

**Parameters:**
- Not given in the sources.

**Edge Cases:**
- If $H=1$ or $W=1$, the corresponding count becomes 0, leading to division by zero [S1, unofficial]. The implementation must guard against this by setting the term to 0 [G].

## Subtopics

### Triplane TV Regularization
Applies TV regularization to triplane features to smooth them and make gradients resemble natural images, which are modeled by a hyper-Laplacian with $\alpha \in [0.5, 0.8]$ [S2]. The regularization target $\alpha$ is typically set to 0.5 [S2]. This variant is chosen when training generative models on triplanes to prevent high-frequency artifacts [S2].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Strengths and limitations: Not given in the sources.

## Background
Origin and history: Not given in the sources.
Usage: Associated with Triplane Diffusion models for 3D neural field generation [S2].

## Sources
[S1] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
[S2] 3D Neural Field Generation Using Triplane Diffusion, https://doi.org/10.1109/cvpr52729.2023.02000, fetched 2026-10-26
