# Context Metric
Also called: Data-domain context metric
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

**Steps:**
1. Initialize $\delta_{data} = 0$ and $\delta_{image} = 0$ [S1].
2. Identify $I_m \in \Theta$ as the DVRI with the minimum value of $(\delta_{data} + \delta_{image})$ relative to $I_c$. This is the DVRI with the most contextual information in the set $\Theta$ [S1].
3. Calculate the data-domain context metric $C_{data}(I_c, I_m)$ and the image-domain context metric $C_{image}(I_c, I_m)$ against $I_m$ [S1].
4. Compute $\delta_{data}$:
   - If $C_{data}(I_c, I_m) \geq t_{data}$, then $\delta_{data} = 0$ [S1].
   - Otherwise, $\delta_{data} = 1 - \frac{C_{data}(I_c, I_m)}{t_{data}}$ [S1].
5. Compute $\delta_{image}$:
   - If $C_{image}(I_c, I_m) \geq t_{image}$, then $\delta_{image} = 0$ [S1].
   - Otherwise, $\delta_{image} = 1 - \frac{C_{image}(I_c, I_m)}{t_{image}}$ [S1].
6. Determine the final context metric $C(c)$:
   - Check if there exists any $I_p \in \Theta$ such that $C_{data}(I_c, I_p) \geq t_{data}$ AND $C_{image}(I_c, I_p) \geq t_{image}$ [S1].
   - If such an $I_p$ exists, $C(c) = 0$ [S1].
   - Otherwise, $C(c) = (1 + (\delta_{data} + \delta_{image})) \cdot K_c$ [S1].

**Formulas:**
The final context metric is defined as [S1]:

$$
C(c) = 
\begin{cases} 
0, & \text{if } \exists I_p \in \Theta \text{ such that } C_{data}(I_c, I_p) \geq t_{data} \text{ and } C_{image}(I_c, I_p) \geq t_{image} \\
(1 + (\delta_{data} + \delta_{image})) \cdot K_c, & \text{otherwise}
\end{cases}
$$

Where $\delta_{data}$ and $\delta_{image}$ are defined as [S1]:

$$
\delta_{data} = 
\begin{cases} 
0, & \text{if } C_{data}(I_c, I_m) \geq t_{data} \\
1 - \frac{C_{data}(I_c, I_m)}{t_{data}}, & \text{otherwise}
\end{cases}
$$

$$
\delta_{image} = 
\begin{cases} 
0, & \text{if } C_{image}(I_c, I_m) \geq t_{image} \\
1 - \frac{C_{image}(I_c, I_m)}{t_{image}}, & \text{otherwise}
\end{cases}
$$

Here, $I_m \in \Theta$ is the DVRI with the minimum value of $(\delta_{data} + \delta_{image})$ [S1].

**Parameters:**
- $t_{data}$: Threshold for data-domain context [S1].
- $t_{image}$: Threshold for image-domain context [S1].
- $K_c$: Large constant for penalty scaling [S1].

**Edge Cases and Pitfalls:**
- If a metric is below its threshold, the penalty is calculated as $(1 - \frac{Metric}{Threshold}) \cdot K_c$ [S1].
- If both metrics are satisfactory for any $I_p \in \Theta$, the result is 0 [S1].
- The sources do not provide the explicit formulas for $C_{data}$ and $C_{image}$ themselves, only their usage in the context metric [S1].

## Subtopics

### Final Context Metric
The final context metric $C(c)$ for a candidate solution $c$ generating DVRI $I_c$, given a set of previously shown DVRIs $\Theta$, returns 0 if context is good enough, otherwise a positive value indicating worse context [S1]. It uses thresholds $t_{data}$ and $t_{image}$ and a large constant $K_c$ [S1]. A pitfall is that if a metric is below its threshold, the penalty is calculated as $(1 - \frac{Metric}{Threshold}) \cdot K_c$ [S1].

## Operating characteristics
Not given in the sources.

## Background
The Context Metric was introduced in the paper "Visibility-Aware Direct Volume Rendering" [S1]. It is used in the context of automatic DVRI generation, often employing genetic algorithms for optimization [S1]. The metric addresses the problem where users may not be able to easily establish relations among structures revealed in different DVRIs if no common structure is presented [S1].

## Sources
[S1] Visibility-Aware Direct Volume Rendering, https://doi.org/10.1007/s11390-011-9428-3, fetched 2026-10-26
