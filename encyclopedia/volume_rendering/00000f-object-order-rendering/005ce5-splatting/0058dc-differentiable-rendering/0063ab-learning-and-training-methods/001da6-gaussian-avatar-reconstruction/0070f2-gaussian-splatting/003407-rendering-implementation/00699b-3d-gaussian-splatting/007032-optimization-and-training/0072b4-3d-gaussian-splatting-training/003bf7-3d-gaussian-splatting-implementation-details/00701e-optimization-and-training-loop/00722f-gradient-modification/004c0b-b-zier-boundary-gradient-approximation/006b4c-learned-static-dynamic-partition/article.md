# Learned Static–Dynamic Partition

Also called: Binary gate optimization, Hard-concrete relaxation gating

## Overview
A method to jointly optimize a binary gate $g_i$ and motion parameters to classify Gaussians as static or dynamic, using a hard-concrete relaxation with straight-through estimator [S1, unofficial]. It solves the problem of incorrectly treating moving Gaussians as static, which leaves frozen copies of moving objects in the scene, by learning a sparse static-dynamic partition [S1, unofficial]. It sets itself apart from sibling methods by explicitly maintaining a binary classification per Gaussian that is committed and frozen after training, enabling efficient rendering of static background without temporal overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Gaussian parameters (position, scale, rotation, opacity, color, velocity), photometric loss gradients, temperature schedule parameters.
- **Outputs:** Binary gate values $g_i \in \{0, 1\}$ for each Gaussian, where $g_i=1$ indicates dynamic and $g_i=0$ indicates static.

**Data Structures:**
Each Gaussian $i$ maintains:
- $\ell_i$: Learnable gate logit (scalar).
- $g_i$: Binary gate value (scalar, 0 or 1).
- $g_{bi}$: Relaxed gate value for gradients (scalar, continuous).
- $e_i$: Promotion evidence accumulator (scalar, exponential moving average).
- `forced_dynamic`: Boolean flag.
- `committed_label`: Boolean flag indicating if the partition is frozen.

**Parameters:**
- $T$: Temperature, annealed from $T_{init}=2.0$ to $T_{final}=0.1$ [S1, unofficial].
- $[\gamma, \zeta] = [-0.1, 1.1]$: Relaxed range bounds [S1, unofficial].
- $\tau_m$: Velocity threshold for forced dynamic classification [S1, unofficial].
- $\eta$: Momentum factor for evidence EMA [S1, unofficial].
- Training phases: Logistic noise for first 5k iterations; deterministic evaluation after; partition committed at 25k iterations [S1, unofficial].

**Steps:**

1. **Initialization:**
   - Initialize $\ell_i$ for all Gaussians.
   - If velocity initialization $v_i$ is available and $\|v_i\|_2 > \tau_m$, set `forced_dynamic` = True and $g_i = 1$.
   - If no velocity initialization, start with strong dynamic prior (or static prior depending on implementation details, but typically dynamic to avoid freezing) [S1, unofficial].
   - Initialize $e_i = 0$.

2. **Forward Pass (Training):**
   - Sample $u_i \sim U(0, 1)$.
   - Compute relaxed sigmoid:
     $$ s_i = \text{sigmoid}\left(\frac{\ell_i + \log \frac{u_i}{1-u_i}}{T}\right) $$
   - Clamp to relaxed range:
     $$ \tilde{s}_i = \text{clamp}(s_i(\zeta - \gamma) + \gamma, 0, 1) $$
   - Compute binary gate:
     $$ g_i = \mathbb{I}[\tilde{s}_i \geq 0.5] $$
   - Compute straight-through estimator for gradients:
     $$ g_{bi} = \text{sg}(g_i) - \text{sg}(\tilde{s}_i) + \tilde{s}_i $$
   - Use $g_{bi}$ in motion parameter updates (Eqs. 19 and 22 in source) [S1, unofficial].

3. **Backward Pass:**
   - Compute loss gradients $\frac{\partial L}{\partial \ell_i}$.
   - Remove gradient contribution from gate regularizers before computing evidence [S1, unofficial].

4. **Evidence Update (Motion Evidence and Promotion):**
   - Update exponential moving average of evidence:
     $$ e_i \leftarrow \eta e_i + (1 - \eta) \text{ReLU}\left(-\frac{\partial L}{\partial \ell_i}\right) $$
   - Negative gradient indicates evidence for motion [S1, unofficial].
   - After 2k warmup, check static population every 500 iterations [S1, unofficial].
   - Promote Gaussians with $e_i$ above the 0.995 quantile, up to 1% of all Gaussians per step [S1, unofficial].
   - Once promoted or forced, a Gaussian cannot revert to static [S1, unofficial].

5. **Sparsity Regularization:**
   - Apply sparsity loss to free Gaussians (excluding forced/promoted dynamic ones):
     $$ L_{sparse} = \frac{1}{|F|} \sum_{i \in F} \text{sigmoid}\left(\frac{\ell_i - T \log \frac{-\gamma}{\zeta}}{T}\right) $$
     $$ L_{bin} = \frac{1}{|F|} \sum_{i \in F} p_i(1 - p_i), \quad p_i = \text{sigmoid}(\ell_i) $$
   - Introduce sparsity loss gradually from iterations 5k to 20k [S1, unofficial].

6. **Densification:**
   - When cloning or splitting Gaussians, new copies inherit $\ell_i$, `forced_dynamic`, `committed_label`, and $e_i$ from the parent [S1, unofficial].

7. **Commitment:**
   - At iteration 25k, freeze the static-dynamic partition.
   - Renderer and compression operate on fixed binary partition $g_i$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_gate(gaussian, loss_grad, T, eta, iteration, total_gaussians):
    """
    Update the static-dynamic partition for a single Gaussian.
    gaussian: object with attributes l_i, g_i, g_bi, e_i, forced_dynamic, committed_label
    loss_grad: gradient of loss w.r.t. l_i (after removing regularizer contributions)
    T: current temperature
    eta: EMA momentum
    iteration: current training iteration
    total_gaussians: total number of Gaussians in scene
    """
    if gaussian.committed_label:
        return  # Partition is frozen

    # Forward pass
    u_i = random.uniform(0, 1)
    s_i = sigmoid((gaussian.l_i + log(u_i / (1 - u_i))) / T)
    gamma, zeta = -0.1, 1.1
    s_tilde = clamp(s_i * (zeta - gamma) + gamma, 0, 1)
    gaussian.g_i = 1 if s_tilde >= 0.5 else 0
    gaussian.g_bi = gaussian.g_i - s_tilde + s_tilde  # Straight-through: sg(g_i) - sg(s_tilde) + s_tilde

    # Evidence update (only if not forced dynamic)
    if not gaussian.forced_dynamic:
        # Remove regularizer gradient contribution from loss_grad here [G]
        evidence_grad = -loss_grad
        gaussian.e_i = eta * gaussian.e_i + (1 - eta) * max(0, evidence_grad)

    # Promotion check (every 500 iterations after 2k warmup)
    if iteration > 2000 and iteration % 500 == 0:
        if not gaussian.forced_dynamic and gaussian.g_i == 0:
            # Check if this Gaussian should be promoted
            # Note: In practice, this requires global quantile calculation
            # Here we simulate the logic for a single Gaussian
            pass  # Global promotion logic handled externally

    # Forced dynamic initialization
    if gaussian.forced_dynamic:
        gaussian.g_i = 1
        gaussian.g_bi = 1

def promote_gaussians(gaussians, iteration, total_gaussians):
    """
    Promote static Gaussians to dynamic based on evidence.
    """
    if iteration <= 2000 or iteration % 500 != 0:
        return

    # Collect static, non-forced Gaussians
    static_candidates = [g for g in gaussians if not g.forced_dynamic and g.g_i == 0]
    if not static_candidates:
        return

    # Calculate 0.995 quantile of evidence
    evidence_values = [g.e_i for g in static_candidates]
    threshold = np.percentile(evidence_values, 99.5)

    # Promote up to 1% of total Gaussians
    max_promotions = int(0.01 * total_gaussians)
    promoted = 0
    for g in static_candidates:
        if g.e_i > threshold and promoted < max_promotions:
            g.forced_dynamic = True
            g.g_i = 1
            g.g_bi = 1
            promoted += 1

def commit_partition(gaussians, iteration):
    """
    Freeze the static-dynamic partition at iteration 25k.
    """
    if iteration == 25000:
        for g in gaussians:
            g.committed_label = True
            g.g_bi = g.g_i  # Ensure forward pass uses binary value
```

## Subtopics

### Motion Evidence and Promotion
A mechanism to prevent moving Gaussians from being incorrectly classified as static by using velocity initialization and reconstruction evidence [S1, unofficial]. A Gaussian is forced to be dynamic if its initialized velocity satisfies $\|v_i\|_2 > \tau_m$. Once forced/promoted, it cannot revert to static [S1, unofficial]. Evidence $e_i$ is updated as $e_i \leftarrow \eta e_i + (1 - \eta) \text{ReLU}\left(-\frac{\partial L}{\partial \ell_i}\right)$ [S1, unofficial]. Promote Gaussians whose evidence is above the 0.995 quantile, at most 1% per step, after 2k warmup, checking every 500 iterations [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per iteration for gate updates, where $N$ is the number of Gaussians. Promotion check is $O(N \log N)$ due to quantile calculation, but only every 500 iterations.
- **Memory Complexity:** $O(N)$ for storing $\ell_i$, $g_i$, $e_i$, and flags per Gaussian.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of Gaussians. The commitment at 25k iterations allows for efficient rendering without temporal overhead for static Gaussians.
- **Strengths:** Prevents frozen artifacts of moving objects; enables efficient rendering of static scenes; sparse dynamic partition reduces computational cost for temporal updates.
- **Limitations:** Requires careful tuning of temperature schedule and promotion thresholds; commitment is irreversible, so misclassifications after 25k cannot be corrected.
- **When to use:** When rendering scenes with both static and dynamic elements, especially on mobile or resource-constrained devices where temporal updates are expensive.
- **When not to use:** When all Gaussians are known to be static or dynamic, or when real-time adaptation to changing dynamics is required after training.

## Background
Originated in Mobile-4DGS: Unified Static-Dynamic Real-time Mobile Gaussian Splatting [S1, unofficial]. The method addresses the inefficiency of modeling temporal variation for static background elements like floors and stationary objects, while avoiding the artifact of frozen moving objects [S1, unofficial]. It builds on [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md) to handle the non-differentiable binary gate.

## Sources
[S1] Mobile-4DGS: Unified Static-Dynamic Real-time Mobile Gaussian Splatting, https://arxiv.org/abs/2610.05289, fetched 2026-10-29 (unofficial)
