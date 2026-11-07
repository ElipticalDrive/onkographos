# Style Aggregation

Also called: Aggregation, Sum of membership functions

## Overview
Style Aggregation is the process of combining multiple fuzzy logic rules to produce a single, unified transfer function for a specific rendering style. It solves the problem of conflicting or overlapping semantic interpretations by summing the truncated membership functions derived from individual rules. It differs from [Multi-Space Transfer Function Blending](5637-multi-space-transfer-function-blending-hardware-limitation-a.md) by operating on the logical aggregation of fuzzy sets rather than blending in a multi-dimensional parameter space.

## How it works
**Inputs:** A set of fuzzy rules $i \in I$, where each rule $i$ has an evaluated antecedent value $a_i \in [0, 1]$ and a consequent membership function $m_i(x)$ for a specific style.
**Outputs:** An aggregated membership function $f(x)$ for the style, which is subsequently defuzzified to a crisp value.

**Data Structures:**
*   **Lookup Tables:** Two pre-computed tables indexed by the antecedent value $a_i$.
    *   Table 1: Stores $\int x m_i'(x, a_i) dx$ (numerator summands).
    *   Table 2: Stores $\int m_i'(x, a_i) dx$ (denominator summands).
    *   *Note:* The sources indicate these are stored in lookup tables for GPU efficiency [S1]. The exact bit-packing or resolution of these tables is not specified [G: Typically implemented as 1D textures or arrays indexed by quantized $a_i$].

**Steps:**
1.  **Implication:** For each rule $i$, truncate the consequent membership function $m_i(x)$ using the antecedent value $a_i$.
    $$m_i'(x, a_i) = \min(a_i, m_i(x))$$
2.  **Aggregation:** Sum the truncated membership functions for all rules $i \in I$ affecting the current style to form the aggregated function $f(x)$.
    $$f(x) = \sum_{i \in I} m_i'(x, a_i)$$
3.  **Defuzzification (Centroid Method):** Calculate the centroid of $f(x)$ to derive a crisp rendering parameter $c_f$.
    $$c_f = \frac{\int x f(x) dx}{\int f(x) dx}$$
4.  **Optimization:** Substitute the aggregation formula into the centroid formula. Since the integrals depend only on $a_i$, pre-compute the summands.
    $$c_f = \frac{\sum_{i \in I} \int x m_i'(x, a_i) dx}{\sum_{i \in I} \int m_i'(x, a_i) dx}$$
5.  **Evaluation:** For each rendering attribute, perform $2n$ texture lookups (where $n$ is the number of rules affecting the attribute), $2(n-1)$ summations, and one division [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def style_aggregation(rules, style_id):
    """
    rules: List of Rule objects.
    Each rule has:
      - antecedent_value (float, 0.0 to 1.0)
      - membership_function (callable or lookup index)
      - affects_style (bool)
    
    style_id: The specific rendering style being aggregated.
    
    Returns:
      crisp_value (float): The centroid of the aggregated function.
    """
    # Filter rules affecting the specific style
    active_rules = [r for r in rules if r.affects_style(style_id)]
    
    numerator_sum = 0.0
    denominator_sum = 0.0
    
    for rule in active_rules:
        a_i = rule.antecedent_value
        
        # Retrieve pre-computed integral summands from lookup tables.
        # The lookup tables are indexed by a_i.
        # Note: In a real implementation, a_i might be quantized to an index.
        # [G] Assuming lookup tables are named num_table and den_table.
        num_term = lookup_table_numerator[a_i]
        den_term = lookup_table_denominator[a_i]
        
        numerator_sum += num_term
        denominator_sum += den_term
        
    # Avoid division by zero
    if denominator_sum == 0:
        return 0.0
        
    crisp_value = numerator_sum / denominator_sum
    return crisp_value
```

**Parameters:**
*   $n$: Number of rules affecting the attribute. Typical values depend on the complexity of the semantic model [S1].

**Edge Cases:**
*   **Zero Denominator:** If $\sum \int m_i'(x, a_i) dx = 0$, the centroid is undefined. The implementation should return a default value (e.g., 0.0) [G].
*   **Precision:** The use of lookup tables for integrals introduces discretization error. The resolution of the lookup tables affects accuracy [S1].

## Subtopics

### Defuzzification (Centroid Method)
Derives a crisp value for each style from the aggregated function using the centroid method. It pre-computes the summands of the numerator ($\int x m_i'(x, a_i) dx$) and denominator ($\int m_i'(x, a_i) dx$) and stores them in lookup tables indexed by $a_i$. For each rendering attribute, it evaluates using $2n$ texture lookups (for $n$ rules), $2(n-1)$ summations, and one division [S1]. The formula is $c_f = \frac{\sum_{i \in I} \int x m_i'(x, a_i) dx}{\sum_{i \in I} \int m_i'(x, a_i) dx}$ [S1].

### Implication
Applies the evaluated antecedent value to the consequent's membership function, truncating it. The formula is $m_i'(x, a_i) = \min(a_i, m_i(x))$, where $m_i(x)$ is the membership function for the semantic value affected by rule $i$, and $a_i$ is the antecedent value of rule $i$ [S1]. This results in a truncation of the membership function at the height of $a_i$ [S1].

## Operating characteristics
*   **Time Complexity:** $O(n)$ per rendering attribute, where $n$ is the number of rules affecting that attribute. The cost is dominated by $2n$ lookups and $2(n-1)$ additions [S1].
*   **Memory Complexity:** Requires storage for pre-computed lookup tables for numerator and denominator integrals. The size depends on the quantization of $a_i$ and the number of rules [S1].
*   **Strengths:** Efficiently implementable on GPU using texture lookups; separates pre-computation (offline) from evaluation (runtime) [S1].
*   **Limitations:** Requires pre-computation of integrals for all possible antecedent values; accuracy depends on lookup table resolution [S1].
*   **When to use:** When interactive, semantics-driven volume rendering is required and rules can be pre-processed [S1].

## Background
Originated in the work of Rautek, Bruckner, and Gröller (2008) for interaction-dependent semantics in illustrative volume rendering [S1]. It is based on fuzzy logic operations where antecedents are combined using fuzzy AND (minimum) and OR (maximum) [S1]. The method addresses the difficulty of implementing fuzzy logic operations (implication, aggregation, defuzzification) directly on GPUs by using pre-computed lookup tables for the centroid calculation [S1].

## Sources
[S1] Interaction‐Dependent Semantics for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2008.01216.x, fetched 2026-11-03
