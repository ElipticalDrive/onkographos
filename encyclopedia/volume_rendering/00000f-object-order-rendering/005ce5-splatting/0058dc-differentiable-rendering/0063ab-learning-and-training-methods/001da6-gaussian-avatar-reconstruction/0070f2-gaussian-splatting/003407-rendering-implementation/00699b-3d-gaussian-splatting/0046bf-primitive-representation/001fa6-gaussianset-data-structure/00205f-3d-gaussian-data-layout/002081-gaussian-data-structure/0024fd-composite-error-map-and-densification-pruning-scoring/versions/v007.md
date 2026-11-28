# Composite Error Map and Densification/Pruning Scoring

Also called: EffGS density control, Frequency-aware mask scoring

## Overview
A density control mechanism that uses a frequency-aware mask and photometric loss to compute densification and pruning scores for Gaussian primitives, reducing redundancy and improving fidelity compared to vanilla 3DGS [S1, unofficial]. It sets itself apart from its siblings by integrating a frequency-aware mask into the error computation to filter noise, and by weighting pruning scores with image-level photometric loss rather than raw pixel counts, whereas [Efficient Pruning Score](3598-efficient-pruning-score.md) focuses on second-order sensitivity and [Fisher Sensitivity Score](9664-fisher-sensitivity-score.md) uses Hessian approximations.

## How it works
**Inputs:**
- Set of Gaussian primitives $\mathcal{G}_t$ at update step $t$.
- $K$ distinct training views sampled uniformly without replacement.
- Rendered images $I_{\text{rend}}^j$ and ground-truth images $I_{\text{gt}}^j$ for each sampled view $j$.
- Frequency-aware mask $M_{\text{freq}}^j$ for each view $j$.
- Parameters: $\tau \in (0, 1)$ (error threshold), $\tau_p \in [0, 1)$ (pruning score threshold), $\tau_d \ge 0$ (densification score threshold), $\tau_s > 0$ (scale threshold), $\rho \in [0, 1]$ (pruning sampling ratio), $\lambda_{\text{dssim}}$ (SSIM weight).

**Outputs:**
- Updated set of Gaussian primitives $\mathcal{G}_{t+1}$.

**Data Structures:**
- $\mathcal{V}_i$: Set of view indices where Gaussian $i$ has a valid projected footprint.
- $\mathcal{A}_t$: Set of active primitives $\{i \in \mathcal{G}_t : |\mathcal{V}_i| > 0\}$.
- $\Omega_i^j$: Set of pixels where Gaussian $i$ contributes during compositing in view $j$.
- $M_{\text{err}}^j$: Composite error mask for view $j$.

**Steps:**

1. **Identify Active Primitives:**
   For each Gaussian $i \in \mathcal{G}_t$ and each sampled view $j \in \{1, \dots, K\}$:
   - Get projected radius $r_i^j$ from the rasterizer.
   - Determine visibility: $\mathcal{V}_i = \{j : r_i^j > 0\}$.
   - Count visible views: $|\mathcal{V}_i| = \sum_{j=1}^K \mathbb{I}[r_i^j > 0]$.
   - Define active set: $\mathcal{A}_t = \{i \in \mathcal{G}_t : |\mathcal{V}_i| > 0\}$.
   - If $\mathcal{A}_t = \emptyset$, skip density control.

2. **Compute Composite Error Map:**
   For each sampled view $j$:
   - Compute raw per-pixel RGB reconstruction error:
     $$e^j(\mathbf{u}) = \frac{1}{3} \sum_{c=1}^3 \left| I_{\text{rend},c}^j(\mathbf{u}) - I_{\text{gt},c}^j(\mathbf{u}) \right|$$
   - Normalize error over pixel grid $\mathcal{P}_j$:
     $$\bar{e}^j(\mathbf{u}) = \mathcal{N}_{\mathcal{P}_j}[e^j](\mathbf{u})$$
     where $\mathcal{N}_{\mathcal{P}_j}$ is min-max normalization. If $e^j$ is constant, map to zero.
   - Compute composite error mask $M_{\text{err}}^j(\mathbf{u})$:
     $$M_{\text{err}}^j(\mathbf{u}) = \mathbb{I} \left[ \bar{e}^j(\mathbf{u}) > \tau \vee \left( M_{\text{freq}}^j(\mathbf{u}) = 1 \wedge \bar{e}^j(\mathbf{u}) > 0.5\tau \right) \right]$$
     Note: The source text contains a typo $M_{\text{reeq}}$ which is corrected to $M_{\text{freq}}$ based on context [S1, unofficial].

3. **Compute Densification Scores:**
   For each $i \in \mathcal{A}_t$:
   - Compute weighted pixel count per view:
     $$n_i^j = \sum_{\mathbf{u} \in \Omega_i^j} M_{\text{err}}^j(\mathbf{u})$$
   - Compute valid-view average count:
     $$C_i = \frac{1}{|\mathcal{V}_i|} \sum_{j \in \mathcal{V}_i} n_i^j$$
   - Compute densification score:
     $$s_d^i(t) = \omega(t)C_i$$
     where $\omega(t)$ is a positive, nondecreasing clipped-linear schedule over the densification interval [S1, unofficial].

4. **Compute Pruning Scores:**
   For each $i \in \mathcal{A}_t$:
   - Compute image-level photometric loss for view $j$:
     $$E_{\text{photo}}^j = (1 - \lambda_{\text{dssim}}) \mathcal{L}_1^j + \lambda_{\text{dssim}} \left( 1 - \text{SSIM}(I_{\text{rend}}^j, I_{\text{gt}}^j) \right)$$
     where $\mathcal{L}_1^j = \frac{1}{|\mathcal{P}_j|} \sum_{\mathbf{u} \in \mathcal{P}_j} e^j(\mathbf{u})$.
   - Compute summed, loss-weighted count:
     $$Q_i = \sum_{j \in \mathcal{V}_i} n_i^j E_{\text{photo}}^j$$
   - Normalize pruning score over active set:
     $$s_p^i = \mathcal{N}_{\mathcal{A}_t}[Q](i)$$

5. **Execute Pruning:**
   - Identify mandatory removals:
     $$\mathcal{H}_t = \{i \in \mathcal{A}_t : o_i < 0.1 \vee \gamma_i < 0.01\}$$
     where $o_i$ is opacity and $\gamma_i$ is the learnable scale-modulation factor [S1, unofficial].
   - Identify candidates:
     $$\mathcal{P}_t = \{i \in \mathcal{A}_t \setminus \mathcal{H}_t : s_p^i > \tau_p\}$$
   - Sample $b_t = \lfloor \rho |\mathcal{P}_t| \rfloor$ candidates without replacement using weights proportional to $s_p^i$.
   - Removal set: $\mathcal{R}_t = \mathcal{H}_t \cup \text{sampled candidates}$.
   - Remove $\mathcal{R}_t$ from $\mathcal{G}_t$.

6. **Execute Densification:**
   - Select surviving active primitives with $s_d^i(t) > \tau_d$.
   - For each selected primitive $i$:
     - If largest base-scale component $< \tau_s$: Clone and shift along positional gradient.
     - Otherwise: Replace with two smaller Gaussians.
     - New primitives inherit parent's unconstrained modulation parameter $\beta_i \in \mathbb{R}$, with $\gamma_i = 2 \text{sigmoid}(\beta_i) \in (0, 2)$.
     - Remaining attributes follow clone/split updates.
   - Primitives outside $\mathcal{A}_t$ are unaffected.

**Edge Cases and Pitfalls:**
- When pruning and densification coincide, pruning must be applied first [S1, unofficial].
- New primitives are scored only at the next density update to avoid using post-update scores for pre-update decisions [S1, unofficial].
- Density control is skipped if $\mathcal{A}_t = \emptyset$ [S1, unofficial].

## Subtopics

### [Efficient Pruning Score](3598-efficient-pruning-score.md)
A sensitivity-based metric that reparameterizes the Hessian approximation to reduce storage requirements by 36×, enabling soft pruning during training. It sets itself apart from the Composite Error Map by focusing on second-order sensitivity to spatial parameters rather than frequency-aware photometric error.

### [Fisher Sensitivity Score](9664-fisher-sensitivity-score.md)
A pruning metric that estimates the impact of each Gaussian on the L2 reconstruction loss using a block-diagonal Hessian approximation. It sets itself apart by evaluating geometric sensitivity through the log-determinant of parameter gradients, avoiding bias toward larger Gaussians.

### [Multi-view Alpha-Weighted Error Accumulation](15393-multi-view-alpha-weighted-error-accumulation.md)
A technique for mapping 2D imagespace photometric errors back to 3D Gaussian primitives by aggregating alpha-weighted error metrics across multiple sampled views. It sets itself apart by ensuring only Gaussians with high visibility and significant contribution to reconstruction error are modified, avoiding overfitting to single-view occlusions.

## Operating characteristics
- **Time Complexity:** Dominated by rendering $K$ views and computing error masks. The error map computation is $O(K \cdot |\mathcal{P}|)$ where $|\mathcal{P}|$ is the number of pixels. Scoring is $O(|\mathcal{G}_t| \cdot K)$.
- **Memory Complexity:** Requires storing the composite error mask $M_{\text{err}}^j$ for each sampled view, adding $O(K \cdot |\mathcal{P}|)$ memory overhead.
- **Strengths:** Reduces redundancy by focusing on high-error, high-frequency regions. Improves fidelity by weighting pruning with photometric loss.
- **Limitations:** Requires rendering $K$ views for scoring, increasing training time. Sensitive to the choice of $\tau$ and $\lambda_{\text{dssim}}$.
- **When to use:** When high-fidelity rendering is required and computational budget allows for additional view sampling during training.
- **When not to use:** When real-time training is critical or memory is constrained.

## Background
Originated in EffGS (Efficient and High-Fidelity Gaussian Splatting) [S1, unofficial]. The method builds on [Gaussian Data Structure](2273-gaussian-data-structure.md) and integrates frequency-aware masking to improve density control.

## Sources
[S1] EffGS: Efficient and High-Fidelity Gaussian Splatting, https://arxiv.org/abs/2609.39553, fetched 2026-10-29 (unofficial)
