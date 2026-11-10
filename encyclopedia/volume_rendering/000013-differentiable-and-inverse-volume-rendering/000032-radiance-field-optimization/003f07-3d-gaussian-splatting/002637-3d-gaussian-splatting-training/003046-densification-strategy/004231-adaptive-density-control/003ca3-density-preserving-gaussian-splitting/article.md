# Density-preserving Gaussian splitting

Also called: BOGausS splitting, density-preserving split

## Overview
Density-preserving Gaussian splitting is a densification strategy for 3D Gaussian Splatting that modifies child parameters during splitting to match the density of the parent Gaussian, thereby preventing loss perturbations and Gaussian clustering [S1]. It distinguishes itself from standard 3DGS by discarding cloning operations entirely and relying solely on splitting with learned opacity and size adaptations, unlike [17] which ignores opacity variations [S1].

## How it works
**Inputs and Outputs:**
- Input: A parent Gaussian $G_m$ defined by mean $\mu_m$, covariance $\Sigma_m$ (or scale $\sigma_m$ and rotation $R_m$), opacity $o_m$, and color $c_m$.
- Output: Two child Gaussians $G_{c1}, G_{c2}$ with updated parameters $(\mu_{c}, \sigma_{c}, o_{c}, c_{c})$.

**Data Structures:**
Each Gaussian is represented by:
- $\mu \in \mathbb{R}^3$: Mean position.
- $\sigma \in \mathbb{R}^3$: Scale vector (diagonal elements of the scale matrix in local frame).
- $q \in \mathbb{R}^4$: Rotation quaternion.
- $o \in [0, 1]$: Opacity.
- $c \in \mathbb{R}^3$: Spherical harmonics coefficients or RGB color.

**Steps:**
1. **Identify Split Axis:** Determine the principal axis with the largest scale value $\sigma_{max}$ from the parent's scale vector $\sigma_m$. Let this axis index be $k$.
2. **Shift Means:** Create two child means by shifting the parent mean $\mu_m$ along the longest axis by $\pm \alpha \sigma_{max}$:
   $$ \mu_{c1} = \mu_m + \alpha \sigma_{max} \mathbf{e}_k $$
   $$ \mu_{c2} = \mu_m - \alpha \sigma_{max} \mathbf{e}_k $$
   where $\mathbf{e}_k$ is the unit vector along axis $k$ in the local coordinate frame, and $\alpha$ is a constant [S1].
3. **Copy Rotation and Color:** The children inherit the parent's rotation $q_m$ and color $c_m$ [S1].
4. **Adapt Scale and Opacity:** Calculate new scales $\sigma_{c1}, \sigma_{c2}$ and opacities $o_{c1}, o_{c2}$ using a learned adaptation function `SplitAdaptation` [S1]:
   $$ (\sigma_{c}, o_{c}) = \text{SplitAdaptation}(\sigma_m, o_m) $$
   This function ensures the sum of the densities of the two children approximates the density of the parent [S1]. The adaptation is pre-computed or learned offline [S1].
5. **Discard Parent:** Remove the parent Gaussian from the scene representation.

**Parameters:**
- $\alpha$: Constant controlling mean shift magnitude. Typical value: $0.3$ [S1].

**Pitfalls:**
- Original splitting causes loss perturbations; this method mitigates this by preserving density [S1].
- Cloning is discarded entirely to limit perturbations and prevent clusters [S1].
- The adaptation function must be robust; if not learned correctly, density mismatches may occur [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def density_preserving_split(parent_gaussian, alpha=0.3):
    """
    Performs density-preserving splitting on a single Gaussian.
    
    Args:
        parent_gaussian: Object with attributes .mu (3,), .sigma (3,), .q (4,), .o (float), .c (3,)
        alpha: Float, mean shift multiplier (default 0.3)
        
    Returns:
        List of two child Gaussian objects
    """
    # Step 1: Identify longest axis
    sigma = parent_gaussian.sigma
    axis_idx = np.argmax(np.abs(sigma))
    max_scale = sigma[axis_idx]
    
    # Define local axis vector
    local_axes = [np.array([1, 0, 0]), np.array([0, 1, 0]), np.array([0, 0, 1])]
    e_k = local_axes[axis_idx]
    
    # Step 2: Shift means
    shift = alpha * max_scale * e_k
    mu1 = parent_gaussian.mu + shift
    mu2 = parent_gaussian.mu - shift
    
    # Step 3: Copy rotation and color
    q1 = parent_gaussian.q.copy()
    q2 = parent_gaussian.q.copy()
    c1 = parent_gaussian.c.copy()
    c2 = parent_gaussian.c.copy()
    
    # Step 4: Adapt scale and opacity
    # SplitAdaptation is a learned function mapping (sigma_m, o_m) -> (sigma_c, o_c)
    # For implementation, this requires loading pre-computed lookup tables or a neural network
    # trained offline. Here we assume a placeholder function that returns adapted values.
    sigma1, o1 = split_adaptation_sigma_o(parent_gaussian.sigma, parent_gaussian.o)
    sigma2, o2 = split_adaptation_sigma_o(parent_gaussian.sigma, parent_gaussian.o)
    
    # Create child objects
    child1 = Gaussian(mu=mu1, sigma=sigma1, q=q1, o=o1, c=c1)
    child2 = Gaussian(mu=mu2, sigma=sigma2, q=q2, o=o2, c=c2)
    
    return [child1, child2]

def split_adaptation_sigma_o(sigma_m, o_m):
    """
    Placeholder for the offline-learned SplitAdaptation function.
    In practice, this maps parent scale and opacity to child scale and opacity
    to preserve density.
    """
    # Not given in the sources. Implementation requires the specific learned model
    # or lookup table from the BOGausS paper.
    # A simple heuristic might be:
    # sigma_c = sigma_m / sqrt(2)
    # o_c = o_m / 2
    # But the source emphasizes this is learned and accounts for opacity variations.
    raise NotImplementedError("SplitAdaptation function must be loaded from offline training data.")
```

## Subtopics

### Multi-View Density Control Criterion
A sibling method that uses multi-view consistency to control density, differing by focusing on view consistency rather than single-Gaussian density preservation during splitting. See [Multi-View Density Control Criterion](5663-multi-view-density-control-criterion.md).

## Operating characteristics
- **Time Complexity:** Splitting is $O(1)$ per Gaussian, but requires lookup or evaluation of the `SplitAdaptation` function.
- **Memory Complexity:** Increases by one Gaussian per split operation (parent removed, two children added).
- **Strengths:** Prevents loss perturbations and Gaussian clustering by maintaining density consistency [S1]. Discards cloning, simplifying the densification logic [S1].
- **Limitations:** Requires offline learning or pre-computation of the adaptation function [S1]. Does not address prioritization of which Gaussians to split (handled by distortion-based prioritization in BOGausS) [S1].
- **When to use:** When training 3DGS experiences instability due to splitting-induced loss perturbations or excessive clustering [S1].

## Background
Density-preserving Gaussian splitting was introduced in BOGausS (Better Optimized Gaussian Splatting) to address perturbations in the loss function caused by original splitting and cloning operations [S1]. It builds on insights from [5, 9] regarding opacity/size corrections and [16] regarding density preservation [S1]. It replaces the opacity reset mechanism of standard 3DGS with pruning based on opacity thresholds [S1]. Unlike [17], it explicitly accounts for opacity variations during adaptation [S1].

## Sources
[S1] BOGausS: Better Optimized Gaussian Splatting, https://doi.org/10.23919/eusipco63237.2025.11226626, fetched 2026-10-26
