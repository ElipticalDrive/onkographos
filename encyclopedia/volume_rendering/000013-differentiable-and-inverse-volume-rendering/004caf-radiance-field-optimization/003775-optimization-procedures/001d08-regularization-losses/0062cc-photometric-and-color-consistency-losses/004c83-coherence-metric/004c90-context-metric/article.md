# Context Metric
Also called: Context-Aware Exploration Sequence, Data-domain context metric
## Overview
The Context Metric ensures that relationships between volume structures remain understandable by maintaining shared context ("old information") while revealing new information in generated Direct Volume Rendering Images (DVRIs) [S1]. It consists of two components: the data-domain context metric and the image-domain context metric [S1]. It is used within the Coherence Metric framework to prevent the generation of DVRIs that lack relational context with previously shown views [S1].
## How it works
**Inputs:**
- $c$: A candidate solution (e.g., transfer function parameters) generating a candidate DVRI $I_c$ [S1].
- $\Theta$: A set of previously shown DVRIs, including both previously generated DVRIs and original DVRIs [S1].
- $t_{data}$: Threshold value for the data-domain context metric [S1].
- $t_{image}$: Threshold value for the image-domain context metric [S1].
- $K_c$: A large constant used to penalize insufficient context [S1].

**Outputs:**
- $C(c)$: The final context metric value. Returns 0 if context is satisfactory; otherwise, returns a positive value indicating worse context [S1].

**Data Structures:**
- $\Theta$: A set of DVRIs. Each element $I_p \in \Theta$ is a rendered image [S1].
- $S_{new}, S_{old}$: Sets of atomic structures extracted from $I_c$ and $I_p$ respectively [S1].

**Steps:**
1. Initialize `any_satisfactory` to false and `best_delta_sum` to infinity [S1].
2. For each $I_p \in \Theta$, compute $C_{data}(I_c, I_p)$ and $C_{image}(I_c, I_p)$ [S1].
3. Check if $C_{data}(I_c, I_p) \geq t_{data}$ and $C_{image}(I_c, I_p) \geq t_{image}$; if true, set `any_satisfactory` to true [S1].
4. Compute temporary deltas for $I_p$: $\delta_{data}(I_p) = \max(0, 1 - C_{data}(I_c, I_p)/t_{data})$ and $\delta_{image}(I_p) = \max(0, 1 - C_{image}(I_c, I_p)/t_{image})$ [S1].
5. If $\delta_{data}(I_p) + \delta_{image}(I_p) < \text{best\_delta\_sum}$, update `best_delta_sum` and store $(\delta_{data}(I_p), \delta_{image}(I_p))$ as the best deltas [S1].
6. If `any_satisfactory` is true, return 0 [S1].
7. Otherwise, return $(1 + \text{best\_deltas}_{data} + \text{best\_deltas}_{image}) \cdot K_c$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def context_metric(I_c, Theta, t_data, t_image, K_c):
    any_satisfactory = False
    best_delta_sum = float('inf')
    best_deltas = (0.0, 0.0)
    
    for I_p in Theta:
        # Data-domain metric: percentage of structures in I_c also in I_p
        S_new = get_structures(I_c)
        S_old = get_structures(I_p)
        c_data = len(S_new & S_old) / len(S_new) if len(S_new) > 0 else 0.0
        
        # Image-domain metric: edge similarity between I_c and I_p
        c_image = compute_edge_similarity(I_c, I_p)
        
        if c_data >= t_data and c_image >= t_image:
            any_satisfactory = True
            
        d_data = 0.0 if c_data >= t_data else 1.0 - c_data / t_data
        d_image = 0.0 if c_image >= t_image else 1.0 - c_image / t_image
        delta_sum = d_data + d_image
        
        if delta_sum < best_delta_sum:
            best_delta_sum = delta_sum
            best_deltas = (d_data, d_image)
            
    if any_satisfactory:
        return 0.0
    return (1.0 + best_deltas[0] + best_deltas[1]) * K_c
```

**Formulas:**
The final context metric is defined as [S1]:
$$
C(c) = 
\begin{cases} 
0, & \text{if } \exists I_p \in \Theta \text{ such that } C_{data}(I_c, I_p) \geq t_{data} \text{ and } C_{image}(I_c, I_p) \geq t_{image} \\
(1 + (\delta_{data} + \delta_{image})) \cdot K_c, & \text{otherwise}
\end{cases}
$$
Where $\delta_{data}$ and $\delta_{image}$ are computed against $I_m \in \Theta$ (the DVRI minimizing $\delta_{data} + \delta_{image}$) [S1]:
$$
\delta_{data} = 
\begin{cases} 
0, & \text{if } C_{data}(I_c, I_m) \geq t_{data} \\
1 - \frac{C_{data}(I_c, I_m)}{t_{data}}, & \text{otherwise}
\end{cases}
\quad
\delta_{image} = 
\begin{cases} 
0, & \text{if } C_{image}(I_c, I_m) \geq t_{image} \\
1 - \frac{C_{image}(I_c, I_m)}{t_{image}}, & \text{otherwise}
\end{cases}
$$
The data-domain context metric measures the percentage of atomic structures previously presented in $I_{old}$ which remain in $I_{new}$ [S1]:
$$
C_{data}(I_{new}, I_{old}) = 
\begin{cases} 
\frac{|S_{new} \cap S_{old}|}{|S_{new}|}, & \text{if } |S_{new}| > 0 \\
0, & \text{otherwise}
\end{cases}
$$
The image-domain context metric estimates perceived contextual information using edge-based image similarity [S1].

**Parameters:**
- $t_{data}$: Threshold for data-domain context [S1].
- $t_{image}$: Threshold for image-domain context [S1].
- $K_c$: Large constant for penalty scaling [S1].

**Edge Cases and Pitfalls:**
- If a metric is below its threshold, the penalty is calculated as $(1 - \frac{Metric}{Threshold}) \cdot K_c$ [S1].
- If both metrics are satisfactory for any $I_p \in \Theta$, the result is 0 [S1].
- Division by zero is avoided by checking $|S_{new}| > 0$ before computing $C_{data}$ [S1].
- Edge detection for $C_{image}$ requires careful parameter tuning to avoid noise sensitivity [S1].

## Subtopics

### Final Context Metric
The final context metric $C(c)$ for a candidate solution $c$ generating DVRI $I_c$, given a set of previously shown DVRIs $\Theta$, returns 0 if context is good enough, otherwise a positive value indicating worse context [S1]. It uses thresholds $t_{data}$ and $t_{image}$ and a large constant $K_c$ [S1]. A pitfall is that if a metric is below its threshold, the penalty is calculated as $(1 - \frac{Metric}{Threshold}) \cdot K_c$ [S1].

## Operating characteristics
Not given in the sources.

## Background
The Context Metric was introduced in the paper "Visibility-Aware Direct Volume Rendering" [S1]. It is used in the context of automatic DVRI generation, often employing genetic algorithms for optimization [S1]. The metric addresses the problem where users may not be able to easily establish relations among structures revealed in different DVRIs if no common structure is presented [S1].

## Sources
[S1] Visibility-Aware Direct Volume Rendering, https://doi.org/10.1007/s11390-011-9428-3, fetched 2026-10-26
