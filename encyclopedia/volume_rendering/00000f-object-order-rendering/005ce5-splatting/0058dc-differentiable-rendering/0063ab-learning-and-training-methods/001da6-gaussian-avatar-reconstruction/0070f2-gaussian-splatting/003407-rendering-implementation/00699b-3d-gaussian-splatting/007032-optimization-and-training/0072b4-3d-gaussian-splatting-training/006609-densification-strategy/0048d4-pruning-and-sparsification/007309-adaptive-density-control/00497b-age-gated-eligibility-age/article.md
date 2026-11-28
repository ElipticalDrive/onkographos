# Age-Gated Eligibility (AGE)
Also called: AGE (Age-Gated Densification), cost-control gate, maturity window

## Overview
Age-Gated Eligibility (AGE) is a densification control mechanism for 3D Gaussian Splatting that prevents newly created primitives from undergoing split or clone operations immediately after birth [S1]. It solves the problem of uncontrolled cascading growth, where a Gaussian born at iteration $k$ splits at $k+1$, its child splits at $k+2$, and so on, which inflates the primitive count, peak memory usage, and training time in dense regions [S1]. Unlike standard adaptive density control which re-evaluates all primitives regardless of age, AGE defers the first split-or-clone decision until a Gaussian has survived a fixed maturity window, thereby damping the densification cascade and bounding the final primitive budget without degrading image quality [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of Gaussians $\mathcal{G}$, each tagged with a birth iteration $b_i$; the current optimization iteration $t$; the densification cadence $\Delta_d$; the maturity threshold $T_{mat}$; and the gradient threshold $\tau_g$ [S1].
*   **Output:** A subset of Gaussians eligible for densification (split or clone) in the current cycle [S1].

**Data Structures:**
Each Gaussian $i$ in the set $\mathcal{G}$ maintains a scalar field $b_i$, representing the iteration index at which the Gaussian was created (for initial points) or last split/clone [S1].

**Parameters:**
*   $T_{mat}$: Maturity threshold in iterations. Default value: 200 [S1]. This is fixed by design as $2\Delta_d$ [S1].
*   $\Delta_d$: Densification cadence (interval between densification steps). Default value: 100 [S1].
*   $\tau_g$: Threshold for accumulated 2D-projected positional gradient magnitude. Standard 3DGS value [S1].

**Algorithm:**
1.  **Initialization:** At the start of training (iteration 0), tag every initial Gaussian $i$ with its birth iteration $b_i = 0$ [S1].
2.  **Iteration Loop:** During optimization, every $\Delta_d$ iterations, execute the densification cycle [S1].
3.  **Age Calculation:** For each Gaussian $i$, calculate its current age $\nu_i(t)$ at iteration $t$ using the formula:
    $$ \nu_i(t) = t - b_i $$
    [S1]
4.  **Eligibility Check:** Determine the mature set $M(t)$ containing all Gaussians that have reached the maturity threshold:
    $$ M(t) = \{ i : \nu_i(t) \ge T_{mat} \} $$
    [S1]
5.  **Densification:** For each Gaussian $i \in M(t)$, evaluate the standard 3DGS split-or-clone criteria [S1]:
    *   Check if the accumulated 2D-projected positional gradient magnitude exceeds $\tau_g$ [S1].
    *   If the gradient condition is met:
        *   If the Gaussian is "large" (scale above a certain threshold), perform a **split** operation [S1].
        *   If the Gaussian is "small" (scale below the threshold), perform a **clone** operation [S1].
    *   For split operations, the new child Gaussians are tagged with the current iteration $t$ as their birth iteration $b_{new} = t$ [S1].
    *   For clone operations, the clone retains the birth iteration of the parent or is tagged with $t$ (standard 3DGS behavior applies, but AGE only gates the *eligibility* to perform the operation) [S1].
6.  **Skip Immature:** Gaussians $i$ where $\nu_i(t) < T_{mat}$ are skipped in this cycle [S1]. They are re-evaluated in the next densification opportunity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def age_gated_densification(gaussians, current_iteration, T_mat, delta_d, tau_g):
    """
    Perform age-gated densification for 3D Gaussian Splatting.
    
    Args:
        gaussians: List of Gaussian objects, each having attributes:
                   - b_i: birth iteration (int)
                   - accumulated_grad: accumulated 2D-projected positional gradient magnitude (float)
                   - scale: scale metric to determine large vs small (float)
                   - opacity: opacity value (float)
        current_iteration: The current optimization iteration number (int)
        T_mat: Maturity threshold in iterations (int, default 200)
        delta_d: Densification cadence (int, default 100)
        tau_g: Gradient threshold for densification (float)
        
    Returns:
        Updated list of Gaussians (with new children added if split/clone occurred)
    """
    if current_iteration % delta_d != 0:
        return gaussians

    final_gaussians = []
    for g in gaussians:
        age = current_iteration - g.b_i
        if age < T_mat:
            final_gaussians.append(g)
            continue

        if g.accumulated_grad > tau_g:
            if is_large(g.scale):
                children = split_gaussian(g, current_iteration)
                final_gaussians.extend(children)
            else:
                clone = clone_gaussian(g, current_iteration)
                final_gaussians.append(clone)
        else:
            final_gaussians.append(g)

    return final_gaussians

def is_large(scale):
    # Standard 3DGS scale threshold check [G]
    return scale > 0.01

def split_gaussian(g, t):
    # Standard 3DGS split: creates two children with b_i = t [G]
    return [Gaussian(b_i=t, scale=g.scale / 1.6, opacity=g.opacity / 2.0),
            Gaussian(b_i=t, scale=g.scale / 1.6, opacity=g.opacity / 2.0)]

def clone_gaussian(g, t):
    # Standard 3DGS clone: creates one child with b_i = t [G]
    return Gaussian(b_i=t, scale=g.scale, opacity=g.opacity)
```

**Edge Cases and Pitfalls:**
*   **Initial Primitives:** Under the standard 3DGS schedule, densification begins at iteration 500 [S1]. Since $T_{mat}=200$, initial primitives (tagged $b_i=0$) have an age of 500, which is $\ge T_{mat}$ [S1]. Thus, initial primitives are already mature at the first densification cycle [S1].
*   **Early Densification Schedules:** If a training schedule densifies earlier than iteration 500 (e.g., at iteration 100), initial primitives would be immature ($100 - 0 < 200$) [S1]. The tag $b_i$ is retained so AGE remains well-defined for such schedules [S1].
*   **Cascade Prevention:** Without AGE, a Gaussian born at step $k$ can split at $k+1$, its child at $k+2$, causing an uncontrolled cascade [S1]. AGE ensures that with $T_{mat}=200$ and $\Delta_d=100$, a Gaussian can split or clone at most once every two densification cycles, converting an unbounded cascade into controlled refinement [S1].

## Subtopics

### [Spawn Gate](15087-spawn-gate.md)
The Spawn Gate is a mechanism that predicts the survival probability of a Gaussian spawn candidate at birth time to veto low-survival candidates, thereby controlling the peak population size during training. It sets itself apart from heuristic densification rules by using a learned predictor to rank candidates, ensuring a deterministic peak-memory reduction without degrading final image quality.

## Operating characteristics
**Time and Memory Complexity:**
*   AGE adds no architectural change and no per-iteration overhead beyond a single age comparison ($\nu_i(t) \ge T_{mat}$) per Gaussian [S1].
*   The added cost is not an early-training overhead: the first 500 iterations take at most one minute in every scene [S1].

**Measured Performance:**
*   On standard benchmarks, enabling AGE reduces the final Gaussian count from 3.45 M to 3.15 M (an 8.7% reduction) [S1].
*   Training time is lowered by 2.4 minutes (from 31.8 to 29.4 minutes) [S1].
*   Peak memory is reduced by 0.47 GB (from 6.04 to 5.57 GB) [S1].
*   Image quality remains unchanged within measurement noise (PSNR +0.05 dB, SSIM +0.001, LPIPS +0.001) [S1].
*   AGE lowers the Gaussian count during densification by 16–23% relative to DART alone at iteration 5k on scenes like bicycle, stump, and drjohnson [S1].

**Strengths and Limitations:**
*   **Strengths:** Bounding the primitive budget, reducing memory and training time, preventing cascading splits in dense regions, and acting as a general control independent of how the initial cloud was formed (works with or without DART augmentation) [S1].
*   **Limitations:** Requires tuning of $T_{mat}$ and $\Delta_d$ (though fixed by design in the source) [S1]. It does not improve quality directly but preserves it while reducing cost [S1].

**When to Use:**
*   Use AGE when training 3D Gaussian Splatting models to control memory usage and training time, especially when using denser initialization strategies like DART that might otherwise inflate the primitive count [S1].

## Background
Age-Gated Eligibility (AGE) was introduced in the paper "DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting" [S1]. It is designed to work in conjunction with Distance-Aware Replication Training (DART) to bound the primitive budget that DART's denser initialization might otherwise inflate [S1]. AGE acts as a cost-control gate, distinct from relocation [9] and gradient-criterion reweighting [4] [S1].

## Sources
[S1] DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting, https://europepmc.org/article/PMC/PMC13705904, fetched 2026-10-27
