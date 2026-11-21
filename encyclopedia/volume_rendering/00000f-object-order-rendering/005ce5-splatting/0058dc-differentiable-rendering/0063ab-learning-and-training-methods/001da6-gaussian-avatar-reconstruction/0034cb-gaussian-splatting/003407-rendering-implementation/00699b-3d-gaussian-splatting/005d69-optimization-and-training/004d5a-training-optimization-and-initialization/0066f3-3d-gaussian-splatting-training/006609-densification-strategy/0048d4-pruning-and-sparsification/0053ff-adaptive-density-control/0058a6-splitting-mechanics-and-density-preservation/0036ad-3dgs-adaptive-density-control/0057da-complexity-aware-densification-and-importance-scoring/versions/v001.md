# Complexity-Aware Densification and Importance Scoring

Also called: Complexity-guided densification, Importance scoring for LoD

## Overview
A two-part mechanism for 3D Gaussian Splatting optimization that uses image complexity as a surrogate for reconstruction error to guide densification during training and select primitives for pruning during inference. Unlike sibling methods like [Age-Gated Eligibility (AGE)](12511-age-gated-eligibility-age.md) which may rely on opacity or age heuristics, this method explicitly biases capacity allocation toward complex regions to prevent under-reconstruction, while retaining large primitives in simple regions to maintain coverage. It trades off rendering speed at low budgets for higher quality and fewer holes compared to compact-primitive-focused methods like MGS.

## How it works
**Inputs:**
- Set of Gaussian primitives $G = \{g_j\}$, each with mean $\mu_j$, covariance $\Sigma_j$, opacity $\alpha_j$, and color.
- Training views $\Pi_i$ with associated image complexity maps $C_i^{2D}$.
- Optimization hyperparameters: EMA learning rate $\eta \in (0, 1]$, complexity dropout probability $\delta = 0.2$.
- Inference budget $\tau$ (fraction of primitives to retain).

**Outputs:**
- Updated Gaussian set with refined positions/opacities based on complexity-guided relocation.
- Pruned Gaussian set for inference based on importance scores.

**Data Structures:**
- Each Gaussian $g_j$ maintains a scalar complexity field $C_j$, initialized to 0.
- Complexity map $C_i^{2D}(u)$ is computed per view using delentropy [S1, unofficial].

**Steps:**

1. **Complexity Field Update (Training):**
   After each full-capacity training render of a view $\Pi_i$:
   a. Identify contributing Gaussians whose centers project into the current view’s frustum.
   b. For each contributing Gaussian $g_j$, compute its 2D projected position $\pi_i(\mu_j)$ in the image plane.
   c. Retrieve the complexity value $C_i^{2D}(\pi_i(\mu_j))$ from the precomputed complexity map for view $i$.
   d. Update the Gaussian's complexity field $C_j$ using exponential moving average:
      $$ C_j \leftarrow (1 - \eta) C_j + \eta C_i^{2D}(\pi_i(\mu_j)) $$
      [S1, unofficial]
   e. Gaussians not projecting into the frustum are not updated in this step. Over training, $C_j$ converges to a multi-view average complexity.

2. **Complexity-Aware Adaptive Density Control (Densification/Relocation):**
   When relocating or spawning new primitives (e.g., during splitting/cloning phases in 3DGS):
   a. Normalize the complexity field $C_j$ to $[0, 1]$ to obtain $\tilde{C}_j$.
   b. Apply complexity dropout: with probability $\delta = 0.2$, set $\tilde{C}_j = 0$. Otherwise, keep $\tilde{C}_j$. [S1, unofficial]
   c. Sample relocation destinations (or select candidates for splitting) with probability proportional to:
      $$ p(g_j) = \alpha_j \cdot \exp(\tilde{C}_j) $$
      [S1, unofficial]
   d. This multiplicative form ensures that primitives with high opacity are retained, while those in complex regions are favored for densification. The exponential term mimics the expected growth of particle count with complexity.

3. **Complexity-Aware Level of Details (Pruning/Inference):**
   At inference time, given a budget $\tau < 1$ (fraction of primitives to keep):
   a. Compute an importance score $S(g_j)$ for each Gaussian. The source notes that for low-complexity regions, opacity alone is insufficient as it does not capture spatial footprint. The method ranks primitives by a score that considers both contribution and spatial support. While Eq. 5 is referenced, the text emphasizes that in complex regions, the score reduces to opacity, whereas in simple regions, larger primitives are preferred. The exact formula for $S(g_j)$ is not fully detailed in the provided snippets, but the ranking logic prioritizes primitives that cover simple regions effectively (large footprint) and complex regions accurately (high opacity/contribution). [S1, unofficial]
   b. Sort primitives by $S(g_j)$ descending.
   c. Retain the top $\tau \cdot |G|$ primitives.

**Parameters:**
- $\eta$: EMA learning rate for complexity update. Typical value not specified, but $\eta \in (0, 1]$. [S1, unofficial]
- $\delta$: Complexity dropout probability. Default $\delta = 0.2$. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Collapse onto fixed complex regions:** Without dropout ($\delta=0$), the method may over-concentrate primitives in highly complex areas, neglecting simpler regions. Dropout mitigates this by occasionally scoring candidates by opacity alone. [S1, unofficial]
- **Low-budget performance:** At very low budgets (e.g., 0.1%), this method is slower than MGS because it retains large primitives that span more tiles, increasing per-frame cost. This is a deliberate trade-off for better coverage and fewer holes. [S1, unofficial]
- **Complexity Map Computation:** Requires efficient computation of $C_i^{2D}$ using delentropy. Errors here propagate to $C_j$. [S1, unofficial]

## Subtopics

### Complexity-Aware Adaptive Density Control
A density control mechanism that biases the relocation of Gaussian primitives towards regions with higher image complexity, using complexity as a surrogate for reconstruction error. It updates the complexity field $C_j$ via exponential moving average after each view render and samples relocation destinations with probability $p(g_j) = \alpha_j \cdot \exp(\tilde{C}_j)$, where $\tilde{C}_j$ is min-max normalized complexity. Complexity dropout ($\delta=0.2$) is applied to prevent collapse onto fixed complex regions. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** Training involves per-view complexity map updates and EMA updates for contributing Gaussians. Inference pruning requires sorting by importance score.
- **Memory Complexity:** Requires storing a scalar complexity field $C_j$ per Gaussian and potentially storing complexity maps for training views.
- **Measured Performance:** At 0.1% capacity on MipNerf360, the method achieves 463 FPS indoors and 874 FPS outdoors. For comparison, MGS achieves 829 FPS indoors and 1,151 FPS outdoors. At full capacity, it is tied with MGS indoors (302 FPS) and slightly ahead outdoors (198 vs. 193 FPS). [S1, unofficial]
- **Scaling:** As budget shrinks, performance accelerates for all methods, but this method saturates below 1% capacity, becoming slower than MGS at low budgets due to retained large primitives. [S1, unofficial]
- **Strengths:** Better coverage and quality, fewer holes, especially in simple regions where opacity-based pruning fails. Requires fewer primitives for equal quality. [S1, unofficial]
- **Limitations:** Slower rendering at low budgets compared to methods favoring compact primitives (like MGS). [S1, unofficial]
- **When to use:** When reconstruction quality and coverage are prioritized over maximum rendering speed, especially at low primitive budgets. [S1, unofficial]
- **When not to use:** When maximum FPS is critical and some hole-filling or lower quality in simple regions is acceptable. [S1, unofficial]

## Background
Originated in the KoalaGS paper, which introduces complexity-aware continuous level of detail for Gaussian Splatting. The method addresses limitations of opacity-based densification and pruning by incorporating image complexity. [S1, unofficial]

## Sources
[S1] KoalaGS: Complexity-Aware Continuous Level of Detail for Gaussian Splatting, https://arxiv.org/abs/2610.28615, fetched 2026-10-26 (unofficial)
