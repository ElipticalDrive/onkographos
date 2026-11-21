# Coherence Metric

Also called: Focus Structure Coherence

## Overview
The Coherence Metric evaluates whether a focus structure's shading distribution is preserved in a rendered Direct Volume Rendering Image (DVRI), ensuring a clear and comprehensible presentation without distortion from partial occlusion [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A candidate rendering solution $c$, a defined focus structure $f$ (unique within the candidate), and a visibility threshold $t_{sv}$ [S1].
*   **Output:** A coherence score $H(c)$ [S1].

**Data Structures:**
*   `image_f`: Grayscale image of the exclusive rendering of $f$ [S1].
*   `image_c`: Grayscale image of the candidate rendering $c$ [S1].
*   `image_diff`: Image storing absolute pixel differences [S1].
*   $P$: Set of pixel coordinates belonging to $f$ in `image_diff` [S1].

**Parameters:**
*   $t_{sv}$: Visibility threshold [S1].
*   $K_h$: Large constant used to penalize bad coherence when visibility is insufficient [S1].

**Steps:**
1.  **Visibility Check:** Calculate the visibility $V(f)$ of the focus structure [S1]. If $V(f) < t_{sv}$, the structure is not visible enough [S1]. Return a bad coherence score:
    $$ H(c) = \left(1 + \frac{t_{sv} - V(f)}{t_{sv}}\right) \cdot K_h $$
    The worse the visibility, the higher the penalty [S1].
2.  **Exclusive Rendering:** If $V(f) \ge t_{sv}$, exclusively render the focus structure $f$ to generate its structure image [S1].
3.  **Grayscale Conversion:** Convert the structure image of $f$ to a grayscale image, denoted as `image_f`, to capture luminance variation [S1].
4.  **Candidate Grayscale:** Convert the grayscale image of the full candidate rendering $c$ to `image_c` [S1].
5.  **Difference Calculation:** Compute the difference image `image_diff` where each pixel value is the absolute difference between the corresponding pixels in `image_f` and `image_c`:
    $$ \text{image\_diff}(x, y) = |\text{image\_f}(x, y) - \text{image\_c}(x, y)| $$
6.  **Pixel Set Identification:** Identify the set $P$ of pixels in `image_diff` that belong to the focus structure $f$ [S1].
7.  **Variance Calculation:** Calculate the coherence score $H(c)$ as the statistical variance $\sigma^2$ of the difference values in the set $P$:
    $$ H(c) = \sigma^2(P) $$
    Low variance implies high coherence, meaning the shading distribution of the focus structure is preserved consistently in the candidate rendering [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coherence_metric(candidate_rendering, focus_structure, visibility_threshold, large_constant):
    # 1. Check visibility
    visibility = compute_visibility(focus_structure, candidate_rendering) # [G] External visibility metric
    
    if visibility < visibility_threshold:
        penalty_factor = 1 + (visibility_threshold - visibility) / visibility_threshold
        return penalty_factor * large_constant
    
    # 2. Render focus structure exclusively
    structure_image = render_exclusively(focus_structure) # [G] External rendering step
    
    # 3. Convert to grayscale
    image_f = to_grayscale(structure_image) # [G] Standard luminance conversion
    
    # 4. Get candidate grayscale image
    image_c = to_grayscale(candidate_rendering) # [G] Standard luminance conversion
    
    # 5. Compute difference image
    image_diff = [[0.0] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            image_diff[y][x] = abs(image_f[y][x] - image_c[y][x])
            
    # 6. Identify pixels belonging to focus structure
    P = get_pixels_belonging_to_structure(focus_structure, image_diff) # [G] Mask-based selection
    
    # 7. Calculate variance of differences in P
    diff_values = [image_diff[p[1]][p[0]] for p in P]
    mean_val = sum(diff_values) / len(diff_values)
    variance = sum((v - mean_val) ** 2 for v in diff_values) / len(diff_values)
    
    return variance
```

**Edge Cases and Pitfalls:**
*   **Visibility Threshold Discontinuity:** The metric uses a hard threshold $t_{sv}$; if $V(f)$ drops below $t_{sv}$, the score jumps to the penalty term [S1].
*   **Penalty Scaling:** The penalty term scales linearly with how far $V(f)$ falls below $t_{sv}$, and $K_h$ must be large enough to represent bad coherence quality [S1].

## Subtopics

### [Context Metric](13254-context-metric.md)
Ensures relationships between volume structures remain understandable by maintaining shared context. It is used within the Coherence Metric framework to prevent the generation of DVRIs that lack relational context with previously shown views.

## Operating characteristics
*   **Complexity:** The method requires rendering the focus structure exclusively and the full candidate, plus pixel-wise operations. Complexity depends on the resolution and the number of samples for visibility calculation [S1].
*   **Strengths:** Specifically targets the clarity of individual structures by evaluating shading distribution preservation [S1].
*   **Limitations:** It is dependent on the definition of a "focus structure." It does not evaluate the overall quality of the volume rendering, only the coherence of a specific part. It requires a visibility metric to be defined and computed [S1].

## Background
The Coherence Metric was introduced in the context of Visibility-Aware Direct Volume Rendering to help users generate clear and comprehensible volume rendering images. It addresses the problem of inconsistent appearance of structures due to varying degrees of occlusion in different regions [S1].

## Sources
[S1] Visibility-Aware Direct Volume Rendering, https://doi.org/10.1007/s11390-011-9428-3, fetched 2026-10-26
