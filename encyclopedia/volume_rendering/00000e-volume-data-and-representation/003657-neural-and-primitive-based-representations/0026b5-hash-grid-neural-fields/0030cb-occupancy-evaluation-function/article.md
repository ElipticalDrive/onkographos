# Occupancy Evaluation Function

Also called: Occupancy Evaluation, Occupancy Query

## Overview
The Occupancy Evaluation Function computes the importance weight (occupancy) at a given position in the occupancy grid, which is used by the sampling estimator to determine where to place samples [S1, unofficial]. It transforms a raw neural field output (typically log-density) into a bounded occupancy probability. It differs from sibling methods like [HashGrid Encoding](3866-hashgrid-encoding.md) by focusing specifically on the final scalar conversion step for sampling importance, rather than the feature extraction or hashing mechanism itself.

## How it works
**Inputs and Outputs:**
*   Input: A coordinate vector $\mathbf{x}$ (typically in AABB space or physical space depending on the branch), and model parameters (clamping bounds, midpoints, widths).
*   Output: A scalar occupancy value $occ \in [0, 1]$.

**Data Structures:**
*   `ne_field`: A neural network or lookup structure that returns the log10 of the density/occupancy value.
*   `aabb_scale`: A scaling factor to map AABB coordinates to physical coordinates.
*   `clamp`: A boolean flag indicating whether to clamp the intermediate log10 value.
*   `log_ne_min`, `log_ne_max`: The lower and upper bounds for clamping the log10 value.
*   `occ_eval_fn_ne_mid`: The midpoint parameter for the sigmoid function.
*   `occ_eval_fn_ne_width`: The width parameter for the sigmoid function.

**Steps:**
The evaluation follows one of two paths depending on whether ground truth neural fields are available and used.

1.  **Ground Truth Path (Inference/Validation):**
    If `self.use_gt_ne` is true and `self.ne_field` is not None:
    a. Map the input AABB coordinate $\mathbf{x}_{AABB}$ to physical coordinates:
       $$ \mathbf{x}_{xyz} = \mathbf{x}_{AABB} \times \text{aabb\_scale} $$
    b. Query the neural field to get the log10 density:
       $$ \text{log10\_ne} = \text{ne\_field}(\mathbf{x}_{xyz}) $$
    c. If `self.clamp` is true, clamp the value:
       $$ \text{log10\_ne} = \text{clamp}(\text{log10\_ne}, \text{log\_ne\_min}, \text{log\_ne\_max}) $$
    d. Convert back to linear scale:
       $$ ne = 10.0^{\text{log10\_ne}} $$
    e. Compute occupancy using the sigmoid function:
       $$ occ = \sigma\left(\frac{\text{log10\_ne} - \text{occ\_eval\_fn\_ne\_mid}}{\text{occ\_eval\_fn\_ne\_width}}\right) $$
       where $\sigma(z) = \frac{1}{1 + e^{-z}}$.

2.  **Training Path:**
    Else (for training):
    a. Call the learned fields predictor:
       $$ (\_, \text{log10\_ne}, \_, \_) = \text{self.\_predict\_learned\_fields}(\mathbf{x}_{AABB}) $$
    b. The `log10_ne` is already clamped if `self.clamp` is true.
    c. Compute occupancy using the sigmoid function:
       $$ occ = \sigma\left(\frac{\text{log10\_ne} - \text{occ\_eval\_fn\_ne\_mid}}{\text{occ\_eval\_fn\_ne\_width}}\right) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_occupancy(x_AABB, use_gt_ne, ne_field, aabb_scale, clamp, 
                       log_ne_min, log_ne_max, occ_eval_fn_ne_mid, 
                       occ_eval_fn_ne_width, predict_learned_fields):
    """
    Evaluates the occupancy at a given AABB coordinate.
    """
    if use_gt_ne and ne_field is not None:
        # Map to physical coords
        x_xyz = x_AABB * aabb_scale
        
        # Query neural field
        log10_ne = ne_field(x_xyz)
        
        # Clamp if necessary
        if clamp:
            log10_ne = max(log_ne_min, min(log10_ne, log_ne_max))
            
        # Compute occupancy from log10_ne directly via sigmoid
        # Note: The source computes 'ne' but then uses 'log10_ne' in the sigmoid formula.
        # The formula provided is: occ = sigmoid((log10_ne - mid) / width)
        # This implies the sigmoid operates on the log10 scale, not the linear 'ne' scale.
        exponent = (log10_ne - occ_eval_fn_ne_mid) / occ_eval_fn_ne_width
        occ = 1.0 / (1.0 + math.exp(-exponent))
        return occ
    else:
        # Training path
        _, log10_ne, _, _ = predict_learned_fields(x_AABB)
        
        # log10_ne is already clamped if self.clamp is true
        
        exponent = (log10_ne - occ_eval_fn_ne_mid) / occ_eval_fn_ne_width
        occ = 1.0 / (1.0 + math.exp(-exponent))
        return occ
```

**Parameters:**
*   `occ_eval_fn_ne_mid`: Midpoint of the sigmoid. Typical value: Not given in the sources.
*   `occ_eval_fn_ne_width`: Width of the sigmoid. Typical value: Not given in the sources.
*   `log_ne_min`, `log_ne_max`: Clamping bounds. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The conversion $10.0^{\text{log10\_ne}}$ is computed in the Ground Truth path but not used in the final sigmoid calculation in the source code snippet, which uses `log10_ne` directly. This suggests the sigmoid is applied to the log-density, not the density itself. Implementers must ensure consistency with the source's specific formula [S1, unofficial].
*   **Clamping:** If `clamp` is enabled, values outside `[log_ne_min, log_ne_max]` are truncated. This prevents extreme values from dominating the sigmoid output.

## Subtopics

### Dichotomy-based Triggering for Occupancy Field
A triggering strategy for Occupancy Fields (OF) using Bolzano's theorem to find zero-crossings via bisection, as SGD is ineffective for relaxed binary OFs [S2]. It is more effective than SGD for OFs, especially when the initial point is far from the surface [S2].
1.  Randomly sample seed points in the implicit OF until a pair $\{x_0^+, x_0^-\}$ is found such that $F(x_0^+) > 0$ and $F(x_0^-) < 0$ [S2].
2.  Assert a zero-crossing exists on the segment between $x_0^+$ and $x_0^-$ based on Bolzano's theorem [S2].
3.  Repeatedly bisect the line segment. At iteration $t$, select the subinterval $[x_t^+, x_t^-]$ whose endpoints have opposite signs of occupancy [S2].
4.  Continue until $F(x_t^+) - F(x_t^-) \le \epsilon$ for a specified tolerance $\epsilon$ [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, dominated by the neural field evaluation (if used) and the sigmoid computation.
*   **Memory Complexity:** $O(1)$ auxiliary memory.
*   **Strengths:** Provides a differentiable occupancy value suitable for gradient-based optimization and importance sampling.
*   **Limitations:** Relies on the quality of the underlying `ne_field` or learned fields. The dichotomy variant requires finding initial seed points with opposite signs, which may be inefficient in sparse regions.

## Background
The Occupancy Evaluation Function is a component of neural radiance field and implicit surface representations, specifically those using hash grid encodings. It bridges the gap between raw neural field outputs (density/log-density) and the probabilistic occupancy values required for efficient sampling strategies. The dichotomy-based triggering method was introduced to address the inefficiency of SGD in finding zero-crossings for relaxed binary occupancy fields [S2].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching, https://doi.org/10.1109/tpami.2021.3135007, fetched 2026-10-26
