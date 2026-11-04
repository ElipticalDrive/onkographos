# k*-Buffer Analysis

Also called: k*-Buffer, k-value Analysis

## Overview
k*-Buffer Analysis is a software-based diagnostic method used to determine the optimal fixed buffer size $k$ for the k-buffer visibility sorting algorithm in hardware-assisted volume rendering [S2, unofficial]. It solves the problem of selecting a $k$ value that balances rendering precision against memory and computational efficiency by analyzing the depth-ordering requirements of a specific dataset [S1][S2, unofficial]. Unlike the unbounded A-buffer, which stores all fragments, or conservative sorting schemes like ZSWEEP, the k-buffer limits storage to $k$ entries per pixel, requiring an analysis of how "nearly sorted" the fragment stream is to ensure correct compositing [S2, unofficial].

## How it works
The method consists of two phases: the operational logic of the k-buffer itself and the analytical procedure to derive the required $k$.

### Inputs and Outputs
**Inputs:**
- A set of 3D primitives (e.g., tetrahedra or triangles) from a volume dataset.
- A camera viewpoint and projection parameters.
- A target resolution (e.g., $512 \times 512$) [S2, unofficial].

**Outputs:**
- The maximum required $k$ ($k_{max}$) for the dataset.
- The distribution of pixels requiring specific $k$ values (e.g., count of pixels where $k > 2$ or $k > 6$) [S2, unofficial].
- Identification of geometric degeneracies (e.g., sliver cells) causing high $k$ requirements [S2, unofficial].

### Data Structures
**A-Buffer (for Analysis):**
- A 2D array of lists, one per pixel.
- Each list stores fragments sorted by depth (distance from viewpoint).
- Fragment entry: `{depth: float, color: vec3, opacity: float}` [S2, unofficial].

**k-buffer (for Operational Logic):**
- A fixed-size array of $k$ entries per pixel.
- Entry: `{depth: float, color: vec3, opacity: float}` [S2, unofficial].
- In hardware implementations, entries are often kept unsorted to simplify logic, relying on the property that the input sequence is $k$-nearly sorted [S2, unofficial].

### Algorithm Steps

#### Phase 1: k*-Buffer Analysis (Software Diagnostic)
This phase simulates an ideal A-buffer to measure the true sorting distance of fragments.

1.  **Initialize** an A-buffer for the target resolution. Each pixel contains an empty sorted list.
2.  **Rasterize** the dataset from the viewpoint, generating a stream of fragments.
3.  **For each fragment** $f_{new}$ with depth $d_{new}$:
    a.  Find the correct insertion position in the pixel's sorted A-buffer list such that depths are in increasing order (for front-to-back compositing).
    b.  Let $i$ be the index of the insertion position (0-based).
    c.  **Record** the insertion depth metric: $k_{pixel} = \max(k_{pixel}, i)$.
        *   Note: The definition of $k$ in the analysis is "the number of positions any fragment had to move to its correct position in the sorted order minus one for compositing" [S2, unofficial]. Effectively, if a fragment must be inserted at index $i$, it implies that up to $i$ closer fragments exist. For compositing, we need access to the closest and second closest, so the buffer size requirement relates to this displacement. The analysis reports $k$ as the maximum index $i$ observed across all fragments for that pixel [S2, unofficial].
4.  **Aggregate** results:
    a.  $k_{max} = \max_{\text{pixels}} (k_{pixel})$.
    b.  Count pixels where $k_{pixel} > 2$.
    c.  Count pixels where $k_{pixel} > 6$.
5.  **Visualize** (optional) the spatial distribution of $k$ requirements to identify problematic regions (e.g., sliver cells) [S2, unofficial].

#### Phase 2: k-buffer Operation (Hardware/Rendering Logic)
This phase describes how the determined $k$ is used during rendering.

1.  **Initialize** a k-buffer of size $k$ for each pixel.
2.  **For each fragment** $f_{new}$ arriving at a pixel:
    a.  If the k-buffer is not full:
        i.   Insert $f_{new}$ into the buffer.
        ii.  If maintaining sorted order: shift elements to maintain depth order.
        iii. If unsorted (hardware simplification): append $f_{new}$ to the end.
    b.  If the k-buffer is full:
        i.   Determine if $f_{new}$ should replace an existing fragment.
        ii.  For front-to-back compositing, the fragment furthest from the viewer (largest depth) is the least relevant for early ray termination and compositing accuracy of the front surface.
        iii. Discard the fragment with the largest depth (or the first entry $f_1$ if sorted front-to-back, where $f_1$ is closest? *Correction*: The source states "dislodges the first entry ($f_1$)" for front-to-back. In a front-to-back sorted list, $f_1$ is the closest. However, standard k-buffer logic for *streaming* usually discards the *furthest* to keep the closest $k$. Let's re-read carefully: "For front-to-back compositing, each time a new fragment $f_{new}$ is inserted... it dislodges the first entry ($f_1$). Note that... $f_{new}$ may be inserted at the beginning... if it is closer... or at the end if further." This implies the buffer maintains the $k$ *closest* fragments. If $f_{new}$ is closer than all, it enters at 0, and the *furthest* (last entry) is discarded? No, the text says "dislodges the first entry ($f_1$)". This phrasing is ambiguous. Let's look at the property: "given a sequence... within k positions from its position in the sorted order... output fragments in correct order."
        
        *Re-evaluation of Logic based on Source [S2, unofficial]:*
        "For front-to-back compositing, each time a new fragment $f_{new}$ is inserted in the k-buffer, it dislodges the first entry ($f_1$)."
        If the buffer is sorted front-to-back ($f_1$ is closest, $f_k$ is furthest), and we insert $f_{new}$:
        - If $f_{new}$ is closer than $f_1$, it becomes the new $f_1$. The old $f_1$ is pushed out? No, that would lose the closest.
        - Standard k-buffer behavior: Keep the $k$ closest fragments. If a new fragment is closer than the furthest in the buffer, it enters, and the furthest leaves.
        - The source text says: "dislodges the first entry ($f_1$)". This might be a typo in the source or a specific definition where $f_1$ is the *furthest*. Let's look at "increasing order for front-to-back". Increasing order means $f_1 < f_2 < ... < f_k$. $f_1$ is closest.
        - If $f_{new}$ is inserted, and it dislodges $f_1$, we are losing the closest fragment. This contradicts front-to-back compositing needs.
        - *Alternative interpretation*: The source might define $f_1$ as the *last* entry in the list (index 1 in 1-based indexing of a list that grows?). Or, more likely, the source describes a buffer that stores the $k$ *most recent* or *relevant* fragments, and the "first entry" refers to the one being evicted.
        - Let's stick to the explicit text: "dislodges the first entry ($f_1$)". And "$f_{new}$ may be inserted at the beginning... if it is closer... or at the end if further."
        - If $f_{new}$ is closer than all, it goes to beginning. The list shifts. Who is dislodged? The *last* entry ($f_k$) should be dislodged to keep the $k$ closest.
        - If $f_{new}$ is further than all, it goes to end. The *first* entry ($f_1$) is dislodged? That would remove the closest.
        - *Conclusion on Ambiguity*: The source text contains a potential contradiction or non-standard definition. However, the key property is: "output the fragments in the correct order" if the input is $k$-nearly sorted.
        - *Implementation Note*: To implement correctly for front-to-back, one must keep the $k$ closest fragments. If the buffer is full, and $f_{new}$ is closer than the furthest fragment in the buffer, insert $f_{new}$ and discard the furthest. If $f_{new}$ is further than all, it is discarded (or replaces the furthest if the buffer is not strictly "closest k" but "window k").
        - Given the source says "dislodges the first entry ($f_1$)" when $f_{new}$ is inserted, and $f_{new}$ can be at the beginning, it is highly likely the source defines $f_1$ as the *evicted* element, which in a "keep closest" strategy is the *last* element. Or, the source defines the list in *decreasing* order (furthest first).
        - Let's look at: "increasing order for front-to-back". This means $f_1$ is closest.
        - If the text says "dislodges $f_1$", and $f_1$ is closest, this is wrong for volume rendering.
        - *Decision*: I will describe the standard k-buffer logic which aligns with the "k-nearly sorted" property and the goal of keeping closest fragments, while noting the source's specific phrasing about dislodging $f_1$ as a potential definition of $f_1$ as the *tail* or an error in the source text. However, to be safe and "implementable", I will follow the standard interpretation of k-buffer: **Keep the $k$ closest fragments.**
        
        *Re-reading Source [S2, unofficial] carefully:* "For front-to-back compositing... it dislodges the first entry ($f_1$). Note that... $f_{new}$ may be inserted at the beginning... if it is closer... or at the end if further."
        If $f_{new}$ is inserted at the *end* (further than all), it means $f_{new}$ is the new furthest. The buffer should discard the *closest* ($f_1$)?? No, that loses data.
        Actually, if the buffer stores the $k$ fragments that are *currently active* in the ray, and the ray moves forward, the "first" fragment (closest) is passed to the compositing stage and removed from the buffer? No, the buffer is per-pixel, static during the frame.
        
        Let's look at the "k-nearly sorted" property. This implies the buffer acts as a sliding window of size $k$ over the sorted stream.
        
        *Safe Implementation Description*:
        The k-buffer maintains the $k$ fragments with the smallest depth values (closest to viewer).
        1. If buffer size $< k$, insert $f_{new}$.
        2. If buffer size $== k$:
           a. If $depth(f_{new}) < depth(f_{max})$ (where $f_{max}$ is the fragment with largest depth in buffer):
              i.   Insert $f_{new}$.
              ii.  Remove $f_{max}$.
           b. Else:
              i.   Discard $f_{new}$.
        
        This ensures the $k$ closest are kept. The source's mention of "dislodging $f_1$" likely refers to a specific implementation detail or a typo where $f_1$ denotes the *last* element in a 0-indexed array or the element being evicted. I will note the source's specific text but provide the logically correct implementation for volume rendering.

3.  **Compositing**:
    a.  Retrieve the $k$ fragments from the buffer.
    b.  Sort them by depth (if not kept sorted).
    c.  Perform front-to-back compositing using the pre-integrated table lookup. Note that $k+1$ entries are required for correct compositing of a $k$-nearly sorted sequence because the closest and second closest must be available for the lookup [S2, unofficial].

### Parameters
- **$k$**: The number of fragment entries stored per pixel.
    - **Typical Value**: 6 is the standard value in existing HAVS implementations [S1][S2, unofficial].
    - **Experimental Values**: 6, 8, 10, 12 [S1].
    - **Trade-off**: Larger $k$ yields more precise results but lower efficiency (higher memory and processing time) [S1].

### Edge Cases and Pitfalls
- **Highly Out-of-Order Input**: If the incoming fragment sequence is not $k$-nearly sorted, the output will be incorrect and artifacts may be visible [S2, unofficial].
- **Sliver Cells**: Large, thin cells with bad aspect ratios cause high $k$ requirements.
    - *Solution 1*: Subdivide degenerate cells into smaller, more symmetric cells using Delaunay tetrahedralization properties (empty circumsphere) [S2, unofficial].
    - *Solution 2*: Mesh smoothing operations to eliminate bad cells without creating additional tetrahedra [S2, unofficial].
- **Artifact Visibility**: Artifacts from limited $k$ are often hard to notice, especially with transparent transfer functions. In worst-case scenarios (e.g., Torso dataset), only a small percentage of pixels (0.6%) were truly incorrect [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def k_buffer_analysis(dataset, resolution, viewpoint):
    """
    Performs k*-Buffer Analysis to determine the required k for a dataset.
    Returns max_k and pixel distribution stats.
    """
    # Initialize A-buffer: list of lists, one per pixel
    # Each inner list stores fragments sorted by depth (ascending)
    a_buffer = [[[] for _ in range(resolution[1])] for _ in range(resolution[0])]
    
    # Track max k per pixel
    k_per_pixel = [[0 for _ in range(resolution[1])] for _ in range(resolution[0])]
    
    # Generate fragments from dataset
    fragments = rasterize(dataset, viewpoint, resolution)
    
    for frag in fragments:
        u, v = frag.screen_coords
        depth = frag.depth
        
        # Find insertion index in sorted A-buffer
        pixel_list = a_buffer[u][v]
        insert_idx = 0
        for i, existing_frag in enumerate(pixel_list):
            if depth < existing_frag.depth:
                insert_idx = i
                break
            else:
                insert_idx = i + 1
        
        # Insert fragment
        pixel_list.insert(insert_idx, frag)
        
        # Record k requirement for this pixel
        # k is defined as the number of positions the fragment had to move 
        # (or its index in the sorted list) minus one for compositing logic?
        # Source says: "number of positions any fragment had to move to its correct position 
        # in the sorted order minus one for compositing"
        # And "Max k" is the max index observed.
        # We use the index as the metric for k requirement.
        k_per_pixel[u][v] = max(k_per_pixel[u][v], insert_idx)
        
    # Aggregate results
    max_k = 0
    count_k_gt_2 = 0
    count_k_gt_6 = 0
    
    for u in range(resolution[0]):
        for v in range(resolution[1]):
            k_val = k_per_pixel[u][v]
            if k_val > max_k:
                max_k = k_val
            if k_val > 2:
                count_k_gt_2 += 1
            if k_val > 6:
                count_k_gt_6 += 1
                
    return {
        "max_k": max_k,
        "pixels_k_gt_2": count_k_gt_2,
        "pixels_k_gt_6": count_k_gt_6,
        "distribution": k_per_pixel
    }

def k_buffer_insert(buffer, frag, k):
    """
    Inserts a fragment into a k-buffer of size k.
    Maintains the k closest fragments.
    """
    # If buffer is not full, insert and sort
    if len(buffer) < k:
        buffer.append(frag)
        buffer.sort(key=lambda x: x.depth)
        return
    
    # Buffer is full
    # Find the fragment with the largest depth (furthest)
    furthest_idx = -1
    max_depth = -1
    for i, f in enumerate(buffer):
        if f.depth > max_depth:
            max_depth = f.depth
            furthest_idx = i
            
    # If new fragment is closer than the furthest in buffer, replace it
    if frag.depth < max_depth:
        buffer[furthest_idx] = frag
        buffer.sort(key=lambda x: x.depth)
    # Else, discard the new fragment
```

## Subtopics

### ZSWEEP
A sibling method that uses a conservative technique for compositing samples to avoid memory space explosion in the A-buffer. The k-buffer offers a less conservative sorting scheme but requires analysis to ensure $k$ is sufficient [S2, unofficial].

### A-buffer Visibility Sort
The parent method that uses an unbounded buffer. k-buffer analysis determines the fixed size $k$ needed to approximate the A-buffer's behavior efficiently [S2, unofficial].

## Operating characteristics
- **Time Complexity**:
    - Analysis: $O(N \log K)$ or $O(N \cdot K)$ depending on implementation, where $N$ is the number of fragments. The source notes that sorting $n$ fragments with a $k$-nearly sorted property can be done in $O(n)$ time [S2, unofficial].
    - Rendering: $O(1)$ per fragment insertion if $k$ is fixed and small.
- **Memory Complexity**: $O(k \cdot W \cdot H)$, where $W, H$ are image dimensions. This is significantly less than the unbounded A-buffer $O(N \cdot W \cdot H)$ in worst cases [S2, unofficial].
- **Measured Performance**:
    - Hardware: Intel Pentium Dual 1.60GHz, NVIDIA GeForce9800 GTX [S1].
    - Datasets:
        - **F117**: 2.5M fragments, Max A-buffer 481, Max $k$ 15. 71 pixels require $k > 6$ [S2, unofficial].
        - **Kew**: 2.8M fragments, Max $k$ 3. 0 pixels require $k > 6$ [S2, unofficial].
        - **Spx2**: 6.6M fragments, Max $k$ 22. 512 pixels require $k > 6$ [S2, unofficial].
        - **Torso**: 7.2M fragments, Max $k$ 15. 1683 pixels require $k > 6$ [S2, unofficial].
        - **Fighter**: 5.4M fragments, Max $k$ 3. 0 pixels require $k > 6$ [S2, unofficial].
- **Strengths**:
    - Reduces memory requirements compared to A-buffer [S2, unofficial].
    - Simple enough for hardware implementation [S2, unofficial].
    - Artifacts are often unnoticeable, especially with transparency [S2, unofficial].
- **Limitations**:
    - Requires preprocessing/analysis to determine optimal $k$ [S2, unofficial].
    - Sensitive to sliver cells, which can inflate $k$ requirements [S2, unofficial].
    - Fixed $k$ may lead to incorrect ordering if the dataset has high depth complexity in specific regions [S2, unofficial].

## Background
- **Origin**: The k-buffer was introduced as part of the Hardware-Assisted Visibility Sorting (HAVS) algorithm for unstructured grid volume rendering [S1][S2, unofficial].
- **History**: Developed by Steve Callahan and colleagues, as detailed in the MS thesis "The k-Buffer and Its Applications to Volume Rendering" [S2, unofficial]. The analysis method was used to validate the effectiveness of the k-buffer on various datasets (F117, Kew, Spx2, Torso, Fighter) [S2, unofficial].
- **Development**: Subsequent work optimized the HAVS algorithm by adjusting $k$ values (6, 8, 10, 12) to balance precision and efficiency, showing that higher $k$ improves precision but reduces performance [S1].

## Sources
[S1] Optimization and Implementation of Unstructured Grid Volume Rendering Algorithm, https://doi.org/10.2991/iccasm.2012.373, fetched 2026-10-26
[S2] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
