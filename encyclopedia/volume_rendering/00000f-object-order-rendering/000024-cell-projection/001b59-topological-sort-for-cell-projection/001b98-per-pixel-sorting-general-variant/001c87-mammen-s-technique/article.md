# Mammen's Technique
Also called: Per-pixel transparent fragment sorting
## Overview
Mammen's Technique is a software algorithm to sort per-pixel transparent fragments in front of the front-most opaque one [S1, unofficial]. It addresses the visibility ordering problem in object-order rendering where transparent surfaces must be composited from back to front [G]. Unlike incremental methods that exploit spatial coherence between adjacent pixels, Mammen's Technique performs independent, complete sorts for each pixel from scratch [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A set of polygonal fragments projected onto a specific pixel. Each fragment has a depth value ($z$) and an opacity/transparency attribute [G].
- Output: A sorted list of transparent fragments ordered from back to front, ready for compositing in front of the front-most opaque fragment [S1, unofficial].

**Data Structures:**
- A list of fragments projecting onto the current pixel. Each fragment contains its depth $z$ and rendering attributes [G].

**Steps:**
1. **Projection:** Project all scene polygons onto the current pixel. Identify fragments that cover the pixel [G].
2. **Opaque Identification:** Identify the front-most opaque fragment. Let its depth be $z_{opaque}$ [G].
3. **Transparent Filtering:** Collect all transparent fragments with depth $z < z_{opaque}$ [S1, unofficial]. Fragments with $z > z_{opaque}$ are occluded by the opaque fragment and discarded [G].
4. **Sorting:** Sort the collected transparent fragments by depth $z$ in ascending order (back to front) [S1, unofficial].
5. **Compositing:** Composite the sorted transparent fragments in order, then composite the front-most opaque fragment [G].

**Formulas:**
- Depth comparison: $z < z_{opaque}$ [G]
- Compositing order: $\text{Result} = \text{Composite}(\text{Sorted Transparent Fragments}, \text{Front-most Opaque Fragment})$ [G]

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Multiple Opaque Fragments:** The algorithm assumes a single front-most opaque fragment [G].
- **Precision:** Depth comparison precision must be sufficient to correctly determine occlusion by the front-most opaque fragment [G].
- **Performance:** The method requires multiple passes through the polygons in the scene if implemented naively, as it does not incrementally update the sort from the previous pixel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mammen_sort(fragments):
    """
    Sorts transparent fragments for a single pixel using Mammen's Technique.
    fragments: List of fragment objects with attributes .z (depth) and .opaque (boolean).
    Returns: List of transparent fragments sorted back-to-front, followed by the front-most opaque fragment.
    """
    # Step 1: Identify the front-most opaque fragment
    opaque_fragments = [f for f in fragments if f.opaque]
    if not opaque_fragments:
        front_most_opaque_z = float('inf')
        front_most_opaque = None
    else:
        front_most_opaque = min(opaque_fragments, key=lambda f: f.z)
        front_most_opaque_z = front_most_opaque.z

    # Step 2: Filter transparent fragments
    transparent_fragments = []
    for f in fragments:
        if not f.opaque and f.z < front_most_opaque_z:
            transparent_fragments.append(f)

    # Step 3: Sort transparent fragments back-to-front (ascending z)
    transparent_fragments.sort(key=lambda f: f.z)

    # Step 4: Return sorted list
    result = transparent_fragments
    if front_most_opaque is not None:
        result.append(front_most_opaque)
    return result
```
## Subtopics

### [R-Buffer](2357-r-buffer.md)
The R-Buffer is a hardware architectural proposal for implementing order-independent transparency (OIT) by storing fragments in a large, unordered, recirculating buffer separate from the Z-buffer and frame buffer. It solves the problem of correctly compositing an arbitrary number of transparent layers per pixel without requiring multiple passes of geometry or per-pixel linked lists. Unlike software-based sorting methods that repeat work for every pixel, the R-buffer amortizes storage costs across the entire screen, allowing fragments to be processed in successive passes until the framebuffer is fully composited.

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ per pixel for sorting $N$ transparent fragments, plus $O(N)$ for filtering and identification [G]. Since the sort is performed independently for each pixel without exploiting coherence, the total time is proportional to the number of pixels times the average number of fragments per pixel [S1, unofficial].
- **Memory Complexity:** $O(N)$ per pixel to store the fragments [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance degrades linearly with the number of fragments per pixel. It does not scale well with high fragment density compared to incremental methods [G].
- **Strengths:** Simple to implement; handles visibility cycles gracefully per pixel; correctly handles occlusion by the front-most opaque fragment [G].
- **Limitations:** Computationally expensive due to repeated sorting; does not exploit spatial coherence; requires multiple passes through polygons in naive implementations [S1, unofficial].
## Background
Mammen's Technique was described in the context of comparing software algorithms for per-pixel transparent fragment sorting with proposed hardware solutions like the R-Buffer [S1, unofficial]. It serves as a baseline for understanding the computational cost of per-pixel sorting without coherence exploitation [G]. The technique is cited in discussions about the trade-offs between software sorting and hardware-based OIT solutions [S1, unofficial].
## Sources
[S1] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
