# LiTe-GS

Also called: Oracle-Efficient NBV Selection for 3D Gaussian Splatting

## Overview
LiTe-GS is an oracle-efficient active next-best-view (NBV) selection framework for 3D Gaussian Splatting (3DGS) that reduces the computational cost of Fisher-information-based view selection by evaluating only a randomized subset of candidate views at each iteration rather than exhaustively scoring the full pool [S1, unofficial]. It formulates NBV selection as the cardinality-constrained maximization of Fisher-information gain, leveraging stochastic submodular optimization to balance estimation accuracy with oracle efficiency, thereby achieving $O(M \log(1/\epsilon))$ oracle complexity independent of the selection cardinality $K$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $H_0$: The prior block-diagonal Fisher information matrix obtained from an initial 3DGS reconstruction trained on a small set of seed views $V_0$ [S1, unofficial].
    *   $V_c$: A pool of $M$ candidate viewpoints for acquisition [S1, unofficial].
    *   $K$: The selection cardinality, i.e., the number of views to select in each acquisition batch [S1, unofficial].
    *   $\epsilon$: An accuracy parameter controlling the trade-off between oracle efficiency and approximation quality [S1, unofficial].
*   **Outputs:**
    *   $S$: A set of $K$ selected views [S1, unofficial].

**Data Structures:**
*   **Fisher Information Matrix ($H$):** Stored as a block-diagonal matrix where each block corresponds to the parameters of an individual Gaussian primitive $i \in \{1, \dots, N\}$ [S1, unofficial].
    *   $H_v \approx \text{blkdiag}(H_v(\hat{\theta}_1), H_v(\hat{\theta}_2), \dots, H_v(\hat{\theta}_N))$ [S1, unofficial].
    *   Each block $H_v(\hat{\theta}_i)$ is computed as $\sigma^{-2} J_{v,i}^\top J_{v,i}$, where $J_{v,i}$ is the Jacobian of the rendered observations for view $v$ with respect to the parameters of Gaussian $i$ [S1, unofficial].
*   **Priority Queue ($Q$):** A max-priority queue storing candidate views from the current sampled subset $P$, ordered by their cached marginal gains $\gamma_v$ [S1, unofficial].

**Formulas:**
*   **3D Gaussian Density:** $G_i(x) = \exp \left( -\frac{1}{2} (x - \mu_i)^\top \Pi_i^{-1} (x - \mu_i) \right)$ [S1, unofficial].
*   **Covariance Parameterization:** $\Pi_i = R_i S_i S_i^\top R_i^\top$, where $R_i \in SO(3)$ and $S_i$ is a scaling matrix [S1, unofficial].
*   **Image-Space Covariance:** $\Pi'_i = J_W \Pi_i W^\top J^\top$, where $W$ is the viewing transformation and $J$ is the Jacobian of the projective transformation [S1, unofficial].
*   **Rendered Color:** $C = \sum_{i \in N} c_i \alpha_i (1 - \sum_{j < i} \alpha_j)$, where $N$ is the set of Gaussians along the pixel’s viewing ray [S1, unofficial].
*   **Fisher Information for View $v$:** $H_v(\hat{\Theta}) \approx \sigma^{-2} \sum_{\text{pixels}} J_v^\top J_v$, where $J_v = \frac{\partial f_v}{\partial \Theta}|_{\hat{\Theta}}$ is the Jacobian of the rendering function [S1, unofficial].
*   **Marginal Gain:** $\rho(\{v\} | S_q)$ is the marginal information gain computed via oracle evaluation, which constructs and accumulates the Fisher-information contribution $H_v$ [S1, unofficial]. The marginal gain is the reduction in uncertainty (increase in information) by adding view $v$ to the selected set $S_q$ [S1, unofficial].
*   **Subset Size:** $p = \frac{M}{K} \log \frac{1}{\epsilon}$ [S1, unofficial].

**Parameters:**
*   $\epsilon$: Typical value 0.1. Tested values include 0.1, 0.2, 0.25, 0.3, 0.4, 0.5, 0.6 [S1, unofficial].
*   $K$: Tested values 3, 4, 6 [S1, unofficial].
*   $\delta$: Small regularization parameter for ill-conditioning, applied as $H_0 \leftarrow H_0 + \delta I$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Initialize selected set $S = \emptyset$ [S1, unofficial].
    *   Initialize current information matrix $H = H_0$ [S1, unofficial].
    *   Precondition $H_0$ by adding regularization: $H_0 \leftarrow H_0 + \delta I$ [S1, unofficial].
    *   Initialize cached marginal gains $\gamma_v \leftarrow \infty$ for all $v \in V_c$ [S1, unofficial].
2.  **Iterative Selection (for $q = 1$ to $K$):**
    *   **Sample Subset:** Sample a subset $P \subset (V_c \setminus S)$ of size $p = \frac{M}{K} \log \frac{1}{\epsilon}$ [S1, unofficial].
    *   **Initialize Queue:** Initialize a max-priority queue $Q$ over views in $P$ using their cached gains $\gamma_v$ [S1, unofficial].
    *   **Lazy Evaluation Loop:**
        *   Repeat:
            *   Pop view $v$ with the largest cached gain from $Q$ [S1, unofficial].
            *   **Oracle Evaluation:** Recompute the true marginal gain $\Delta_v = \rho(\{v\} | S_q)$ by constructing and accumulating the Fisher information contribution $H_v$ [S1, unofficial].
            *   Update cached gain $\gamma_v \leftarrow \Delta_v$ [S1, unofficial].
            *   Re-insert $v$ into $Q$ with updated priority $\gamma_v$ [S1, unofficial].
        *   Until $\gamma_v = \max_{u \in P} \gamma_u$ (i.e., the popped view's true gain is greater than or equal to the cached gains of all other views in $Q$) [S1, unofficial].
    *   **Select and Update:**
        *   Add selected view $v$ to $S$: $S \leftarrow S \cup \{v\}$ [S1, unofficial].
        *   Update information matrix: $H \leftarrow H + H_v$ [S1, unofficial].
3.  **Return:** Selected set $S$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import random
import heapq

def log_det(M):
    """Computes the natural logarithm of the determinant of matrix M."""
    return np.linalg.slogdet(M)[1]

def compute_jacobian(view):
    """Computes the Jacobian J_v = d(f_v)/d(Theta) for the given view."""
    # Standard differentiable rendering computes this via backpropagation.
    return view.render_jacobian()

def compute_fisher_block(jacobian, sigma2_inv):
    """Computes H_v = sigma^-2 * J_v^T J_v."""
    return sigma2_inv * (jacobian.T @ jacobian)

def liTe_gs(h0, candidate_views, k, epsilon, delta, sigma2_inv):
    m = len(candidate_views)
    # Precondition H0 with regularization
    h = h0 + delta * np.eye(h0.shape[0])
    
    s = []
    remaining_views = list(candidate_views)
    cached_gains = {v: float('inf') for v in candidate_views}
    
    for q in range(1, k + 1):
        # Sample subset P of size p
        p_size = int((m / k) * np.log(1 / epsilon))
        p_size = min(p_size, len(remaining_views))
        p = random.sample(remaining_views, p_size)
        
        # Max-heap using negative gains
        heap = [(-cached_gains[v], v) for v in p]
        heapq.heapify(heap)
        
        selected_view = None
        while heap:
            neg_gain, v = heapq.heappop(heap)
            
            # Oracle evaluation: recompute true marginal gain
            jacobian = compute_jacobian(v)
            h_v = compute_fisher_block(jacobian, sigma2_inv)
            true_gain = log_det(h) - log_det(h + h_v)
            
            cached_gains[v] = true_gain
            
            # Check if true_gain >= max cached gain of remaining views in heap
            if heap:
                max_other_cached = -heap[0][0]
            else:
                max_other_cached = -float('inf')
                
            if true_gain >= max_other_cached:
                selected_view = v
                h = h + h_v
                break
            else:
                heapq.heappush(heap, (-true_gain, v))
                
        if selected_view is None:
            selected_view = p[0]
            jacobian = compute_jacobian(selected_view)
            h_v = compute_fisher_block(jacobian, sigma2_inv)
            h = h + h_v
            
        s.append(selected_view)
        remaining_views.remove(selected_view)
        
    return s
```

**Edge Cases and Pitfalls:**
*   **Ill-conditioning:** The prior information matrix $H_0$ can be ill-conditioned [S1, unofficial]. This is handled by preconditioning with a small regularization parameter $\delta$: $H_0 \leftarrow H_0 + \delta I$ [S1, unofficial].
*   **Subset Size:** The subset size $p$ depends on $\epsilon$ [S1, unofficial]. If $\epsilon$ is large, $p$ becomes small, potentially leading to poor selection quality if the subset does not contain informative views [S1, unofficial].
*   **Dynamic Resampling:** LiTe-GS dynamically resamples candidate subsets at each acquisition step [S1, unofficial]. This differs from Static-Subset Greedy (SSG), which samples once per stage [S1, unofficial]. Dynamic resampling prevents the algorithm from being permanently restricted to a potentially unfavorable subset realization [S1, unofficial].

## Subtopics

### POP-GS
A baseline method that uses the same Fisher-information-based objective but evaluates the full oracle over all candidate views at each iteration, resulting in higher oracle complexity [S1, unofficial].

### Static-Subset Greedy (SSG)
A baseline that samples a fixed subset of candidates once per acquisition stage and keeps it fixed, whereas LiTe-GS dynamically resamples candidate subsets at each acquisition step [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:**
    *   **Oracle Complexity:** Expected $O(M \log(1/\epsilon))$ oracle evaluations with respect to the number of candidate views $M$, independent of the selection cardinality $K$ [S1, unofficial].
    *   **Memory:** The block-diagonal approximation of the Fisher information matrix makes storage tractable, scaling with the number of Gaussian primitives $N$ but avoiding the full dense matrix [S1, unofficial].
*   **Measured Performance:**
    *   On Blender (Ship) with $K=6, \epsilon=0.1$: 208 oracle evaluations, PSNR 25.49, SSIM 0.830, LPIPS 0.158 [S1, unofficial].
    *   Increasing $K$ from 3 to 6 increases POP-GS evaluations from 275 to 549, while LiTe-GS remains nearly constant (210, 209, 208) [S1, unofficial].
    *   For fixed $K=6$, increasing $\epsilon$ from 0.1 to 0.6 reduces evaluations from 208 to 44, yielding 80.42% and 91.99% reductions respectively [S1, unofficial].
*   **Strengths and Limitations:**
    *   **Strengths:** Significantly reduces oracle evaluations compared to exhaustive methods like POP-GS while maintaining comparable reconstruction quality [S1, unofficial]. Provides an explicit trade-off between efficiency and accuracy via $\epsilon$ [S1, unofficial].
    *   **Limitations:** Approximation quality depends on $\epsilon$. A fixed subset (SSG) can occasionally outperform LiTe-GS if the fixed subset is favorable, but LiTe-GS is more robust across different scenes due to dynamic resampling [S1, unofficial].

## Background
LiTe-GS was proposed in the paper "LiTe-GS: Oracle-Efficient Next Best View Selection for 3D Gaussian Splatting" [S1, unofficial]. It addresses the computational bottleneck of information-driven view selection in 3DGS, where repeated evaluations of expensive information-gain oracles become prohibitive as the number of candidate views increases [S1, unofficial]. The method leverages stochastic submodular optimization techniques to design an efficient active-view selection strategy [S1, unofficial].

## Sources
[S1] LiTe-GS: Oracle-Efficient Next Best View Selection for 3D Gaussian Splatting, https://arxiv.org/abs/2609.30393, fetched 2026-10-29 (unofficial)
