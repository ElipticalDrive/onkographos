# Point-wise Inverse Discrete Wavelet Transform (PW-IDWT) via Generalized Projection

Also called: GP of sample, Generalized Projection (GP), PW-IDWT, Point-wise Inverse Discrete Wavelet Transform

## Overview
The Point-wise Inverse Discrete Wavelet Transform (PW-IDWT) via Generalized Projection (GP) reconstructs a single signal sample (or a set of samples) from a wavelet decomposition by determining the minimal set of subband coefficients required [S1]. It solves the problem of identifying which coefficients at a coarser level contribute to a specific sample at a finer level during synthesis, exploiting the recursive nature of the lifting scheme [S1]. Unlike methods that decode entire subbands, this approach calculates the exact Generalized Projection of a sample into the transformed domain, minimizing the number of subband images to decode [S1]. It builds on [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) by providing an efficient, sample-specific inverse transform mechanism [G].

## How it works
### Inputs and Outputs
**Inputs:**
*   Wavelet decomposition levels $L$ (where $L=0$ is the finest and $L$ is the coarsest) [S1].
*   Target sample position $k$ in the finest resolution signal domain (level $0$) [S1].
*   Filter type parameters: causal length $l_s$ (or $t_l$), anti-causal length $l_s^+$ (or $l_t^+$), and step type $s$ (update) or $t$ (prediction) [S1].
*   Number of lifting steps $m$ per level transition [S1].

**Outputs:**
*   The set of coefficient indices required for reconstruction, defined by boundaries $b_j^{a,-}, b_j^{a,+}, b_j^{d,-}, b_j^{d,+}$ for each subband $j$ (approximation $a$ or detail $d$) at each level $l$ [S1].

### Data Structures
The method tracks boundaries for the Generalized Projection (GP) at each lifting step [S1].
*   $\hat{b}_a^{i,-}, \hat{b}_a^{i,+}$: Lower and upper boundary indices for the approximation subband at lifting step $i$ [S1].
*   $\hat{b}_d^{i,-}, \hat{b}_d^{i,+}$: Lower and upper boundary indices for the detail subband at lifting step $i$ [S1].
*   $b_j^{a,-}, b_j^{a,+}, b_j^{d,-}, b_j^{d,+}$: Final boundaries for subband $j$ after $m$ lifting steps [S1].

### Numbered Steps
1.  **Initialize**: For the target sample position $k$ at the finest level, determine if $k$ is even or odd ($a=0$ or $a=1$) to select the correct set of lifting steps for synthesis [S1]. Initialize the Generalized Projection (GP) boundaries for the coarsest level $L$ as $[k \gg L, k \gg L]$ for the approximation subband [G].
2.  **Iterate Coarsest to Finest**: Iterate for $l = L-1$ down to $0$ [S1]. At each finer resolution, there is a set of samples to be recovered $\text{GP}(l, j)_k$ instead of a single one [S1].
3.  **Process Lifting Steps**: For each lifting step $i$ from $0$ to $m-1$ in the synthesis chain, update the boundary indices $\hat{b}_a$ and $\hat{b}_d$ using min/max updating rules based on the filter type (prediction or update) and filter lengths [S1].
    *   If the step is an **Update step** (type $s$):
        *   $\hat{b}_a^{i+1,-} = \hat{b}_a^{i,-}$ [S1]
        *   $\hat{b}_a^{i+1,+} = \hat{b}_a^{i,+}$ [S1]
        *   $\hat{b}_d^{i+1,-} = \min\{\hat{b}_d^{i,-}, \hat{b}_a^{i,-} + l_s\}$ [S1]
        *   $\hat{b}_d^{i+1,+} = \max\{\hat{b}_d^{i,+}, \hat{b}_d^{i,+} + l_s^+\}$ [S1]
    *   If the step is a **Prediction step** (type $t$):
        *   $\hat{b}_a^{i+1,-} = \min\{\hat{b}_a^{i,-}, \hat{b}_d^{i,-} + t_l\}$ [S1]
        *   $\hat{b}_a^{i+1,+} = \max\{\hat{b}_a^{i,+}, \hat{b}_d^{i,+} + l_t^+\}$ [S1]
        *   $\hat{b}_d^{i+1,-} = \hat{b}_d^{i,-}$ [S1]
        *   $\hat{b}_d^{i+1,+} = \hat{b}_d^{i,+}$ [S1]
4.  **Finalize Boundaries**: After completing all $m$ lifting steps for a level transition, assign the final boundaries $b_j^{a,-}, b_j^{a,+}, b_j^{d,-}, b_j^{d,+}$ based on the final values of $\hat{b}_j^{m}$ and $\hat{b}_j^{m+1}$ [S1]:
    *   $b_j^{a,-} = \hat{b}_j^{m,-}$ [S1]
    *   $b_j^{a,+} = \hat{b}_j^{m,+}$ [S1]
    *   $b_j^{d,-} = \hat{b}_j^{m+1,-}$ [S1]
    *   $b_j^{d,+} = \hat{b}_j^{m+1,+}$ [S1]
5.  **Set of Samples**: To find the GP for a set of samples, recursively compute the GP for individual samples in the approximation subband and take the union of their resulting coefficient sets [S1]:
    $$ \text{GP}^\mu(l+1, j) = \cup_k \text{GP}^\mu(l+1, j)_k \quad \forall k \in \text{GP}(l, j)^a $$
    where $\text{GP}(l, j)^a$ is the set of samples in the approximation subband at level $l$ [S1].

### Formulas
**IDWT Reconstruction:**
The approximation subband at finer level $l-1$ is reconstructed from coarser level $l$ coefficients:
$$ a(l-1, k) = \sum_{n=-\infty}^{\infty} h(k-2n)a(l, n) + \sum_{n=-\infty}^{\infty} g(k-2n)d(l, n) $$
where $a$ and $d$ are approximation and detail subbands, and $h, g$ are synthesis filters [S1].

**Total Subbands for 3D:**
For 3D reconstruction (filtering on x, y, z, assuming 2-D images stacked along z), the total number of subband images $N_k$ needed for reconstruction of image $k$ is:
$$ N_k = 4 \times [\text{GP}(L, a)_k + \text{GP}(L, d)_k] + \sum_{l=L-1}^1 [3 \times \text{GP}(l, a)_k + 4 \times \text{GP}(l, d)_k] $$
[S1]

**Boundary Updating Rules:**
*   Update Step ($s$):
    $$ \hat{b}_d^{i+1,-} = \min\{\hat{b}_d^{i,-}, \hat{b}_a^{i,-} + l_s\} $$
    $$ \hat{b}_d^{i+1,+} = \max\{\hat{b}_d^{i,+}, \hat{b}_d^{i,+} + l_s^+\} $$
    [S1]
*   Prediction Step ($t$):
    $$ \hat{b}_a^{i+1,-} = \min\{\hat{b}_a^{i,-}, \hat{b}_d^{i,-} + t_l\} $$
    $$ \hat{b}_a^{i+1,+} = \max\{\hat{b}_a^{i,+}, \hat{b}_d^{i,+} + l_t^+\} $$
    [S1]

### Parameters
*   **Filter Choice**: The 5-3 filter (interpolating, two-step chain) is recommended to minimize the number of subband images to decode [S1].
*   **Parity Index $a$**: Distinguishes between even ($a=0$) and odd ($a=1$) positions, as the set of lifting steps differs [S1].
*   **Filter Lengths**: $l_s, l_s^+$ for update steps; $t_l, l_t^+$ for prediction steps [S1].

### Edge Cases and Pitfalls
*   **Integer DWT Rounding Noise**: Integer DWT introduces rounding noise proportional to the number of rounding operations (decomposition depth and lifting chain length), which degrades rate/distortion performance and causes PSNR oscillation along the axis [S1].
*   **Boundary Conditions**: Special handling is required for signal boundaries, including a noted sign inversion $b_d = -b_d$ for detail subband boundaries during prediction steps [S1].
*   **Periodicity**: The number of samples required in each subband is a periodic function of the sample position $k$ with period 2 [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gp_boundaries(sample_k, decomposition_level_L, filter_params):
    """
    Computes the boundaries of the Generalized Projection (GP) for a single sample.
    
    Args:
        sample_k: int, position of the sample in the finest resolution signal.
        decomposition_level_L: int, total number of decomposition levels (coarsest level).
        filter_params: dict, containing 'm' (number of lifting steps), 
                       'steps' (list of step types: 's' for update, 't' for prediction),
                       'l_s', 'l_s_plus', 't_l', 't_l_plus'.
    
    Returns:
        dict: Boundaries for each level l from L down to 0.
              Keys are level indices. Values are dicts with 'a_minus', 'a_plus', 'd_minus', 'd_plus'.
    """
    # Initialize boundaries at the coarsest level L
    # The GP starts as a single point at the coarsest level for the approximation subband
    k_coarse = sample_k >> decomposition_level_L
    current_a_minus = k_coarse
    current_a_plus = k_coarse
    current_d_minus = k_coarse
    current_d_plus = k_coarse
    
    gp_boundaries = {}
    gp_boundaries[decomposition_level_L] = {
        'a_minus': current_a_minus, 'a_plus': current_a_plus,
        'd_minus': current_d_minus, 'd_plus': current_d_plus
    }
    
    steps = filter_params['steps']
    m = filter_params['m']
    l_s = filter_params['l_s']
    l_s_plus = filter_params['l_s_plus']
    t_l = filter_params['t_l']
    t_l_plus = filter_params['t_l_plus']
    
    # Iterate from coarsest level L-1 down to 0
    for l in range(decomposition_level_L - 1, -1, -1):
        # Initialize step boundaries for this level transition
        step_a_minus = current_a_minus
        step_a_plus = current_a_plus
        step_d_minus = current_d_minus
        step_d_plus = current_d_plus
        
        # Process each lifting step
        for i in range(m):
            step_type = steps[i]
            
            if step_type == 's': # Update step
                # Update approximation boundaries (unchanged)
                new_a_minus = step_a_minus
                new_a_plus = step_a_plus
                
                # Update detail boundaries
                new_d_minus = min(step_d_minus, step_a_minus + l_s)
                new_d_plus = max(step_d_plus, step_d_plus + l_s_plus)
                
                step_a_minus = new_a_minus
                step_a_plus = new_a_plus
                step_d_minus = new_d_minus
                step_d_plus = new_d_plus
                
            elif step_type == 't': # Prediction step
                # Update detail boundaries (unchanged)
                new_d_minus = step_d_minus
                new_d_plus = step_d_plus
                
                # Update approximation boundaries
                new_a_minus = min(step_a_minus, step_d_minus + t_l)
                new_a_plus = max(step_a_plus, step_d_plus + t_l_plus)
                
                step_a_minus = new_a_minus
                step_a_plus = new_a_plus
                step_d_minus = new_d_minus
                step_d_plus = new_d_plus
        
        # After all lifting steps for this level, assign final boundaries
        gp_boundaries[l] = {
            'a_minus': step_a_minus,
            'a_plus': step_a_plus,
            'd_minus': step_d_minus,
            'd_plus': step_d_plus
        }
        
        # Update current boundaries for the next iteration (finer level)
        current_a_minus = step_a_minus
        current_a_plus = step_a_plus
        current_d_minus = step_d_minus
        current_d_plus = step_d_plus
        
    return gp_boundaries

def compute_gp_for_set(samples, decomposition_level_L, filter_params):
    """
    Computes the GP for a set of samples by taking the union of individual GPs.
    """
    all_boundaries = {}
    for k in samples:
        boundaries = compute_gp_boundaries(k, decomposition_level_L, filter_params)
        for l, b in boundaries.items():
            if l not in all_boundaries:
                all_boundaries[l] = {'a_minus': float('inf'), 'a_plus': float('-inf'), 
                                     'd_minus': float('inf'), 'd_plus': float('-inf')}
            all_boundaries[l]['a_minus'] = min(all_boundaries[l]['a_minus'], b['a_minus'])
            all_boundaries[l]['a_plus'] = max(all_boundaries[l]['a_plus'], b['a_plus'])
            all_boundaries[l]['d_minus'] = min(all_boundaries[l]['d_minus'], b['d_minus'])
            all_boundaries[l]['d_plus'] = max(all_boundaries[l]['d_plus'], b['d_plus'])
    return all_boundaries
```

## Operating characteristics
*   **Time Complexity**: The method is efficient for sparse reconstruction as it only computes the necessary coefficients, with complexity proportional to the number of lifting steps and the depth of decomposition, which is minimal for short filters like 5-3 [S1].
*   **Memory Complexity**: Low, as it does not require storing or decoding entire subbands, only the specific coefficients identified by the GP [S1].
*   **Performance**: The 5-3 filter requires significantly fewer coefficients than the 9/7 filter. For L=3, the 5-3 filter requires 9-15 samples per 2D slice, while 9/7 requires 19-37. This reduces decoding time and power consumption [S1].
*   **Strengths**: Minimizes the number of subband images to decode, reducing computational load and power consumption. Ideal for applications where only specific samples or regions of interest need to be reconstructed [S1].
*   **Limitations**: Integer DWT introduces rounding noise proportional to the number of rounding operations, which can degrade quality, especially for deep decompositions [S1]. The method is sensitive to the position of the sample due to periodicity and boundary effects [S1].

## Background
The Point-wise Inverse Discrete Wavelet Transform (PW-IDWT) via Generalized Projection (GP) was introduced in the context of three-dimensional encoding/two-dimensional decoding of medical data [S1]. The method exploits the recursive nature of the wavelet transform and the separability of the transform to map the problem to a one-dimensional case [S1]. It is particularly useful for applications where computational resources are limited, such as mobile or embedded systems, due to its efficiency in decoding only the necessary coefficients [S1]. The choice of the 5-3 filter is motivated by its short length and two-step chain, which minimizes the number of subband images to decode [S1].

## Sources
[S1] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
