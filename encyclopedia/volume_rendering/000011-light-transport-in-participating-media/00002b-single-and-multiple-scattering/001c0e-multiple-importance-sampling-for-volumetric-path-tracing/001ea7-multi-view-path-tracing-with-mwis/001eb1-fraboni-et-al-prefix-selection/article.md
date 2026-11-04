# Fraboni et al. Prefix Selection

Also called: Fraboni prefix selection, Prefix Selection via Total Variation Distance, Stochastic acceptance/rejection of shifted prefixes, TV-based prefix selection

## Overview
Fraboni et al. Prefix Selection is a stochastic filtering mechanism for multi-view path tracing that rejects shifted path prefixes when their scattering distributions at the pivot point significantly mismatch the base prefix [S1]. It addresses the variance explosion caused by large Jacobian variations and scattering function differences when reusing paths across different camera views [S1]. Unlike methods that reuse all prefixes regardless of scattering properties, this method ensures that only prefixes with similar scattering behavior are included in the Multiple Importance Sampling (MIS) estimator, thereby reducing noise and preventing the bias artifacts associated with naive reuse [S1].

## How it works
The method operates during the prefix shifting phase of multi-view path tracing. It evaluates the similarity between the scattering distribution of a candidate shifted prefix and the base prefix using a discrete approximation of the Total Variation (TV) distance. If the distributions are sufficiently similar, the prefix is accepted; otherwise, it is rejected.

### Inputs and Outputs
*   **Inputs:**
    *   Base prefix $\bar{x}_i$ sampled for pixel $i$.
    *   Candidate shifted prefix $\bar{x}_j$ for camera $k$ (pixel $j$).
    *   Scattering distribution $p$ associated with the base prefix at the pivot interaction.
    *   Scattering distribution $q$ associated with the candidate prefix at the pivot interaction.
    *   Pivot interaction type (surface or medium).
*   **Outputs:**
    *   Boolean: Accept or Reject the candidate prefix $\bar{x}_j$.
    *   If accepted, a direction is sampled from the uniform mixture of accepted scattering distributions to continue the suffix.

### Data Structures
No specific complex data structures are required beyond standard path storage. The method requires storing the scattering function parameters (e.g., roughness, anisotropy, layer properties) at the pivot point for both the base and candidate prefixes to evaluate the PDFs $p(\omega)$ and $q(\omega)$.

### Steps
1.  **Select Evaluation Directions:** Determine a small set of directions $\{\omega_k\}_k$ to evaluate the scattering distributions. The number of directions is typically 2 [S1]. The selection depends on the pivot interaction type:
    *   **Surface Interaction:** Use the mirror directions of both viewing directions (base view direction reflected about the surface normal, and candidate view direction reflected about the surface normal) [S1].
    *   **Medium Interaction:** Use forward directions if the phase function is forward-scattering, and backward directions if it is backward-scattering [S1].
2.  **Evaluate PDFs:** Evaluate the probability density functions (PDFs) of the scattering distributions for the base prefix $p(\omega_k)$ and the candidate prefix $q(\omega_k)$ at each selected direction $\omega_k$ [S1].
3.  **Compute TV Distance:** Calculate the approximate Total Variation distance between the two discrete distributions using the formula below [S1].
4.  **Stochastic Acceptance:** Compute the acceptance probability $P_{i \rightarrow j}$ based on the TV distance. Generate a uniform random number $u \in [0, 1]$. If $u < P_{i \rightarrow j}$, accept the prefix; otherwise, reject it [S1].
5.  **Suffix Sampling (if accepted):** If the prefix is accepted, sample a direction $\omega$ from the uniform mixture of the scattering distributions of all accepted prefixes. Continue the path using regular path tracing for the remaining suffix [S1].

### Formula
The approximate Total Variation distance between two discrete distributions $p$ and $q$ evaluated on a set of directions $\{\omega_i\}_i$ is defined as [S1]:

$$
\text{TV}(p, q, \{\omega_i\}_i) = \frac{1}{2} \sum_i \left| \frac{p(\omega_i)}{\sum_k p(\omega_k)} - \frac{q(\omega_i)}{\sum_k q(\omega_k)} \right|
$$

Where:
*   $p(\omega_i)$ is the PDF value of the base scattering distribution at direction $\omega_i$.
*   $q(\omega_i)$ is the PDF value of the candidate scattering distribution at direction $\omega_i$.
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
*   **Performance:** The test can be performed before null interactions shifts to avoid unnecessary computations [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_tv_distance(p_pdf_values, q_pdf_values):
    """
    Computes the approximate Total Variation distance between two discrete 
    distributions represented by PDF values at specific directions.
    
    Args:
        p_pdf_values: List of float, PDF values for base distribution p.
        q_pdf_values: List of float, PDF values for candidate distribution q.
        
    Returns:
        float: The TV distance.
    """
    sum_p = sum(p_pdf_values)
    sum_q = sum(q_pdf_values)
    
    # Avoid division by zero if all PDFs are zero
    if sum_p == 0 or sum_q == 0:
        return 1.0  # Maximal distance if distributions are degenerate
        
    tv_sum = 0.0
    for p_val, q_val in zip(p_pdf_values, q_pdf_values):
        norm_p = p_val / sum_p
        norm_q = q_val / sum_q
        tv_sum += abs(norm_p - norm_q)
        
    return 0.5 * tv_sum

def select_prefix(base_prefix, candidate_prefix, pivot_interaction_type, phase_function_type):
    """
    Determines whether to accept a candidate shifted prefix based on 
    scattering distribution similarity using TV distance.
    
    Args:
        base_prefix: The base path prefix.
        candidate_prefix: The shifted path prefix.
        pivot_interaction_type: 'surface' or 'medium'.
        phase_function_type: 'forward_scattering' or 'backward_scattering' (for medium).
        
    Returns:
        bool: True if accepted, False if rejected.
    """
    # 1. Select evaluation directions
    # Note: In a real implementation, these directions are computed based on 
    # viewing vectors and surface normals/phase function properties.
    # Here we assume helper functions exist to get the specific directions.
    
    if pivot_interaction_type == 'surface':
        # Use mirror directions of both viewing directions
        omega_1 = mirror_direction(base_prefix.view_direction, base_prefix.normal)
        omega_2 = mirror_direction(candidate_prefix.view_direction, candidate_prefix.normal)
    elif pivot_interaction_type == 'medium':
        if phase_function_type == 'forward_scattering':
            omega_1 = base_prefix.view_direction
            omega_2 = candidate_prefix.view_direction
        else: # backward_scattering
            omega_1 = -base_prefix.view_direction
            omega_2 = -candidate_prefix.view_direction
    else:
        # Fallback or error handling
        return True 

    directions = [omega_1, omega_2]
    
    # 2. Evaluate PDFs for base and candidate at these directions
    # Assume scattering function evaluation is available
    p_pdf_values = []
    q_pdf_values = []
    
    for omega in directions:
        # Evaluate base scattering distribution p at omega
        p_val = evaluate_scattering_pdf(base_prefix.scattering_function, omega)
        p_pdf_values.append(p_val)
        
        # Evaluate candidate scattering distribution q at omega
        q_val = evaluate_scattering_pdf(candidate_prefix.scattering_function, omega)
        q_pdf_values.append(q_val)
        
    # 3. Compute TV Distance
    tv_dist = evaluate_tv_distance(p_pdf_values, q_pdf_values)
    
    # 4. Stochastic Acceptance
    # Acceptance probability P = 1 - TV
    acceptance_prob = 1.0 - tv_dist
    
    # Clamp probability to [0, 1] to handle numerical errors or edge cases
    acceptance_prob = max(0.0, min(1.0, acceptance_prob))
    
    # Generate random number for stochastic test
    u = random_uniform()
    
    return u < acceptance_prob
```

## Subtopics

### Multiple Importance Sampling without Selection (mismvpt)
A variant that uses MIS but reuses all prefixes regardless of scattering distribution similarity, leading to higher noise in anisotropic volumes [S1].

### Multi-View Path Tracing with MWIS
The parent method that incorporates this prefix selection to reduce noise and bias [S1].

## Operating characteristics
*   **Time Complexity:** The TV evaluation is inexpensive as it only requires evaluating the scattering PDF at a small number of directions (typically 2) [S1].
*   **Memory Complexity:** No additional memory overhead beyond storing the scattering function parameters at the pivot point.
*   **Performance:** Reduces variance in anisotropic volumes compared to naive reuse (mismvpt) [S1]. However, it still exhibits fireflies and visible bias compared to more advanced multi-view solutions [S1].
*   **Strengths:** Limits degradations due to large Jacobian variations; handles general distributions including phase functions; inexpensive to compute [S1].
*   **Limitations:** Exhibits fireflies and visible bias; TV approximation overestimates true distance; artifacts persist compared to state-of-the-art multi-view methods [S1].
*   **When to use:** When using multi-view path tracing with MIS and scattering distribution similarity is a concern, particularly for rough surfaces and standard phase functions [S1].
*   **When not to use:** When high-quality results without bias are required, as this method introduces visible bias [S1].

## Background
The method is based on the work of Fraboni et al. [FINB19], which is cited in the source [S1]. The specific paper is "Volumetric Multi‐View Rendering" by the authors of [S1], which references Fraboni et al. [FINB19] as the origin of the prefix selection technique [S1]. The technique was developed to address the challenges of reusing path prefixes in multi-view rendering, specifically the variance issues arising from Jacobian variations and scattering function mismatches [S1].

## Sources
[S1] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
