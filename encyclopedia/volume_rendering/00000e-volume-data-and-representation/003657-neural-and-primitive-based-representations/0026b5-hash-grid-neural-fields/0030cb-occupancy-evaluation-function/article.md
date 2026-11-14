# Occupancy Evaluation Function

Also called: Occupancy Evaluation, Occupancy Query

## Overview
The Occupancy Evaluation Function computes the importance weight (occupancy) at a given position in the occupancy grid, which is used by the sampling estimator to determine where to allocate computational resources [S1, unofficial]. It maps a continuous coordinate vector to a scalar occupancy probability, serving as a binary or soft mask for the scene geometry. It differs from sibling methods like [HashGrid Encoding](3866-hashgrid-encoding.md) which typically output dense feature vectors for color/density decoding; instead, this function specifically outputs a scalar occupancy value derived from neural field outputs, often using a sigmoid transformation to create a smooth differentiable surface representation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A continuous coordinate vector $\mathbf{x}$ (typically 3D spatial position, potentially in AABB space or physical space).
- Output: A scalar occupancy value $occ \in [0, 1]$.

**Data Structures:**
The implementation relies on a neural field $F_\Theta$ (or `self.ne_field`) that maps physical coordinates to a log-occupancy value. Specific parameters include:
- `self.aabb_scale`: Scaling factor to map AABB coordinates to physical space.
- `self.occ_eval_fn_ne_mid`: The midpoint parameter for the sigmoid function.
- `self.occ_eval_fn_ne_width`: The width parameter for the sigmoid function.
- `self.clamp`: Boolean flag indicating whether to clamp intermediate log-occupancy values.
- `log_ne_min`, `log_ne_max`: Bounds for clamping log-occupancy values.

**Steps:**
1. **Coordinate Transformation:** If the input $\mathbf{x}_{AABB}$ is in axis-aligned bounding box (AABB) space, map it to physical coordinates $\mathbf{x}_{xyz}$ using the scale factor:
   $$ \mathbf{x}_{xyz} = \mathbf{x}_{AABB} \times \text{self.aabb\_scale} $$
   [S1, unofficial]

2. **Neural Field Query:** Query the neural field to obtain the log-occupancy value $\log_{10}ne$.
   - If ground truth neural fields are available (`self.use_gt_ne` is true and `self.ne_field` is not None), query directly:
     $$ \log_{10}ne = \text{self.ne\_field}(\mathbf{x}_{xyz}) $$
     [S1, unofficial]
   - Otherwise (during training with learned fields), call the prediction function:
     $$ \_, \log_{10}ne, \_, \_ = \text{self.\_predict\_learned\_fields}(\mathbf{x}_{AABB}) $$
     [S1, unofficial]

3. **Clamping (Optional):** If `self.clamp` is enabled, clamp $\log_{10}ne$ to the range $[\log\_ne\_min, \log\_ne\_max]$:
   $$ \log_{10}ne = \text{clamp}(\log_{10}ne, \log\_ne\_min, \log\_ne\_max) $$
   [S1, unofficial]

4. **Exponentiation:** Convert log-occupancy to linear scale (if not already linear in the field output, though the source implies `ne` is computed from `log10_ne`):
   $$ ne = 10.0^{\log_{10}ne} $$
   [S1, unofficial]

5. **Sigmoid Transformation:** Compute the final occupancy $occ$ using a sigmoid function centered at `self.occ_eval_fn_ne_mid` with width `self.occ_eval_fn_ne_width`:
   $$ occ = \sigma \left( \frac{\log_{10}ne - \text{self.occ\_eval\_fn\_ne\_mid}}{\text{self.occ\_eval\_fn\_ne\_width}} \right) $$
   where $\sigma(z) = \frac{1}{1 + e^{-z}}$.
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_occupancy(x_AABB, aabb_scale, ne_field, use_gt_ne, 
                       occ_eval_fn_ne_mid, occ_eval_fn_ne_width, 
                       clamp=False, log_ne_min=-10.0, log_ne_max=10.0):
    """
    Evaluate occupancy at a given AABB coordinate.
    
    Args:
        x_AABB: Input coordinate in AABB space (3D vector).
        aabb_scale: Scaling factor to physical space.
        ne_field: Neural field function returning log10 occupancy.
        use_gt_ne: Boolean, use ground truth neural field if true.
        occ_eval_fn_ne_mid: Midpoint for sigmoid.
        occ_eval_fn_ne_width: Width for sigmoid.
        clamp: Whether to clamp log10_ne.
        log_ne_min: Lower bound for clamping.
        log_ne_max: Upper bound for clamping.
        
    Returns:
        occ: Scalar occupancy value in [0, 1].
    """
    # Step 1: Coordinate Transformation
    x_xyz = x_AABB * aabb_scale
    
    # Step 2: Neural Field Query
    if use_gt_ne and ne_field is not None:
        log10_ne = ne_field(x_xyz)
    else:
        # Placeholder for learned field prediction logic
        # In practice, this calls self._predict_learned_fields(x_AABB)
        # which returns (_, log10_ne, _, _)
        # For this self-contained function, we assume a generic query
        # If ne_field is not provided and use_gt_ne is false, 
        # this step requires the specific learned field implementation.
        # [G] Assuming ne_field is used for both cases if available, 
        # or a separate learned field network. 
        # Based on [S1, unofficial], if not GT, it calls _predict_learned_fields.
        # Since we cannot implement the full network here, we assume
        # ne_field represents the queryable log10 occupancy source.
        log10_ne = ne_field(x_xyz) # [G] Simplified assumption for self-containment
        
    # Step 3: Clamping
    if clamp:
        log10_ne = max(log_ne_min, min(log10_ne, log_ne_max))
        
    # Step 4: Exponentiation (if ne_field returns log10)
    # Note: [S1, unofficial] computes ne = 10.0 ** log10_ne, but then uses log10_ne in sigmoid.
    # The sigmoid input is log10_ne, so ne is not strictly needed for the final occ.
    
    # Step 5: Sigmoid Transformation
    z = (log10_ne - occ_eval_fn_ne_mid) / occ_eval_fn_ne_width
    occ = 1.0 / (1.0 + math.exp(-z))
    
    return occ
```

**Parameters:**
- `occ_eval_fn_ne_mid`: Midpoint of the sigmoid. Typical value: Not given in the sources.
- `occ_eval_fn_ne_width`: Width of the sigmoid. Typical value: Not given in the sources.
- `clamp`: Boolean. Typical value: Not given in the sources.
- `log_ne_min`, `log_ne_max`: Bounds for clamping. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The use of `log10` and exponentiation may introduce floating-point precision issues if values are extreme. Clamping helps mitigate this [S1, unofficial].
- **Boundaries:** The function assumes the input $\mathbf{x}$ is within the valid AABB range. Out-of-bounds queries may return undefined values unless handled by the neural field [S1, unofficial].
- **Degenerate Input:** If `occ_eval_fn_ne_width` is zero, division by zero occurs. This should be guarded against [G].

## Subtopics

### Dichotomy-based Triggering for Occupancy Field
A triggering strategy for Occupancy Fields (OF) using Bolzano's theorem to find zero-crossings via bisection, as SGD is ineffective for relaxed binary OFs [S2]. It is more effective than SGD for OFs, especially when the initial point is far from the surface [S2].
- **Definition:** A method to find $x \in Z$ (zero-level set) in an implicit OF $F = f \circ T$ where $f$ is a sigmoid [S2].
- **Steps:**
  1. Randomly sample seed points in the implicit OF until a pair $\{x_0^+, x_0^-\}$ is found such that $F(x_0^+) > 0$ and $F(x_0^-) < 0$ [S2].
  2. Assert a zero-crossing exists on the segment between $x_0^+$ and $x_0^-$ based on Bolzano's theorem [S2].
  3. Repeatedly bisect the line segment. At iteration $t$, select the subinterval $[x_t^+, x_t^-]$ whose endpoints have opposite signs of occupancy [S2].
  4. Continue until $F(x_t^+) - F(x_t^-) \le \epsilon$ for a specified tolerance $\epsilon$ [S2].
- **Comparison:** More effective than SGD for OFs, especially when the initial point is far from the surface [S2].

## Operating characteristics
- **Time Complexity:** The basic occupancy evaluation is $O(1)$ per query, dominated by the neural field forward pass. The dichotomy-based triggering is $O(\log(1/\epsilon))$ iterations, with each iteration requiring a neural field query [S2].
- **Memory Complexity:** Depends on the neural field size. The occupancy evaluation function itself uses minimal memory [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the complexity of the neural field. The dichotomy method scales logarithmically with precision [S2].
- **Strengths:** Provides a differentiable occupancy mask for sampling; dichotomy method robustly finds surfaces without gradient issues [S1, unofficial][S2].
- **Limitations:** Requires a neural field; dichotomy method requires finding initial opposite-sign points [S1, unofficial][S2].
- **When to use:** When importance-weighted sampling is needed for volume rendering or ray marching [S1, unofficial]. Use dichotomy-based triggering when precise surface extraction is needed and SGD fails [S2].
- **When not:** When a simple binary grid is sufficient without neural field overhead [S1, unofficial].

## Background
The Occupancy Evaluation Function is a component of neural volume rendering pipelines that use hash grid neural fields. It was introduced in the context of CoroNeRF, which uses multi-resolution hash encoding for efficient neural rendering [S1, unofficial]. The dichotomy-based triggering method was proposed in "Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching" to address the inefficiency of SGD in finding zero-crossings in relaxed binary occupancy fields [S2].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching, https://doi.org/10.1109/tpami.2021.3135007, fetched 2026-10-26
