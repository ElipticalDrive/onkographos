# Delta Tracking with Volume State Changes

Also called: Delta tracking with density updates, Sample reuse in volumes

## Overview
Delta tracking with volume state changes is a Monte Carlo technique for rendering participating media where the volume density changes between optimization iterations or time steps, such as in inverse rendering or animation. It solves the problem of maintaining unbiased sampling when reusing path vertices across frames or iterations by ensuring the majorant extinction coefficient bounds the density of both the current and previous volume states. It differs from standard delta tracking by explicitly handling temporal or iterative correlations, which increases null collision overhead but leverages the fact that density changes are typically small [S1].

## How it works
**Inputs and Outputs:**
- Input: A sequence of volume density fields $\rho_t(\mathbf{x})$ for time/iteration $t$ and $t+1$.
- Input: A set of path vertices $\{\mathbf{x}_i\}$ sampled from the previous state $t$.
- Output: Unbiased radiance estimates for the current state $t+1$ using the existing path structure.

**Data Structures:**
- `density_field`: A function or grid mapping $\mathbf{x} \to \rho(\mathbf{x})$.
- `majorant`: A scalar $M$ such that $M \ge \sigma_t(\mathbf{x})$ for all $\mathbf{x}$ in the domain.

**Method Steps:**
1. **Determine the Combined Majorant:**
   When reusing a path vertex $\mathbf{x}$, the extinction coefficient must be bounded for both the old state $\sigma_t(\mathbf{x})$ and the new state $\sigma_{t+1}(\mathbf{x})$. The majorant $M$ must satisfy:
   $$ M \ge \max(\sigma_t(\mathbf{x}), \sigma_{t+1}(\mathbf{x})) $$
   In practice, $M$ is often precomputed as the global maximum of the union of densities or updated incrementally if the change is small [S1].

2. **Sample Free Path:**
   For a ray segment starting at $\mathbf{x}$, sample the distance to the next collision $d$ using the combined majorant:
   $$ d = -\frac{\ln(u)}{M} $$
   where $u \sim \text{Uniform}(0, 1)$.

3. **Check for Collision:**
   Evaluate the density at the proposed collision point $\mathbf{x}' = \mathbf{x} + d \cdot \omega$.
   - If $\mathbf{x}'$ is outside the volume, terminate the path.
   - Otherwise, determine if a real collision occurs. The probability of a real collision is:
     $$ p_{\text{real}} = \frac{\sigma_{t+1}(\mathbf{x}')}{M} $$
     Generate $v \sim \text{Uniform}(0, 1)$. If $v < p_{\text{real}}$, accept the collision as a real scattering event. If $v \ge p_{\text{real}}$, it is a null collision (virtual interaction).

4. **Handle State Transition:**
   - If a real collision occurs, update the path vertex to $\mathbf{x}'$ and continue tracking.
   - If a null collision occurs, continue tracking from $\mathbf{x}'$ with the same direction, accumulating zero radiance contribution for this step.
   - Note: The majorant $M$ must remain valid for both states throughout the path. If the path traverses regions where the density change is significant, $M$ must be large enough to bound the maximum possible density in either state [S1].

**Parameters:**
- $M$: Majorant extinction coefficient. Typical value: $\max_{\mathbf{x}} (\max(\sigma_t(\mathbf{x}), \sigma_{t+1}(\mathbf{x})))$ [S1].

**Edge Cases and Pitfalls:**
- **Increased Null Collisions:** Bounding the majorant for both states increases the number of null collisions compared to tracking each state independently. However, this overhead is limited because density changes between iterations are typically small [S1].
- **Precision:** Ensure $M$ is computed accurately to avoid bias. If $M < \sigma(\mathbf{x})$ at any point, the sampling distribution is invalid.
- **Degenerate Input:** If the volume density becomes zero in one state but not the other, the majorant must still bound the non-zero state.

## Subtopics

### [Heterogeneous Volumetric Raymarching](11138-heterogeneous-volumetric-raymarching.md)
Uses delta tracking on density fields for non-uniform media. Chooses this when handling complex, non-analytic density distributions.

## Operating characteristics
- **Time Complexity:** $O(N \cdot \frac{M}{\bar{\sigma}})$, where $N$ is the path length and $\bar{\sigma}$ is the average extinction. The overhead increases with the ratio of the combined majorant to the actual extinction.
- **Memory Complexity:** $O(1)$ per path vertex, plus storage for the density fields.
- **Performance:** The method introduces additional null collisions due to the conservative majorant. However, since density changes between optimization steps are small, the performance penalty is often acceptable [S1].
- **Strengths:** Enables efficient reuse of path vertices in inverse rendering and animation, reducing variance from path initialization.
- **Limitations:** Increased null collision rate compared to independent tracking. Not suitable if density changes are large or discontinuous between states.
- **When to Use:** Inverse rendering, optimization trajectories, or animation where path vertices can be reused across frames/iterations with small density changes [S1].
- **When Not to Use:** When density changes are large, or when independent sampling is preferred to avoid correlation artifacts.

## Background
The technique arises in the context of recursive control variates for inverse rendering, where sample reuse is critical for variance reduction. It was identified that delta tracking requires the majorant to bound both old and new volumes, adding overhead [S1]. The method is analogous to rendering spectrally varying volumes, suggesting potential benefits from techniques designed for such cases [S1].

## Sources
[S1] Recursive Control Variates for Inverse Rendering, https://research.nvidia.com/labs/rtr/publication/nicolet2023recursive/, fetched 2026-11-08
