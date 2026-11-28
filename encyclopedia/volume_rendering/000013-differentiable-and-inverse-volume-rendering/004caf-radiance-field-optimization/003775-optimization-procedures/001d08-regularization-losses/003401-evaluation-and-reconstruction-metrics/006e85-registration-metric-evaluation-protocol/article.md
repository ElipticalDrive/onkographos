# Registration Metric Evaluation Protocol

Also called: Optimization Independent Protocol, Skerl et al. Protocol

## Overview
The Registration Metric Evaluation Protocol is an optimization-independent method for evaluating rigid registration similarity measures by sampling the parameter space around a known "golden standard" transformation [S1, unofficial]. It distinguishes itself from optimization-dependent benchmarks by decoupling metric evaluation from the specific convergence behavior of a registration algorithm, providing statistical characterizations of the metric's landscape (accuracy, distinctiveness, capture range, local maxima, and non-convergence risk) [S1, unofficial].

## How it works
**Inputs:**
- A "golden standard" rigid transformation $T_{GS}$ (or an approximation if unavailable) [S1, unofficial].
- A similarity measure function $S(T)$ that returns a scalar value for a given transformation $T$ [S1, unofficial].
- Protocol parameters: Hyper-sphere radius $R$, number of rays $N$, samples per ray $M$ [S1, unofficial].

**Outputs:**
- Five statistical estimates: Accuracy (ACC), Measure of Distinctiveness (DO(r)), Capture Range (CR), Number of Local Maxima (NOM(r)), and Risk of Non-convergence (RON(r)) [S1, unofficial].

**Data Structures:**
- Parameter Space: A 6-dimensional space representing rigid body transformations (3 translations, 3 rotations).
- Normalization: The space is normalized such that a unit change in any parameter causes the same mean shift in voxels [S1, unofficial].
- Sampling Grid: $N$ rays, each containing $M+1$ equally spaced points.

**Steps:**
1. **Normalization and Sampling:** Define the normalized parameter space centered at $T_{GS}$. Sample a hyper-sphere of radius $R$ centered at $T_{GS}$ by generating $N$ random directions. Along each direction, place $M+1$ equally spaced pose parameters [S1, unofficial].
2. **Similarity Computation:** Compute the similarity measure $S(T)$ for all $N \times (M+1)$ sampled transformations [S1, unofficial].
3. **Statistical Estimation:** Calculate the five metrics based on the computed similarity values along each ray [S1, unofficial].

**Formulas:**
- Step size along a ray: $\delta = \frac{2}{R \cdot M}$ [S1, unofficial].
- Distance from origin: $r = k \cdot \delta$, where $k$ is the number of steps from the origin [S1, unofficial].

1. **Accuracy (ACC):** The root mean-square distance between the hypersphere origin ($T_{GS}$) and the global maxima found along each line [S1, unofficial].
   $$ \text{ACC} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} \| T_{GS} - T_{max,i} \|^2} $$
   where $T_{max,i}$ is the transformation yielding the global maximum on ray $i$ [S1, unofficial].

2. **Measure of Distinctiveness (DO(r)):** The average change of similarity measures near the global maximum at distance $r$ [S1, unofficial].
   $$ \text{DO}(r) = \frac{1}{N} \sum_{i=1}^{N} \text{Change}_i(r) $$
   where $\text{Change}_i(r)$ quantifies the variation of $S(T)$ within the neighborhood defined by $r$ from the global maximum on ray $i$ [S1, unofficial].

3. **Capture Range (CR):** The smallest distance between the position of the global maximum and the closest local minimum along each line [S1, unofficial].
   $$ \text{CR} = \min_{i=1..N} (\text{dist}(T_{max,i}, T_{min,i})) $$
   where $T_{min,i}$ is the closest local minimum to $T_{max,i}$ on ray $i$ [S1, unofficial].

4. **Number of Local Maxima (NOM(r)):** The total number of local maxima found at distance $r$ from each of the $N$ global maxima [S1, unofficial].
   $$ \text{NOM}(r) = \sum_{i=1}^{N} \text{Count}_{local\_maxima}(T_{max,i}, r) $$

5. **Risk of Non-convergence (RON(r)):** The average of positive gradients within distance $r$ from each of the $N$ global maxima [S1, unofficial].
   $$ \text{RON}(r) = \frac{1}{N} \sum_{i=1}^{N} \text{AvgPositiveGradient}_i(r) $$

**Pitfalls:**
- If no gold standard parameters are available, approximate parameters obtained from the registration method itself are used instead, which may bias the evaluation [S1, unofficial].
- The normalization of the parameter space is critical; incorrect scaling leads to misleading distance metrics [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The protocol was proposed by Skerl et al. [S1, unofficial]. It is used in medical volume visualization and preoperative planning systems, such as kidney registration, to evaluate the performance of various similarity measures (e.g., Mutual Information, Tsallis Mutual Information) [S1, unofficial].

## Sources
[S1] Research of New Medical Volume Visualization Methods: Application in the Kidney Preoperative Planning System, https://theses.hal.science/tel-00355629v1, fetched 2026-10-26 (unofficial)
