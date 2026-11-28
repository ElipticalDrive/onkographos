# Blended Error Densification and Splitting

Also called: Blended Error Splitting

## Overview
Blended Error Densification and Splitting is a strategy for growing Gaussian primitives in 3D Gaussian Splatting that uses an auxiliary per-pixel error metric to drive densification decisions, rather than relying on positional gradient magnitudes [S1]. Blended errors are a per-primitive heuristic quantity where a higher error suggests that the primitive should be split [S1]. This method is distinct from sibling strategies by explicitly utilizing accumulated L1 error norms as probability weights for splitting [S1].

## How it works
**Inputs:**
- Current set of Gaussian primitives $\mathcal{G} = \{g_i\}$, where each primitive $g_i$ has mean $\mu_i$, covariance $\Sigma_i$ (defined by scales $\sigma_{i,1}, \sigma_{i,2}$ and rotation), opacity $\alpha_i$, and color/feature coefficients [G].
- Per-primitive blended L1 errors from the backward pass of the current training iteration [S1].
- Blended error accumulator $B_i$ for each primitive [S1].

**Outputs:**
- Updated set of primitives $\mathcal{G}'$, where a subset of primitives has been split into two [S1].

**Data Structures:**
- `blended_errors`: Array of floats, size $N$ (number of primitives). Stores the accumulated scalar error value for each primitive [S1].
- `primitives`: List of primitive objects containing fields: `mean` (vec2/vec3), `scale` (vec2), `rotation` (quat), `opacity` (float), `colors` (tensor), `gamma` (float), `axis_1` (unit vector along longest axis) [G].

**Steps:**

1. **Backward Pass Error Calculation:**
   During the backward pass of each training iteration, compute the blended L1 error for each primitive as defined by Rota Bulò et al. [S1]. Let $b_i$ be the blended error contribution for primitive $i$ in the current iteration.

2. **Accumulation:**
   Update the accumulated blended error for each primitive by adding the current iteration's contribution, rather than taking the maximum:
   $$ B_i^{(t)} = B_i^{(t-1)} + b_i^{(t)} $$
   where $t$ is the iteration index [S1].

3. **Normalization and Sampling:**
   Normalize the accumulated blended errors to form probability weights:
   $$ p_i = \frac{B_i}{\sum_j B_j} $$
   Use these weights to sample 5% of the current primitives for splitting [S1].

4. **Splitting Operation:**
   For each selected primitive $i$, assume $\sigma_{i,1} \ge \sigma_{i,2}$ [S1]. Create two new primitives with scales $(\frac{\sigma_{i,1}}{2}, \sigma_{i,2})$ and means $\mu_i - \frac{\sigma_{i,1}}{2} v_{i,1}$ and $\mu_i + \frac{\sigma_{i,1}}{2} v_{i,1}$, where $v_{i,1}$ is the unit vector along the principal axis corresponding to $\sigma_{i,1}$ [G]. Copy all other properties from the original primitive, including the $\gamma$ value [S1]. Remove the original primitive and insert the two new primitives.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blended_error_densification(primitives, blended_errors, iteration_errors):
    """
    primitives: List of primitive dicts/objects
    blended_errors: List[float], accumulated error per primitive
    iteration_errors: List[float], blended L1 error contribution per primitive for current step
    
    Returns: Updated list of primitives
    """
    # Step 2: Accumulate errors
    for i in range(len(primitives)):
        blended_errors[i] += iteration_errors[i]
        
    # Step 3: Normalize and sample
    total_error = sum(blended_errors)
    if total_error == 0:
        return primitives
        
    probabilities = [e / total_error for e in blended_errors]
    num_to_split = max(1, int(0.05 * len(primitives)))
    
    # Weighted sampling without replacement (standard implementation)
    indices_to_split = []
    available = list(range(len(primitives)))
    avail_probs = probabilities[:]
    
    for _ in range(num_to_split):
        total_p = sum(avail_probs)
        r = random.uniform(0, total_p)
        cumulative = 0.0
        chosen_idx = -1
        for j, p in enumerate(avail_probs):
            cumulative += p
            if r <= cumulative:
                chosen_idx = j
                break
        indices_to_split.append(available[chosen_idx])
        available.pop(chosen_idx)
        avail_probs.pop(chosen_idx)
        
    new_primitives = []
    for i, prim in enumerate(primitives):
        if i in indices_to_split:
            # Step 4: Split
            # Assume scales are stored as [sigma1, sigma2] with sigma1 >= sigma2
            sigma1 = prim.scale[0]
            sigma2 = prim.scale[1]
            v1 = prim.axis_1 # Unit vector along longest axis
            
            # New scales: halve the longest axis
            new_scale = [sigma1 / 2.0, sigma2]
            
            # New means
            offset = (sigma1 / 2.0) * v1
            mean_a = prim.mean - offset
            mean_b = prim.mean + offset
            
            # Create new primitives
            prim_a = {
                'mean': mean_a, 'scale': new_scale, 'rotation': prim['rotation'],
                'opacity': prim['opacity'], 'colors': prim['colors'], 'gamma': prim['gamma']
            }
            
            prim_b = {
                'mean': mean_b, 'scale': new_scale, 'rotation': prim['rotation'],
                'opacity': prim['opacity'], 'colors': prim['colors'], 'gamma': prim['gamma']
            }
            
            new_primitives.append(prim_a)
            new_primitives.append(prim_b)
        else:
            new_primitives.append(prim)
            
    return new_primitives
```

**Edge Cases and Pitfalls:**
- The method assumes $\sigma_{i,1} \ge \sigma_{i,2}$; axes must be sorted before splitting to ensure the correct axis is chosen [S1].
- When $\gamma_i \to \infty$, this split operation perfectly halves the quad along its long axis [S1].
- Precision issues may arise if $B_i$ values vary significantly in magnitude; normalization helps but underflow/overflow should be monitored [G].

## Subtopics

### [Error-based Densification](15319-error-based-densification.md)
Error-based densification is a strategy for growing Gaussian primitives in 3D Gaussian Splatting that uses an auxiliary per-pixel error metric (e.g., Structural Similarity or $L_1$) to drive densification decisions, rather than relying on positional gradient magnitudes.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths and Limitations:** Not given in the sources.

## Background
This method is described in the context of "Nexels: Neurally-Textured Surfels for Real-Time Novel View Synthesis with Sparse Primitives" [S1]. It builds upon densification strategies but introduces a specific error-based splitting heuristic inspired by 3DGS-MCMC [S1]. The error calculation references the work of Rota Bulò et al. [S1].

## Sources
[S1] Nexels: Neurally-Textured Surfels for Real-Time Novel View Synthesis with Sparse Primitives, https://doi.org/10.1007/978-3-032-37531-5_9, fetched 2026-10-28
