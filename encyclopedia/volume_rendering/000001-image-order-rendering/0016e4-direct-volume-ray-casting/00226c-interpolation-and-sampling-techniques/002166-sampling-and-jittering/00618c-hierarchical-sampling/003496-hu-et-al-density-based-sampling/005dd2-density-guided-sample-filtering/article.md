# Density-guided Sample Filtering

Also called: Density-based Sample Culling

## Overview
Density-guided Sample Filtering is a rendering optimization that reduces the computational cost of volumetric appearance evaluation by culling sample points based on their contribution to the final image [S1, unofficial]. It improves GPU runtime without sacrificing visual quality by evaluating the expensive appearance network only at "valuable" points identified via a lightweight density model [S1, unofficial]. Unlike other sample culling methods that suffer from excessive branching or fail near object boundaries, this method fits the GPU execution model by using a priority queue to determine evaluation order [S1, unofficial].

## How it works
The method splits the computation into a cheap density estimation phase and an expensive appearance evaluation phase. It evaluates the density model $\pi_d$ densely along the ray to compute blending weights, then uses these weights to prioritize which samples require the full appearance MLP $\pi_c$ [S1, unofficial].

**Inputs and Outputs**
*   **Input:** Ray origin $o$, direction $d$, pre-computed feature grid $\psi$, density decoder $\pi_d$, appearance decoder $\pi_c$, number of samples $N$ per ray.
*   **Output:** Final pixel color $C(o, d)$.

**Data Structures**
1.  **Feature Grid $\psi$:** A 3D texture storing view-independent features. Access is via trilinear interpolation [S1, unofficial].
2.  **Density Decoder $\pi_d$:** A single $4 \times 4$ fully-connected layer followed by a SoftPlus activation [S1, unofficial].
3.  **Appearance Decoder $\pi_c$:** A larger MLP (details not specified in sources) taking features and view direction [S1, unofficial].
4.  **Priority Queue:** A max-heap storing tuples $(w_i, i)$, where $w_i$ is the blending weight and $i$ is the sample index [S1, unofficial].

**Formulas**
The density $\sigma_i$ for a sample $i$ is computed from its feature vector $f_i$:
$$ \sigma_i = \text{SoftPlus}(\pi_d(f_i)) $$
The blending weight $w_i$ is computed using standard volume rendering accumulation (typically $w_i = \alpha_i \prod_{j=0}^{i-1} (1-\alpha_j)$ where $\alpha_i = 1 - e^{-\sigma_i \delta}$) [S1, unofficial]. The final color is:
$$ C(o, d) = \sum_{i=1}^{N} w_i c_i $$
where $c_i$ is the color output by $\pi_c$ [S1, unofficial].

**Parameters**
*   $\pi_d$ architecture: $4 \times 4$ fully-connected layer [S1, unofficial].
*   Activation: SoftPlus [S1, unofficial].
*   Aggregation density target: Used to terminate density evaluation early if the accumulated density is sufficient (value not specified) [S1, unofficial].

**Steps**
1.  **Sample Points:** Generate $N$ sample points along the ray $o, d$.
2.  **Feature Fetch:** For each sample point, fetch the feature vector $f_i$ from the grid $\psi$ using trilinear interpolation [S1, unofficial].
3.  **Dense Density Evaluation:** Evaluate $\pi_d(f_i)$ for all samples $i=1 \dots N$ (or until an aggregation density target is met). This requires one texture fetch and five SIMD dot products per sample [S1, unofficial].
4.  **Weight Computation:** Compute the blending weights $w_i$ for all evaluated samples based on $\sigma_i$ [S1, unofficial].
5.  **Priority Queue Construction:** Insert all $(w_i, i)$ pairs into a max-priority queue [S1, unofficial].
6.  **Filtered Appearance Evaluation:**
    *   While the priority queue is not empty:
        *   Extract the sample index $i$ with the highest weight $w_i$.
        *   Fetch features $f_i$ (if not cached) and view direction.
        *   Evaluate the appearance MLP $\pi_c$ to get color $c_i$ [S1, unofficial].
        *   Remove $i$ from the queue.
7.  **Compositing:** Accumulate the colors $c_i$ using their pre-computed weights $w_i$ to produce the final pixel color [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def density_guided_filtering(ray_o, ray_d, feature_grid, pi_d, pi_c, num_samples):
    # Step 1 & 2: Sample and fetch features
    samples = generate_samples(ray_o, ray_d, num_samples)
    features = [feature_grid.trilinear_interpolate(s) for s in samples]
    
    # Step 3: Dense density evaluation
    # pi_d is a 4x4 FC layer + SoftPlus
    densities = []
    for f in features:
        # Simplified representation of 4x4 FC + SoftPlus
        z = softplus(dot_product(f, pi_d.weights) + pi_d.bias)
        densities.append(z)
        
    # Step 4: Compute weights
    # Assuming standard volume rendering alpha calculation
    alphas = [1.0 - exp(-d) for d in densities]
    weights = []
    transmittance = 1.0
    for a in alphas:
        alpha = a * transmittance
        weights.append(alpha)
        transmittance *= (1.0 - a)
        
    # Step 5: Build priority queue
    # Queue stores (weight, index)
    pq = PriorityQueue()
    for i, w in enumerate(weights):
        pq.push((w, i))
        
    # Step 6: Filtered appearance evaluation
    colors = [None] * num_samples
    while not pq.is_empty():
        w, i = pq.pop()
        if colors[i] is None:
            # Evaluate expensive MLP only for valuable points
            colors[i] = pi_c(features[i], ray_d)
            
    # Step 7: Compositing
    final_color = vec3(0.0)
    for i in range(num_samples):
        if colors[i] is not None:
            final_color += weights[i] * colors[i]
            
    return final_color
```

**Edge Cases and Pitfalls**
*   **Boundary Rays:** Other culling methods fail for rays close to object boundaries; this method is robust [S1, unofficial].
*   **Multiple Intersections:** Other methods fail for rays with multiple major intersections; this method handles them via the priority queue [S1, unofficial].
*   **GPU Branching:** Excessive branching in culling logic degrades GPU performance; this method avoids branching by using a priority queue and sequential evaluation [S1, unofficial].
*   **Free Space Artifacts:** Using only photometric loss can cause artifacts in free space; weight regularization is recommended to encourage zero weights or consolidated weights [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Density evaluation is $O(N)$ for all samples. Appearance evaluation is $O(K)$ where $K \le N$ is the number of "valuable" points. The priority queue construction is $O(N \log N)$ or $O(N)$ depending on implementation [S1, unofficial].
*   **Memory Complexity:** Requires storage for features, densities, weights, and the priority queue. $O(N)$ space [S1, unofficial].
*   **Performance:** Significantly improves render performance on GPU hardware by reducing the number of expensive MLP evaluations [S1, unofficial].
*   **Strengths:** Fits GPU execution model (low branching), robust near boundaries and multiple intersections, maintains high quality [S1, unofficial].
*   **Limitations:** Requires extra pass for density evaluation and weight computation. The priority queue adds overhead for small $N$ or when most points are valuable [S1, unofficial].

## Background
Density-guided Sample Filtering was introduced in the context of "Neural Assets," a system for volumetric object capture and rendering for interactive environments [S1, unofficial]. It addresses the bottleneck of evaluating large neural networks for every sample point in volume rendering [S1, unofficial]. The method was designed to be transpiled into efficient shader code (HLSL/GLSL) for integration into render and game engines [S1, unofficial].

## Sources
[S1] Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments, https://core.ac.uk/download/552466290.pdf, fetched 2026-10-30 (unofficial)
