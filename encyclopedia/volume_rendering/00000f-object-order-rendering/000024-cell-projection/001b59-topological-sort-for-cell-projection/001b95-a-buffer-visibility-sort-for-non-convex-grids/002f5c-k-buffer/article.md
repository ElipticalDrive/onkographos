# k-buffer

Also called: fixed depth A-buffer

## Overview
The k-buffer is a fixed-depth A-buffer used for sorting fragments in image space, introduced to shift the visibility sorting burden from the CPU to the GPU [S1, unofficial]. It serves as a fragment stream sorter that stores a fixed number of fragments per pixel, reducing memory requirements compared to the original A-buffer by combining and discarding fragments as new ones arrive [S2, unofficial][S3, unofficial]. It is a generalized Z-Buffer allowing a limited number of fragments per pixel [S4, unofficial].

## How it works
The k-buffer operates as part of a two-phase visibility sorting algorithm. The first phase performs a partial visibility ordering of primitives in object-space using the CPU, which does not guarantee an exact visibility order during rasterization [S1, unofficial]. The second phase uses the k-buffer to sort fragments in exact order on the GPU [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A stream of fragments generated during rasterization. Each fragment has a distance from the viewpoint.
*   **Output:** A sorted list of up to $k$ fragments per pixel, ordered for compositing.

**Data Structures:**
For each pixel, the k-buffer stores a constant $k$ number of entries.
*   **Layout:** A fixed-size array or list of size $k$.
*   **Fields:** Each entry contains the distance of the fragment from the viewpoint.
*   **Representation:** $\langle f_1, f_2, \dots, f_k \rangle$ [S2, unofficial][S3, unofficial].

**Algorithm Steps:**
1.  **Initialization:** For each pixel, initialize the k-buffer with $k$ empty slots or a sentinel value representing infinite distance (for front-to-back) or zero distance (for back-to-front).
2.  **Fragment Arrival:** When a new fragment $f_{new}$ arrives for a pixel:
    *   Determine the sort order based on the compositing strategy:
        *   **Front-to-back:** Sort in increasing order of distance from the viewpoint.
        *   **Back-to-front:** Sort in decreasing order of distance from the viewpoint.
    *   Insert $f_{new}$ into the k-buffer maintaining the sorted order.
    *   **Dislodging:** If the buffer is full (contains $k$ fragments), remove the fragment that is least relevant to the current compositing pass.
        *   For **front-to-back** compositing, the new fragment $f_{new}$ dislodges the first entry ($f_1$), which is the fragment closest to the viewer (smallest distance) [S2, unofficial][S3, unofficial].
        *   For **back-to-front** compositing, the new fragment would dislodge the fragment furthest from the viewer (largest distance), though the sources specifically detail the front-to-back dislodging mechanism [S2, unofficial].
    *   **Boundary Cases:** Handle cases where $f_{new}$ is inserted at the beginning (closer than all others) or at the end (further than all others) [S2, unofficial][S3, unofficial].
3.  **Composition:** After all fragments for a pixel have been processed, the remaining $k$ fragments in the buffer are composited in the correct order [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming front-to-back compositing (increasing distance order)
# Buffer stores distances: [d1, d2, ..., dk] where d1 < d2 < ... < dk

def insert_fragment(buffer, k, new_distance):
    """
    Inserts a new fragment into the k-buffer for a specific pixel.
    Maintains sorted order (increasing distance).
    Dislodges the closest fragment (index 0) if buffer is full.
    """
    # If buffer is empty or has fewer than k entries
    if len(buffer) < k:
        # Insert new_distance in sorted position
        insert_sorted(buffer, new_distance)
    else:
        # Buffer is full. Check if new fragment is closer than the furthest stored fragment.
        # In front-to-back, we keep the k closest fragments.
        # The furthest stored fragment is at index k-1.
        if new_distance < buffer[k-1]:
            # Remove the furthest fragment (last element)
            buffer.pop()
            # Insert new fragment in sorted position
            insert_sorted(buffer, new_distance)
        # Else: new fragment is further than all stored, discard it.

def insert_sorted(buffer, value):
    """
    Inserts value into buffer maintaining ascending order.
    """
    # Find insertion point
    i = 0
    while i < len(buffer) and buffer[i] < value:
        i += 1
    buffer.insert(i, value)

def composite_pixel(buffer):
    """
    Composites the fragments in the buffer from closest to furthest.
    """
    result = empty_color
    for distance in buffer:
        # Retrieve fragment color/opacity at this distance
        # Composite result with fragment
        result = composite(result, fragment)
    return result
```

**Parameters:**
*   $k$: The maximum number of fragments per pixel.
    *   Classic Z-buffer uses $k=1$ [S4, unofficial].
    *   Z3 buffer uses $k=3$ [S4, unofficial].
    *   Typical values depend on the desired visual quality and memory constraints [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Cases:** Proper handling is required when $f_{new}$ is inserted at the beginning (closer than all others) or at the end (further than all others) [S2, unofficial][S3, unofficial].
*   **Memory Efficiency:** The fixed size prevents unbounded memory growth associated with the original A-buffer [S2, unofficial][S3, unofficial].
*   **Sorting Assumption:** The k-buffer correctly outputs fragments in the correct order only if the input sequence is "k-nearly sorted," meaning each fragment is within $k$ positions from its position in the fully sorted order [S2, unofficial][S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Given a sequence of fragments where each fragment is within $k$ positions from its sorted position, the k-buffer outputs fragments in the correct order in $O(n)$ time for $n$ fragments [S2, unofficial][S3, unofficial].
*   **Memory Complexity:** $O(k \cdot N)$ where $N$ is the number of pixels. This is significantly less than the original A-buffer which requires potentially unbounded memory per pixel [S2, unofficial][S3, unofficial].
*   **Strengths:** Simple implementation on existing graphics architectures; reduces memory requirements; shifts sorting burden to GPU [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Limitations:** Requires partial pre-sorting on the CPU to ensure the "k-nearly sorted" property; limited number of fragments may lead to visibility errors if $k$ is too small for the scene complexity [S1, unofficial][S4, unofficial].
*   **When to use:** When rendering unstructured volumetric data with dynamic LOD, and when GPU-based sorting is preferred over CPU-based exact sorting [S1, unofficial].

## Background
The k-buffer was introduced as a contribution to shift sorting burden to the GPU, building on previous work by Farias et al., Carpenter, and Danskin and Hanrahan [S1, unofficial]. It is a key component of the Hardware Assisted Visibility Sorting (HAVS) algorithm for rendering unstructured volumetric data [S2, unofficial][S3, unofficial]. The concept is related to the Z3 buffer by Joupi et al. [S4, unofficial].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S3] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S4] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
