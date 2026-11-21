# Mammen's Technique

Also called: A-buffer, A-buffer visibility sort, Carpenter's A-buffer, Per-pixel transparent fragment sorting

## Overview
Mammen's Technique is a software algorithm that sorts per-pixel transparent fragments in front of the front-most opaque one [S2, unofficial]. It operates on pixel fragments instead of object fragments, allowing objects to be rasterized in any order without requiring object-space visibility sorting [S1, unofficial][S3]. This distinguishes it from object-space methods like MPVO, which must handle complex degeneracies and ordering constraints before rasterization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of geometric primitives (polygons) defining the scene.
*   **Output:** A depth-sorted list of fragments for each pixel, ready for compositing.

**Data Structures:**
*   **Per-Pixel Linked Lists:** For each pixel $(x, y)$, a linked list stores fragments.
*   **Fragment Structure:** Each fragment stores its depth value $z$ and color/transparency attributes. The exact bit-packing of the fragment structure is not specified in the sources [G: Standard implementation would include $z$, color, and a pointer to the next fragment].

**Steps:**
1.  **Rasterization:** Rasterize all objects in the scene into sets of pixel fragments. Objects may be processed in any order [S1, unofficial].
2.  **Storage:** Save the generated fragments into per-pixel linked lists. Each fragment must store its depth value [S1, unofficial][S3].
3.  **Sorting:** After all objects have been rasterized, sort the fragments in each per-pixel linked list using the stored depth values [S1, unofficial][S3].
4.  **Compositing:** (Implicit) Composite the sorted fragments from front to back.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global data structure: A list of linked lists, one for each pixel
# pixel_lists[x][y] is a linked list of Fragment objects
# Fragment object has fields: z (depth), color, next (pointer)

def mammen_technique(scene_primitives, image_width, image_height):
    # Initialize per-pixel linked lists
    pixel_lists = initialize_linked_lists(image_width, image_height)
    
    # Step 1 & 2: Rasterize all objects and store fragments
    for primitive in scene_primitives:
        fragments = rasterize(primitive)
        for frag in fragments:
            # Add fragment to the per-pixel list
            # Note: Simple insertion sort during rasterization is possible,
            # but the source describes sorting *after* all objects are rasterized.
            # We will append for now, assuming unsorted lists.
            append_fragment(pixel_lists[frag.x][frag.y], frag)
            
    # Step 3: Sort the lists after all objects have been rasterized
    for x in range(image_width):
        for y in range(image_height):
            head = pixel_lists[x][y]
            if head is not None:
                sort_linked_list_by_depth(head)
                
    return pixel_lists

def sort_linked_list_by_depth(head):
    # Standard linked list sort (e.g., merge sort) based on fragment.z
    # Implementation details omitted as not specified in sources [G]
    pass

def append_fragment(pixel_list, fragment):
    # Append to end of list
    # Implementation details omitted as not specified in sources [G]
    pass
```

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Memory Requirements:** The technique has substantial memory requirements because it stores all incoming fragments for every pixel [S1, unofficial][S3].
*   **Hardware Implementation:** Current hardware implementations of this technique require multiple passes through the polygons in the scene [S2, unofficial][S3].

## Subtopics

### [R-Buffer](2357-r-buffer.md)
The R-Buffer is a hardware architecture proposal for implementing order-independent transparency (OIT) by storing fragments in a large, unordered, recirculating buffer shared across the entire screen, rather than using fixed-size per-pixel buffers. It implements a sorting and compositing method similar to Mammen's software algorithm to handle transparent fragments in front of the front-most opaque one. Unlike previous hardware implementations that required multiple passes through the geometry, the R-Buffer scan-converts all polygons only once, storing uncomposited fragments in the recirculating buffer for subsequent depth-comparison passes. It is distinguished from its sibling [Mammen's Technique](1279-mammen-s-technique.md) by moving the sorting logic from software to dedicated hardware logic, and from [k-Buffer](1280-k-buffer.md) by using a single global FIFO buffer instead of fixed-size per-pixel arrays, thereby amortizing storage costs over the screen.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Substantial, as it stores all incoming fragments for every pixel [S1, unofficial][S3].
*   **Strengths:** Objects can be rasterized in any order, avoiding the complexity of object-space visibility sorting [S1, unofficial].
*   **Limitations:** High memory consumption [S1, unofficial][S3]. Software implementation requires sorting from scratch at each pixel, which is more work than incremental updates [S2, unofficial]. Hardware implementations may require multiple passes through the geometry [S2, unofficial][S3].
*   **When to use:** When object-space ordering is difficult or impossible (e.g., due to visibility cycles) and memory is not a constraint.
*   **When not to use:** When memory is limited or when incremental sorting per pixel is feasible.

## Background
The technique was proposed by Carpenter as the A-buffer [S1, unofficial][S3]. It is described in a software algorithm by Mammen [S2, unofficial]. The method operates on pixel fragments instead of object fragments [S1, unofficial][S3].

## Sources
[S1] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
[S3] PDF Hardware-Assisted Visibility Sorting for Unstructured Volume Rendering, https://www.sci.utah.edu/~stevec/papers/havs.pdf, fetched 2026-10-29
