# Dartmouth Distance Sampling
Also called: PDF for Sampling, Probability Density Function (PDF) for Sampling, Segment-based Distance Sampling

## Overview
Dartmouth Distance Sampling is a Monte Carlo importance sampling technique for volume rendering that constructs a probability density function (PDF) matching the exponential transmittance component of the light transport integral to cancel it analytically [S1]. It differs from standard ray marching by treating the integration domain as a series of trapezoidal segments defined by the projected corners of the bounding box, allowing for direct sampling of the free path length rather than fixed or adaptive step iteration [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Camera ray origin $O$, direction $D$, bounding box corners projected onto the ray $t_{C,1}, t_{C,2}, t_{C,3}$ (sorted), extinction coefficient $\sigma_t$, and the distance from the ray origin to the light source direction $d_m(x)$ [S1].
*   **Output:** A sampled distance $x$ along the ray, drawn from the constructed PDF [S1].

**Data Structures**
The method relies on the geometric projection of the volume's bounding box corners onto the camera ray. These define three intervals (segments) along the ray:
1.  $[0, t_{C,1}]$
2.  $[t_{C,1}, t_{C,2}]$
3.  $[t_{C,2}, t_{C,3}]$ (and potentially beyond, though the source defines the PDF piecewise up to the third corner) [S1].

**Formulas**
The PDF is constructed by combining the probability of selecting a segment with the conditional probability of selecting a distance within that segment [S1].

1.  **Segment Selection Probability:** The probability of selecting segment $i$ is proportional to the radiance scattered within that segment relative to the total maximum unoccluded radiance [S1]:
    $$ P(x \in [t_{C,i-1}, t_{C,i}]) = \frac{\mathcal{I}_i}{\text{total}} $$
    Where $\mathcal{I}_i$ is the integral of transmittance over segment $i$ [S1].

2.  **Conditional Distance Probability:** Given segment $i$ is selected, the probability density for distance $x$ is the transmittance at $x$ normalized by the segment integral [S1]:
    $$ P(x | x \in [t_{C,i-1}, t_{C,i}]) = \frac{T'_i(x)}{\mathcal{I}_i} = \frac{e^{-\sigma_t(x + d_m(x))}}{\mathcal{I}_i} $$
    Where $T'_i(x)$ is the transmittance at $x$, and $d_m(x)$ is the distance from the point $x$ towards the light source [S1].

3.  **Final PDF:** The assembled PDF for a point $x$ relative to the projected corners $t_{C,1}, t_{C,2}, t_{C,3}$ is [S1]:
    $$ \text{PDF}(x) = \begin{cases} \frac{\mathcal{I}_1}{\text{total}} \cdot \frac{\mathcal{I}'_1(x)}{\mathcal{I}_1} & x < t_{C,1} \\ \frac{\mathcal{I}_2}{\text{total}} \cdot \frac{\mathcal{I}'_2(x)}{\mathcal{I}_2} & t_{C,1} < x < t_{C,2} \\ \frac{\mathcal{I}_3}{\text{total}} \cdot \frac{\mathcal{I}'_3(x)}{\mathcal{I}_3} & \text{otherwise} \end{cases} $$
    Note: The source uses $\mathcal{I}'_i$ in the numerator of the first two cases but $\mathcal{I}'_3(x)$ in the third, implying the conditional density varies with $x$ [S1].

**Steps**
1.  **Precompute Integrals:** Calculate $\mathcal{I}_i$ for each segment. For homogeneous media, this is done analytically using the exponential transmittance formula [S2, unofficial]. For heterogeneous media, subdivide the space and multiply transmittance evaluations [S2, unofficial].
2.  **Select Segment:** Generate a random number to select a segment $i$ based on the weights $\mathcal{I}_i / \text{total}$ [S1].
3.  **Sample Distance:** Within the selected segment, sample the exact distance $x$ using the conditional PDF $P(x | \text{segment } i)$ [S1]. This transitions from discrete bin selection to continuous distance sampling [S2, unofficial].
4.  **Evaluate:** Use the sampled $x$ to compute visibility and scattering [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dartmouth_distance_sampling(ray_origin, ray_dir, box_corners, sigma_t, light_dir):
    # 1. Project box corners onto the ray to get t_C values
    # t_C1, t_C2, t_C3 are the sorted intersection distances of the ray with the box faces
    t_C1, t_C2, t_C3 = project_corners_to_ray(ray_origin, ray_dir, box_corners)
    
    # 2. Precompute segment integrals (I_i)
    # For homogeneous medium, I_i = integral of exp(-sigma_t * (x + d_m(x))) over the segment
    # This requires knowing d_m(x), the distance to the light source from point x
    I_1 = compute_segment_integral(0, t_C1, sigma_t, light_dir)
    I_2 = compute_segment_integral(t_C1, t_C2, sigma_t, light_dir)
    I_3 = compute_segment_integral(t_C2, t_C3, sigma_t, light_dir)
    
    total_I = I_1 + I_2 + I_3
    
    # 3. Select segment based on probability I_i / total_I
    r = random_uniform(0, 1)
    if r < I_1 / total_I:
        segment = 1
        start, end = 0, t_C1
    elif r < (I_1 + I_2) / total_I:
        segment = 2
        start, end = t_C1, t_C2
    else:
        segment = 3
        start, end = t_C2, t_C3
        
    # 4. Sample x within the segment using the conditional PDF
    # The conditional PDF is proportional to exp(-sigma_t * (x + d_m(x)))
    # This typically requires inversion sampling or rejection sampling within [start, end]
    x = sample_from_conditional_pdf(segment, start, end, sigma_t, light_dir)
    
    return x

def compute_segment_integral(start, end, sigma_t, light_dir):
    # Analytical integration for homogeneous medium
    # Integral of exp(-sigma_t * (x + d_m(x))) dx
    # d_m(x) depends on geometry. If light is at infinity, d_m(x) is linear.
    # If light is a point, d_m(x) is distance to point.
    # Not given in sources for general d_m(x). Assuming simple case or numerical integration.
    # [G] Placeholder for specific analytical form based on light geometry.
    pass

def sample_from_conditional_pdf(segment, start, end, sigma_t, light_dir):
    # Sample x in [start, end] with PDF proportional to exp(-sigma_t * (x + d_m(x)))
    # [G] Implementation details for inversion/rejection sampling not provided in sources.
    pass
```

**Edge Cases and Pitfalls**
*   **Geometry Mismatch:** The method fails when the exit point along the camera ray is much closer than the distance from the origin towards the light source [S1].
*   **Non-Constant Distance:** It deviates in cases where the distance to the box boundary is not constant (i.e., $d_{E,S} \neq d_{E,E}$) [S1].
*   **Discontinuities:** The presence of box corners creates a function that is not smooth, with clear differences in the rate of change around the distance of the projected corners [S1].

## Subtopics

### Segment-based Distance Sampling
A variant that explicitly discretizes the ray into segments corresponding to geometric features (like box corners) before applying the PDF [S1].

### PDF for Sampling
Emphasizes the construction of the probability density function to match the integrand's exponential term [S1].

## Operating characteristics
*   **Complexity:** Requires precomputation of segment integrals, which can be costly for heterogeneous media [S2, unofficial].
*   **Strengths:** Cancels the exponential transmittance term analytically, reducing variance in Monte Carlo integration [S1].
*   **Limitations:** Fails or deviates significantly when the light source geometry does not align with the camera ray's exit point or when boundary distances are not constant [S1]. The PDF is not necessarily monotonically decreasing [S1].
*   **When to use:** Homogeneous media with simple light geometries where the exponential term dominates the variance [S1].

## Background
Dartmouth Distance Sampling was introduced in the paper "Collimated Whole Volume Light Scattering in Homogeneous Finite Media" [S1]. It is part of an optimized Monte Carlo integration strategy for volume rendering [S1].

## Sources
[S1] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
[S2] Volume Rendering: Implementation, https://computergraphics.stackexchange.com/questions/5220/volume-rendering-implementation?noredirect=1&lq=1, fetched 2026-10-30 (unofficial)
