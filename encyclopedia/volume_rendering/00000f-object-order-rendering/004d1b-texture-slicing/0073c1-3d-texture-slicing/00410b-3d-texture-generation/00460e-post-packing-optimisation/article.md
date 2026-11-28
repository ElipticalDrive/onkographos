# Post-packing Optimisation

Also called: Aspect Ratio Packing Optimization, Slice Orientation Standardization

## Overview
Post-packing Optimisation is a corrective stage applied after greedy packing to optimize inefficiently packed layouts by enforcing a global slice orientation, serving as a 'damage control' measure when initial greedy packing fails or increases memory requirements [S2]. It addresses the problem of mixed slice orientations (tall vs. wide) in the final 3D texture, which causes significant memory padding waste [S1, unofficial]. Unlike its siblings, which focus on the initial packing strategy, this method operates on the already-packed result to standardize orientations and reduce the overall texture area [S1, unofficial][S2].

## How it works
**Inputs:** Set of dataset slices $S$, aspect ratio threshold $ratio$.
**Outputs:** Optimised 2D texture dimensions $(W, H)$ and the rotated/repacked slice set.

**Data Structures:**
*   **Slice:** Contains `width`, `height`, and `orientation` (tall, wide, or square-like).
*   **Strategy Cost:** Calculated as the total area of the bounding texture for a specific orientation constraint.

**Steps:**
1.  **Pre-classification:** For each slice $s \in S$, determine if it is "square-like" or "imbalanced" based on the $ratio$ threshold. If $|width - height| > ratio \times \min(width, height)$, label as imbalanced; otherwise, label as square-like [S1, unofficial].
2.  **Calculate Potential Strategy Costs:** Evaluate two layout strategies:
    *   **Strategy A (Uniformly Vertical/Tall):** Force all imbalanced slices to be tall.
        *   $Width_A = \sum widths_{square} + \sum widths_{tall} + \sum heights_{wide}$
        *   $Height_A = \sum heights_{square} + \sum heights_{wide} + \sum widths_{tall}$
        *   $Area_A = Width_A \times Height_A$
    *   **Strategy B (Uniformly Horizontal/Wide):** Force all imbalanced slices to be wide.
        *   $Width_B = \sum widths_{square} + \sum widths_{wide} + \sum heights_{tall}$
        *   $Height_B = \sum heights_{square} + \sum heights_{tall} + \sum widths_{wide}$
        *   $Area_B = Width_B \times Height_B$
    *   *Note:* The sums include the dimensions of the slices after any necessary rotation to fit the target orientation [S1, unofficial].
3.  **Select and Apply Optimal Rotation:** Compare $Area_A$ and $Area_B$.
    *   If $Area_A \le Area_B$: Select Strategy A. Final dimensions are $(Width_A, Height_A)$. For each slice originally classified as "wide", rotate it by 90 degrees and repack it to fit the tall orientation.
    *   Else: Select Strategy B. Final dimensions are $(Width_B, Height_B)$. For each slice originally classified as "tall", rotate it by 90 degrees and repack it to fit the wide orientation.
    *   Square-like slices remain unchanged [S1, unofficial].

**Parameters:**
*   **ratio:** User-defined aspect ratio threshold used to classify slices as square-like or imbalanced. Typical values are not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sparse Datasets:** Post-packing acts as a corrective measure but cannot always fully compensate for an inefficient initial layout on very large or sparse datasets [S2].
*   **Precision:** The rotation step is followed by a repacking process applied only to the affected slices, which reduces post-processing time [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def post_packing_optimization(slices, ratio):
    """
    Optimizes packed slices by standardizing orientation.
    
    Args:
        slices: List of slice objects with 'width' and 'height'.
        ratio: Aspect ratio threshold for classification.
        
    Returns:
        tuple: (final_width, final_height, optimized_slices)
    """
    square_slices = []
    tall_slices = []
    wide_slices = []
    
    # Step 1: Pre-classification
    for s in slices:
        min_dim = min(s.width, s.height)
        diff = abs(s.width - s.height)
        
        if diff > ratio * min_dim:
            # Imbalanced
            if s.height > s.width:
                tall_slices.append(s)
            else:
                wide_slices.append(s)
        else:
            # Square-like
            square_slices.append(s)
            
    # Step 2: Calculate Potential Strategy Costs
    # Strategy A: Force all imbalanced to be tall (vertical)
    # Wide slices must be rotated 90 deg (width becomes height, height becomes width)
    width_A = sum(s.width for s in square_slices) + sum(s.width for s in tall_slices) + sum(s.height for s in wide_slices)
    height_A = sum(s.height for s in square_slices) + sum(s.height for s in wide_slices) + sum(s.width for s in tall_slices)
    area_A = width_A * height_A
    
    # Strategy B: Force all imbalanced to be wide (horizontal)
    # Tall slices must be rotated 90 deg
    width_B = sum(s.width for s in square_slices) + sum(s.width for s in wide_slices) + sum(s.height for s in tall_slices)
    height_B = sum(s.height for s in square_slices) + sum(s.height for s in tall_slices) + sum(s.width for s in wide_slices)
    area_B = width_B * height_B
    
    # Step 3: Select and Apply Optimal Rotation
    optimized_slices = []
    
    if area_A <= area_B:
        # Strategy A: Uniformly Vertical
        final_width = width_A
        final_height = height_A
        
        for s in square_slices:
            optimized_slices.append(s) # No change
        for s in tall_slices:
            optimized_slices.append(s) # No change
        for s in wide_slices:
            # Rotate wide slices to tall
            rotated_s = Slice(height=s.width, width=s.height)
            optimized_slices.append(rotated_s)
    else:
        # Strategy B: Uniformly Horizontal
        final_width = width_B
        final_height = height_B
        
        for s in square_slices:
            optimized_slices.append(s) # No change
        for s in wide_slices:
            optimized_slices.append(s) # No change
        for s in tall_slices:
            # Rotate tall slices to wide
            rotated_s = Slice(height=s.height, width=s.width)
            optimized_slices.append(rotated_s)
            
    return (final_width, final_height, optimized_slices)

class Slice:
    def __init__(self, width, height):
        self.width = width
        self.height = height
```

## Subtopics

### Post-packing
A specific variant described as a "damage control" measure applied after greedy packing [S2]. It enforces a global slice orientation on the layout produced by the greedy packing algorithm to reorganize slices for better memory efficiency [S2]. Unlike greedy packing, which degrades on sparse datasets, post-packing can reclaim initial overhead even when the primary algorithm fails, though it cannot always fully compensate for an inefficient initial layout on very large or sparse datasets [S2]. On the Head128x128x137 dataset at density threshold $t=0.17$, post-packing converted a 157.2% memory increase (from greedy packing alone) into a final saving of 74.5% [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   The algorithm involves a single pass for classification, a constant-time calculation of two strategy costs, and a single pass for rotation and repacking of the minority group [S1, unofficial].
*   Repacking is applied only to the affected slices, reducing post-processing time [S1, unofficial].
*   Memory overhead is significantly reduced by minimizing padding [S1, unofficial].

**Measured Performance:**
*   On the Head128x128x137 dataset at $t=0.17$, it converted a 157.2% memory increase into a 74.5% saving [S2].
*   On sparse datasets like Legs512x128x263, performance of the initial greedy packing degrades considerably, but post-packing remains effective in reclaiming initial overhead [S2].

**Strengths and Limitations:**
*   **Strengths:** Effective at reducing memory overhead caused by mixed aspect ratios; acts as a robust corrective measure for failed greedy packing [S1, unofficial][S2].
*   **Limitations:** Cannot fully compensate for inefficient initial layouts on very large or sparse datasets [S2]. The CPU-intensive partitioning and increased number of draw calls can penalize rendering performance on discrete GPUs like the Nvidia 1050M [S2].

**When to Use:**
*   Use when greedy packing results in mixed slice orientations leading to high padding waste [S1, unofficial].
*   Use as a final corrective step when the initial packing algorithm fails or increases memory requirements [S2].

**When Not to Use:**
*   Avoid on very large or sparse datasets where the initial layout is highly inefficient, as post-packing may not fully recover the overhead [S2].
*   Consider regular 2D packing as an alternative if post-packing is insufficient [S2].

## Background
Originating from research on hybrid CPU-GPU approaches for memory-efficient volume rendering of medical image data, this method addresses the inefficiencies of greedy packing algorithms when dealing with varying slice aspect ratios [S1, unofficial][S2]. It is particularly relevant in medical imaging where slice dimensions can vary significantly [S1, unofficial].

## Sources
[S1] A Hybrid CPU-GPU Approach for Memory-Efficient Volume ..., https://www.researchgate.net/publication/408581180_A_Hybrid_CPU-GPU_Approach_for_Memory-Efficient_Volume_Rendering_of_Medical_Image_Data, fetched 2026-10-26 (unofficial)
[S2] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://doi.org/10.24132/csrn.2026-d53, fetched 2026-10-26
