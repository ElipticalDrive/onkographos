# Age-Gated Eligibility (AGE)
Also called: AGE (Age-Gated Densification), cost-control gate, maturity window

## Overview
Age-Gated Eligibility (AGE) is a densification strategy for 3D Gaussian Splatting that prevents newly created primitives from splitting or cloning immediately, thereby bounding the primitive budget and controlling memory and training time costs [S1]. It solves the problem of uncontrolled cascading growth where a Gaussian born at iteration $k$ splits at $k+1$, its child at $k+2$, and so on, which inflates the primitive count and leaves redundant, under-supported primitives in dense regions [S1]. Unlike sibling methods that rely on gradient reweighting or relocation, AGE acts as a temporal gate, deferring the first split-or-clone decision until a Gaussian has matured for a fixed number of iterations [S1].

## How it works
AGE modifies the adaptive density control (ADC) step of 3D Gaussian Splatting by introducing an age-based eligibility check [S1]. It requires no architectural changes to the rasterizer or pruning criteria, adding only a single age comparison per Gaussian per densification cycle [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Current training iteration $t$.
    *   Set of all Gaussians $\mathcal{G}$, where each Gaussian $i$ has a birth iteration tag $b_i$.
    *   Densification cadence $\Delta_d$ (interval between densification steps).
    *   Maturity threshold $T_{mat}$ (minimum age required for eligibility).
    *   Gradient threshold $\tau_g$ (standard 3DGS threshold for accumulated 2D-projected positional gradient magnitude).
*   **Outputs:**
    *   Updated set of Gaussians $\mathcal{G}'$ with new primitives added via split or clone operations only for eligible Gaussians.

### Data Structures
Each Gaussian $i$ in the scene must store an additional integer field:
*   $b_i$: The birth iteration. This is the iteration index at which the Gaussian was created (either at initialization or via a previous split/clone operation).
    *   For the initial set of Gaussians, $b_i = 0$ [S1].

### Parameters
*   $T_{mat}$: Maturity threshold. Default value is 200 iterations [S1]. This is typically set to $2\Delta_d$ [S1].
*   $\Delta_d$: Densification cadence. Default value is 100 iterations [S1].
*   $\tau_g$: Gradient magnitude threshold. Uses the standard value from 3DGS [S1].

### Algorithm
The method operates during the optimization loop. The densification step is executed every $\Delta_d$ iterations [S1].

1.  **Initialization:** At the start of training (iteration 0), tag every initial Gaussian $i$ with $b_i = 0$ [S1].
2.  **Iteration Loop:** For each training iteration $t$:
    *   Perform standard forward and backward passes to accumulate gradients [S1].
    *   If $t \pmod{\Delta_d} == 0$ (i.e., it is a densification step):
        1.  Define the **mature set** $M(t)$ as the subset of Gaussians $\mathcal{G}$ where the age $\nu_i(t) \ge T_{mat}$ [S1].
        2.  The age of Gaussian $i$ at iteration $t$ is calculated as:
            $$ \nu_i(t) = t - b_i $$
        3.  A Gaussian $i$ is eligible for densification if and only if:
            $$ \nu_i(t) \ge T_{mat} $$
        4.  Iterate through all Gaussians $i \in \mathcal{G}$:
            *   If $i \notin M(t)$ (immature): Skip densification checks for this Gaussian. It remains in the scene and will be re-evaluated in the next densification cycle [S1].
            *   If $i \in M(t)$ (mature):
                *   Check if the accumulated 2D-projected positional gradient magnitude $g_i$ exceeds $\tau_g$ [S1].
                *   If $g_i > \tau_g$:
                    *   If the Gaussian is "small" (below a scale threshold defined in standard 3DGS): **Clone** the Gaussian [S1].
                    *   If the Gaussian is "large" (above a scale threshold defined in standard 3DGS): **Split** the Gaussian [S1].
                *   If $g_i \le \tau_g$: Do not densify.
        5.  For any new Gaussian $j$ created by a split or clone operation at iteration $t$:
            *   Set its birth tag $b_j = t$ [S1].
            *   Note: This new Gaussian is now immature and will not be eligible for densification until $t + T_{mat}$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def age_gated_densification(current_iteration, gaussians, delta_d, t_mat, tau_g):
    """
    Performs age-gated densification for 3D Gaussian Splatting.
    
    Args:
        current_iteration (int): The current training iteration t.
        gaussians (list): List of Gaussian objects. Each object has attributes:
            - birth_iter (int): b_i
            - grad_mag (float): Accumulated 2D-projected positional gradient magnitude g_i
            - scale (float): Current scale of the Gaussian
            - is_clone (bool): Whether this Gaussian is a clone (for split/clone logic)
        delta_d (int): Densification cadence (default 100).
        t_mat (int): Maturity threshold (default 200).
        tau_g (float): Gradient threshold for densification.
        
    Returns:
        list: Updated list of Gaussians.
    """
    # Only run densification at specific intervals
    if current_iteration % delta_d != 0:
        return gaussians

    new_gaussians = []
    mature_gaussians = []

    # Step 1: Identify mature Gaussians
    for g in gaussians:
        age = current_iteration - g.birth_iter
        if age >= t_mat:
            mature_gaussians.append(g)
        # Immature Gaussians are skipped and remain in the scene implicitly
        # by not being processed for split/clone in this step

    # Step 2: Apply standard densification only to mature Gaussians
    for g in mature_gaussians:
        if g.grad_mag > tau_g:
            # Standard 3DGS logic for split vs clone based on scale
            # Note: Exact scale thresholds are defined in the parent 3DGS ADC method
            if is_small_gaussian(g.scale):
                # Clone operation
                child = clone_gaussian(g)
                child.birth_iter = current_iteration # Tag new Gaussian with current iteration
                new_gaussians.append(child)
            else:
                # Split operation
                children = split_gaussian(g)
                for child in children:
                    child.birth_iter = current_iteration # Tag new Gaussians with current iteration
                    new_gaussians.append(child)
        else:
            # Gradient too low, no densification
            pass

    # Combine original (non-densified) and new Gaussians
    # Note: In a real implementation, 'gaussians' list is updated in-place or replaced.
    # The 'mature_gaussians' that did not split/clone remain in the scene.
    # The 'new_gaussians' are added.
    return gaussians + new_gaussians
```

### Edge Cases and Pitfalls
*   **Initial Primitives:** Under the standard 3DGS schedule, densification begins at iteration 500. Since $T_{mat} = 200$, initial primitives (with $b_i=0$) have an age of 500, which is $\ge T_{mat}$. Thus, initial primitives are always mature at the first densification step [S1].
*   **Early Densification Schedules:** If a training schedule densifies before iteration 200 (e.g., at iteration 100), initial primitives would be immature. The gate still applies correctly because $b_i$ is retained and checked against $t$ [S1].
*   **Cascade Prevention:** Without AGE, a Gaussian born at step $k$ can split at $k+1$, its child at $k+2$, causing an uncontrolled cascade. AGE ensures that a Gaussian must survive $T_{mat}$ iterations before its first split/clone, limiting the maximum cascading rate [S1]. With $T_{mat}=200$ and $\Delta_d=100$, a Gaussian can split at most once every two densification cycles [S1].

## Operating characteristics
*   **Time Complexity:** Adds negligible overhead. The cost is a single integer comparison ($t - b_i \ge T_{mat}$) per Gaussian per densification cycle [S1]. The first 500 iterations take at most one minute in every scene, indicating no early-training overhead [S1].
*   **Memory Complexity:** No additional architectural memory overhead. The only extra storage is the integer $b_i$ per Gaussian [S1].
*   **Performance:**
    *   Reduces final Gaussian count by 8.7% (from 3.45M to 3.15M) compared to DART alone [S1].
    *   Lowers training time by 2.4 minutes (31.8 to 29.4 minutes) [S1].
    *   Lowers peak memory by 0.47 GB (6.04 to 5.57 GB) [S1].
    *   Quality remains unchanged within measurement noise (PSNR +0.05 dB, SSIM +0.001, LPIPS +0.001) [S1].
    *   Reduces Gaussian count during densification by 16–23% relative to DART alone at iteration 5k on specific scenes (bicycle, stump, drjohnson) [S1].
*   **Strengths:**
    *   Bounding the primitive budget without sacrificing quality [S1].
    *   Suppresses under-supported primitives that result from unconstrained cascading [S1].
    *   Backbone-agnostic: Works with vanilla 3DGS and anchor-based methods like Scaffold-GS [S1].
    *   Independent of how the initial cloud was formed (SfM or DART-augmented) [S1].
*   **Limitations:**
    *   Requires tuning of $T_{mat}$ and $\Delta_d$ if the densification schedule changes significantly, though the default $T_{mat}=2\Delta_d$ is robust [S1].
    *   Does not address the initial distribution of Gaussians (this is handled by the parent method, e.g., DART) [S1].

## Background
Age-Gated Eligibility (AGE) was introduced in the paper "DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting" [S1]. It was developed to address the issue of primitive budget inflation caused by denser initialization methods like Distance-Aware Replication (DART) [S1]. The method is distinct from relocation techniques [9] and gradient-criterion reweighting [4] [S1]. It is often used in conjunction with the compactness regularizer from 3D-GS-MCMC [9] [S1].

## Sources
[S1] DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting, https://europepmc.org/article/PMC/PMC13705904, fetched 2026-10-27
