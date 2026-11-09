# Stochastic Evolutive TF Design

Also called: Genetic Algorithm TF Design

## Overview
Stochastic Evolutive TF Design is a genetic algorithm-based method for generating Transfer Functions (TFs) where the TF lookup table itself serves as the chromosome, allowing combination with other 1D TF design methods [S1]. It solves the problem of manual TF specification by using evolutionary search guided by user selection of rendered thumbnails [S1]. It differs from vector-based genetic approaches by encoding the entire lookup table as genes, enabling direct integration with other 1D design techniques [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A current population of TFs and their associated rendered thumbnails.
*   **Output:** A new population of TFs and thumbnails for the next iteration.

**Data Structures:**
*   **Chromosome:** The TF lookup table itself. Each entry in the lookup table is a gene [S1].
*   **Population:** A set of chromosomes (TFs) and their corresponding rendered images (thumbnails) [S1].

**Parameters:**
*   **Cutting Points:** Two random points in the voxel value domain used for crossover [S1].
*   **Mutation Region:** Defined by a center and range (for M2, M3) or global scale (for M4) [S1].
*   **Limited Intervals:** Bounds for random values in M3 [S1].

**Steps:**
1.  **User Selection:** The user selects the best thumbnails/TFs from the current set to serve as parents for a new population [S1].
2.  **Evolution:** Generate descendants from the selected parents using mutation, crossover, or both, in a random way [S1].
3.  **New Population:** Obtain a new set of TFs and thumbnails. Preserve only the parents in the selection pool for the next iteration [S1].
4.  **Next Iteration:** The user defines parents of the next TF population by selecting and deselecting thumbnails of the new set [S1].
5.  **High-Resolution Inspection:** If a thumbnail's quality is low due to resolution, the user can inspect the volume at a higher resolution in a second level of interaction [S1].
6.  **Manual Specification (Optional):** At the second level, users can manually draw TFs by dragging the mouse to set opacity shapes or using the right button to set intervals of influence [S1].

**Crossover Operator:**
1.  Randomly select two parent TFs [S1].
2.  Randomly determine two cutting points in the voxel value domain [S1].
3.  Combine the interval between the two points of one TF with the two outer intervals (defined by the same two points) of the other TF [S1].

**Mutation Operators:**
*   **M1 (Triangular):**
    1.  Define an opacity value for the center of a mutation region [S1].
    2.  Apply linear interpolation between the central opacity value and the opacity values at each border of the mutation [S1].
*   **M2 (Random):**
    1.  Randomly set opacities for voxel values around the mutation center [S1].
*   **M3 (Rectangular):**
    1.  Randomly set the center, range, colors, and opacities of the mutation between limited intervals [S1].
*   **M4 (Global Scale):**
    1.  Apply a random global scale to all opacity values [S1].

**Pitfalls:**
*   Genetic operators produce high-frequency features in the TF domain which might cause artifacts [S1]. This is handled by using Engel's pre-integrated volume rendering method, which deals well with high frequencies [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def crossover(tf1, tf2):
    """
    Performs crossover between two TFs.
    tf1, tf2: Lists of opacity values representing the TF lookup tables.
    Returns: A new TF list.
    """
    # Randomly determine two cutting points
    # Assuming voxel values are indexed 0 to N-1
    n = len(tf1)
    if n < 2:
        return tf1.copy()
    
    # Ensure cut1 < cut2
    cut1, cut2 = sorted([random.randint(0, n - 1), random.randint(0, n - 1)])
    
    # Combine the interval between the two points of tf1 
    # with the two outer intervals of tf2
    # Outer intervals of tf2: [0, cut1] and [cut2, n-1]
    # Interval between points of tf1: [cut1, cut2]
    
    new_tf = []
    # Outer interval 1 from tf2
    new_tf.extend(tf2[0:cut1])
    # Interval between points from tf1
    new_tf.extend(tf1[cut1:cut2 + 1])
    # Outer interval 2 from tf2
    new_tf.extend(tf2[cut2 + 1:])
    
    return new_tf

def mutate_triangular(tf, center, opacity_center, range_start, range_end):
    """
    M1: Triangular mutation.
    tf: List of opacity values.
    center: Index of the mutation center.
    opacity_center: Opacity value at the center.
    range_start, range_end: Indices defining the mutation region borders.
    """
    new_tf = tf.copy()
    # Linear interpolation between central opacity and border opacities
    # Left side: from range_start to center
    if range_start <= center:
        for i in range(range_start, center + 1):
            # Interpolate between new_tf[range_start] and opacity_center
            # Assuming linear interpolation based on distance
            if range_start == center:
                new_tf[i] = opacity_center
            else:
                t = (i - range_start) / (center - range_start)
                new_tf[i] = new_tf[range_start] * (1 - t) + opacity_center * t
    
    # Right side: from center to range_end
    if center <= range_end:
        for i in range(center, range_end + 1):
            if center == range_end:
                new_tf[i] = opacity_center
            else:
                t = (i - center) / (range_end - center)
                new_tf[i] = opacity_center * (1 - t) + new_tf[range_end] * t
                
    return new_tf

def mutate_random(tf, center, range_start, range_end):
    """
    M2: Random mutation.
    tf: List of opacity values.
    center: Index of the mutation center.
    range_start, range_end: Indices defining the mutation region.
    """
    new_tf = tf.copy()
    # Randomly set opacities for voxel values around the mutation center
    for i in range(range_start, range_end + 1):
        new_tf[i] = random.random() # Assuming opacity in [0, 1]
    return new_tf

def mutate_rectangular(tf, center, range_start, range_end, color, opacity):
    """
    M3: Rectangular mutation.
    tf: List of opacity values.
    center: Index of the mutation center.
    range_start, range_end: Indices defining the mutation region.
    color: Color value (not used in opacity-only TF, but part of definition).
    opacity: Opacity value for the rectangle.
    """
    new_tf = tf.copy()
    # Randomly set the center, range, colors, and opacities of the mutation 
    # between limited intervals (parameters provided here).
    for i in range(range_start, range_end + 1):
        new_tf[i] = opacity
    return new_tf

def mutate_global_scale(tf, scale_factor):
    """
    M4: Global Scale mutation.
    tf: List of opacity values.
    scale_factor: Random global scale factor.
    """
    new_tf = [opacity * scale_factor for opacity in tf]
    # Clip to valid range [0, 1] if necessary
    new_tf = [max(0, min(1, op)) for op in new_tf]
    return new_tf
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Allows combination with other 1D TF design methods; user-guided selection allows for intuitive exploration; handles high-frequency TF features via Engel's pre-integrated volume rendering [S1].
*   **Limitations:** Genetic operators can produce high-frequency features causing artifacts if not handled by specific rendering methods [S1].
*   **When to use:** When manual TF specification is difficult and a guided evolutionary search is preferred.
*   **When not:** When fully automated TF generation without user interaction is required.

## Background
The method is based on an approach by Kindlmann and Durkin [S1], but also a more flexible implementation of the technique proposed by He et al. [S1]. It was introduced in the paper "Volume visualization and exploration through flexible transfer function design" [S1].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
