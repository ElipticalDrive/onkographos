# Post-packing Optimisation

Also called: Aspect Ratio Packing Optimization, Slice Orientation Standardization

## Overview
Post-packing Optimisation is a corrective stage applied after greedy packing to optimize inefficiently packed layouts by enforcing a global slice orientation, serving as a 'damage control' measure when initial greedy packing fails or increases memory requirements [S2]. It addresses the memory waste caused by mixing tall (height > width) and wide (width > height) slices in the final 3D texture, which forces the texture dimensions to accommodate the maximum width and height across all slices, leading to significant padding [S1, unofficial]. Unlike its sibling [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md) which may produce mixed orientations, this method standardizes orientations to minimize the final texture area [S1, unofficial].

## How it works
**Inputs:** Set of dataset slices $S$, aspect ratio threshold $ratio$.
**Outputs:** Optimised 2D texture dimensions $(W, H)$ and repacked slice orientations.

**Data Structures:**
*   **Slice:** Contains `width`, `height`, and `orientation` (tall/wide/square-like).
*   **Strategy:** Contains `Width`, `Height`, `Area`.

**Steps:**
1.  **Pre-classification:** For each slice $s$ in $S$:
    *   If $|width - height| / \max(width, height) > ratio$: Label $s$ as "imbalanced".
    *   Else: Label $s$ as "square-like".
    *   [G] For imbalanced slices, determine initial orientation: if $height > width$, it is tall; else wide.
2.  **Calculate Potential Strategy Costs:**
    *   **Strategy A (Uniformly Vertical/Tall):**
        *   $Width_A = \sum_{s \in square} width_s + \sum_{s \in tall} width_s + \sum_{s \in wide} height_s$
        *   $Height_A = \sum_{s \in square} height_s + \sum_{s \in wide} width_s + \sum_{s \in tall} height_s$
        *   $Area_A = Width_A \times Height_A$
    *   **Strategy B (Uniformly Horizontal/Wide):**
        *   $Width_B = \sum_{s \in square} width_s + \sum_{s \in wide} width_s + \sum_{s \in tall} height_s$
        *   $Height_B = \sum_{s \in square} height_s + \sum_{s \in tall} width_s + \sum_{s \in wide} height_s$
        *   $Area_B = Width_B \times Height_B$
3.  **Select and Apply Optimal Rotation:**
    *   If $Area_A < Area_B$:
        *   Final dimensions $W = Width_A, H = Height_A$.
        *   For each $s$ in "wide slices": RotateAndRepack($s$).
    *   Else:
        *   Final dimensions $W = Width_B, H = Height_B$.
        *   For each $s$ in "tall slices": RotateAndRepack($s$).

**Parameters:**
*   $ratio$: User-defined aspect ratio threshold used to classify slices as square-like or imbalanced. [S1, unofficial]

**Edge Cases:**
*   **Sparse Datasets:** Greedy packing may degrade on sparse datasets, but post-packing acts as a corrective measure to reclaim initial overhead [S2].
*   **Very Large/Sparse Datasets:** Post-packing cannot always fully compensate for an inefficient initial layout on very large or sparse datasets [S2].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def post_packing_optimization(slices, ratio):
    square_slices = []
    tall_slices = []
    wide_slices = []

    for s in slices:
        if abs(s.width - s.height) / max(s.width, s.height) > ratio:
            if s.height > s.width:
                tall_slices.append(s)
            else:
                wide_slices.append(s)
        else:
            square_slices.append(s)

    # Strategy A: Force all to be tall (vertical)
    width_A = sum(s.width for s in square_slices) + sum(s.width for s in tall_slices) + sum(s.height for s in wide_slices)
    height_A = sum(s.height for s in square_slices) + sum(s.width for s in wide_slices) + sum(s.height for s in tall_slices)
    area_A = width_A * height_A

    # Strategy B: Force all to be wide (horizontal)
    width_B = sum(s.width for s in square_slices) + sum(s.width for s in wide_slices) + sum(s.height for s in tall_slices)
    height_B = sum(s.height for s in square_slices) + sum(s.width for s in tall_slices) + sum(s.height for s in wide_slices)
    area_B = width_B * height_B

    if area_A < area_B:
        final_w, final_h = width_A, height_A
        # Rotate wide slices to become tall
        for s in wide_slices:
            s.width, s.height = s.height, s.width
    else:
        final_w, final_h = width_B, height_B
        # Rotate tall slices to become wide
        for s in tall_slices:
            s.width, s.height = s.height, s.width

    return final_w, final_h
```

## Subtopics

### Post-packing
A corrective stage applied after greedy packing to optimize inefficiently packed layouts by enforcing a global slice orientation, serving as a 'damage control' measure when initial greedy packing fails or increases memory requirements [S2]. It converts a memory increase from greedy packing into a final saving; for example, on the Head128x128x137 dataset at density threshold $t=0.17$, it converted a 157.2% memory increase into a final saving of 74.5% [S2]. Unlike greedy packing, which degrades on sparse datasets, post-packing acts as a corrective measure that can reclaim initial overhead even when the primary algorithm fails, though it cannot always fully compensate for an inefficient initial layout on very large or sparse datasets [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Reduces memory overhead by minimizing padding. On the Head128x128x137 dataset at $t=0.17$, it achieved a 74.5% final saving after correcting a 157.2% increase from greedy packing [S2].
*   **Performance:** On a discrete Nvidia 1050M GPU, there is an inverse relationship between the number of boxes generated and frame rate, but post-packing primarily affects memory efficiency rather than rendering speed directly [S2].
*   **Strengths:** Effective "damage control" for inefficient initial layouts; significant memory savings on dense datasets [S2].
*   **Limitations:** Cannot fully compensate for inefficient initial layouts on very large or sparse datasets [S2].

## Background
Originated in the context of hybrid CPU-GPU approaches for memory-efficient volume rendering of medical image data [S1, unofficial][S2]. It was developed to address the memory waste caused by mixed slice orientations in 3D textures generated via greedy packing [S1, unofficial][S2].

## Sources
[S1] A Hybrid CPU-GPU Approach for Memory-Efficient Volume ..., https://www.researchgate.net/publication/408581180_A_Hybrid_CPU-GPU_Approach_for_Memory-Efficient_Volume_Rendering_of_Medical_Image_Data, fetched 2026-10-26 (unofficial)
[S2] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://doi.org/10.24132/csrn.2026-d53, fetched 2026-10-26
