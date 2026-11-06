# Fraboni et al. Prefix Selection

Also called: Fraboni prefix selection, Prefix Selection via Total Variation Distance, Stochastic acceptance/rejection of shifted prefixes, TV-based prefix selection

## Overview
Fraboni et al. Prefix Selection is a stochastic filtering mechanism for multi-view path tracing that rejects shifted path prefixes when their scattering distributions at the pivot point significantly mismatch the base prefix [S1]. It addresses the variance explosion caused by large Jacobian variations and scattering function differences when reusing paths across different camera views [S1]. Unlike methods that reuse all prefixes regardless of scattering properties, this method ensures that only prefixes with similar scattering behavior are included in the Multiple Importance Sampling (MIS) estimator, thereby reducing noise and preventing the bias artifacts associated with naive reuse [S1].

## How it works
The method operates during the prefix shifting phase of multi-view path tracing [S1]. It evaluates the similarity between the scattering distribution of a candidate shifted prefix and the base prefix using a discrete approximation of the Total Variation (TV) distance [S1]. If the distributions are sufficiently similar, the prefix is accepted; otherwise, it is rejected [S1].

### Inputs and Outputs
*   **Inputs:** Base prefix $\bar{x}_i$ sampled for pixel $i$, candidate shifted prefix $\bar{x}_j$ for camera $k$ (pixel $j$), scattering distributions $p$ (base) and $q$ (candidate) at the pivot interaction, and the pivot interaction type (surface or medium) [S1].
*   **Outputs:** Boolean accept/reject decision for $\bar{x}_j$, and if accepted, a sampled direction from the uniform mixture of accepted scattering distributions to continue the suffix [S1].

### Data Structures
Not given in the sources.

### Steps
1.  **Select Evaluation Directions:** Determine a small set of directions $\{\omega_k\}_k$ to evaluate the scattering distributions [S1]. The number of directions is 2 [S1]. The selection depends on the pivot interaction type: for surface interactions, use the mirror directions of both viewing directions; for medium interactions, use forward directions if the phase function is forward-scattering, and backward directions if it is backward-scattering [S1].
2.  **Evaluate PDFs:** Evaluate the probability density functions (PDFs) of the scattering distributions for the base prefix $p(\omega_k)$ and the candidate prefix $q(\omega_k)$ at each selected direction $\omega_k$ [S1].
3.  **Compute TV Distance:** Normalize both discrete PDF sets and calculate the approximate TV distance using the provided formula [S1].
4.  **Stochastic Acceptance:** Compute the acceptance probability $P_{i \rightarrow j} = 1 - \text{TV}(p_i, p_j, \{\omega_k\}_k)$ [S1]. Generate a uniform random number $u \in [0, 1]$ [G]. If $u < P_{i \rightarrow j}$, accept the prefix; otherwise, reject it [S1].
5.  **Suffix Sampling:** If the prefix is accepted, sample a direction through the uniform mixture of scattering distributions associated with the accepted prefixes [S1]. Build the remaining part of the suffix by regular path tracing [S1].

### Formula
The approximate Total Variation distance between two discrete distributions $p$ and $q$ evaluated on a set of directions $\{\omega_i\}_i$ is defined as [S1]:

$$
\text{TV}(p, q, \{\omega_i\}_i) = \frac{1}{2} \sum_i \left| \frac{p(\omega_i)}{\sum_k p(\omega_k)} - \frac{q(\omega_i)}{\sum_k q(\omega_k)} \right|
$$

Where:
*   $p(\omega_i)$ is the PDF value of the base scattering distribution at direction $\omega_i$ [S1].
*   $q(\omega_i)$ is the PDF value of the candidate scattering distribution at direction $\omega_i$ [S1].
*   The denominators $\sum_k p(\omega_k)$ and $\sum_k q(\omega_k)$ normalize the discrete distributions over the sampled directions [S1].

The acceptance probability for a candidate prefix is [S1]:

$$
P_{i \rightarrow j} = 1 - \text{TV}(p_i, p_j, \{\omega_k\}_k)
$$

### Parameters
*   **Number of directions for TV evaluation:** 2 [S1].
*   **Direction selection heuristic:** Mirror directions for surfaces; forward/backward directions for media based on phase function anisotropy [S1].

### Edge Cases and Pitfalls
*   **TV Approximation Bias:** The discrete TV approximation overestimates the true TV distance [S1]. However, it provides correct estimates for rough single and bi-layered materials (e.g., rough plastics, conductors, dielectrics, coatings) and phase functions with varying anisotropy [S1].
*   **Fireflies and Bias:** The method exhibits fireflies and a visible bias that is not consistent as the number of samples increases [S1].
*   **Jacobian Variations:** While it limits degradations due to large Jacobian variations, it still shows artifacts compared to more advanced multi-view solutions [S1].
*   **Performance Optimization:** The test can be performed before null interactions shifts to avoid unnecessary computations [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def evaluate_tv_distance(p_vals, q_vals):
    """Computes the approximate TV distance between two discrete distributions."""
    sum_p = sum(p_vals)
    sum_q = sum(q_vals)
    if sum_p == 0.0 or sum_q == 0.0:
        return 1.0
    tv_sum = 0.0
    for p_val, q_val in zip(p_vals, q_vals):
        tv_sum += abs(p_val / sum_p - q_val / sum_q)
    return 0.5 * tv_sum

def reflect_direction(omega, normal):
    """Computes the mirror direction: omega - 2 * dot(omega, normal) * normal."""
    dot_val = omega[0]*normal[0] + omega[1]*normal[1] + omega[2]*normal[2]
    return (omega[0] - 2.0 * dot_val * normal[0],
            omega[1] - 2.0 * dot_val * normal[1],
            omega[2] - 2.0 * dot_val * normal[2])

def select_prefix(base_prefix, candidate_prefix, pivot_type, phase_type):
    """Determines whether to accept a candidate shifted prefix based on TV distance."""
    # 1. Select evaluation directions
    if pivot_type == 'surface':
        omega_1 = reflect_direction(base_prefix.view_dir, base_prefix.normal)
        omega_2 = reflect_direction(candidate_prefix.view_dir, candidate_prefix.normal)
    elif pivot_type == 'medium':
        if phase_type == 'forward':
            omega_1 = base_prefix.view_dir
            omega_2 = candidate_prefix.view_dir
        else:
            omega_1 = (-base_prefix.view_dir[0], -base_prefix.view_dir[1], -base_prefix.view_dir[2])
            omega_2 = (-candidate_prefix.view_dir[0], -candidate_prefix.view_dir[1], -candidate_prefix.view_dir[2])
    else:
        return True

    directions = [omega_1, omega_2]

    # 2. Evaluate PDFs
    # Note: The sources do not provide a closed-form PDF evaluator for arbitrary materials/phase functions.
    # This step assumes access to the scattering function's PDF evaluation routine.
    p_vals = [base_prefix.scattering.pdf(omega) for omega in directions]
    q_vals = [candidate_prefix.scattering.pdf(omega) for omega in directions]

    # 3. Compute TV Distance
    tv_dist = evaluate_tv_distance(p_vals, q_vals)

    # 4. Stochastic Acceptance
    acceptance_prob = max(0.0, min(1.0, 1.0 - tv_dist))
    u = random.random()
    return u < acceptance_prob
```

## Operating characteristics
*   **Time Complexity:** The TV evaluation is inexpensive as it only requires evaluating the scattering PDF at a small number of directions (typically 2) [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Reduces variance in anisotropic volumes compared to naive reuse [S1]. However, it still exhibits fireflies and visible bias compared to more advanced multi-view solutions [S1].
*   **Strengths:** Limits degradations due to large Jacobian variations; handles general distributions including phase functions; inexpensive to compute [S1].
*   **Limitations:** Exhibits fireflies and visible bias; TV approximation overestimates true distance; artifacts persist compared to state-of-the-art multi-view methods [S1].
*   **When to use:** When using multi-view path tracing with MIS and scattering distribution similarity is a concern, particularly for rough surfaces and standard phase functions [S1].
*   **When not to use:** When high-quality results without bias are required, as this method introduces visible bias [S1].

## Background
The method is based on the work of Fraboni et al. [FINB19], which is cited in the source [S1]. The technique was developed to address the challenges of reusing path prefixes in multi-view rendering, specifically the variance issues arising from Jacobian variations and scattering function mismatches [S1]. It is integrated into multi-view path tracing with MWIS to reduce noise and bias [S1].

## Sources
[S1] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
