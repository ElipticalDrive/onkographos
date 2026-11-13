# Hardware-Assisted Volume Streaming

Also called: HAVS (Hardware-Assisted Volume Streaming), HAVS2, HAVS6

## Overview
Hardware-Assisted Volume Streaming (HAVS) is an irregular volume rendering technique that streams volumetric data to the GPU to perform rendering, addressing the memory limitations of storing entire large-scale volumes in GPU memory [S1, unofficial]. It distinguishes itself from sibling methods like HAPT by performing more accurate visibility sorting, avoiding the artifacts associated with centroid-based sorting, while maintaining a flexible pipeline that does not require multiple rendering passes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A large irregular volume dataset (e.g., tetrahedral meshes) that exceeds available GPU memory.
*   **Output:** A rendered image with correct alpha compositing.

**Data Structures:**
The method relies on streaming primitives (primitives or chunks) to the GPU. Specific internal data structure layouts (e.g., exact byte packing of tetrahedra attributes) are not given in the sources [S1, unofficial]. The method avoids the high memory footprint of storing the whole volume (e.g., 144 bytes/tet for HARCn or 96 bytes/tet for HARCp) [S1, unofficial].

**Steps:**
1.  **Streaming:** The volume is partitioned and streamed to the GPU. This allows rendering datasets that would otherwise require significant data transfer overhead if chunks were uploaded/downloaded repeatedly [S1, unofficial].
2.  **Visibility Sorting:** HAVS performs accurate visibility sorting on the GPU. Unlike HAPT's centroid sort methods (HAPTQ, HAPTB, HAPTS), HAVS ensures correct fragment ordering for non-convex unstructured meshes [S1, unofficial].
3.  **Rendering:** The sorted fragments are rasterized and composited. The implementation offers a flexible pipeline [S1, unofficial].

**Parameters:**
*   **HAVS2/HAVS6:** Variants exist (HAVS2, HAVS6) with different performance characteristics, but specific parameter definitions (e.g., sorting buffer sizes, stream chunk sizes) are not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Limits:** Methods relying on storing the whole volume (like PTINT or HARC) suffer performance loss due to uploading/downloading animation frame chunks [S1, unofficial]. HAVS mitigates this via streaming.
*   **Sorting Accuracy:** Inaccurate sorting (as in centroid sorts) can lead to visual artifacts in complex topologies [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# HAVS Implementation Logic
# Note: Specific sorting network details and streaming chunk management
# are not provided in the source text. The following represents the
# high-level flow described.

def render_havs(volume_data, camera):
    # 1. Stream volume data to GPU
    # The volume is partitioned and streamed to avoid memory limits.
    gpu_stream = stream_to_gpu(volume_data)
    
    # 2. Perform accurate visibility sorting
    # HAVS uses hardware-assisted sorting, more accurate than centroid sorts.
    # The specific sorting mechanism (e.g., k-buffer details) is not detailed
    # in the provided text, but it handles fragment ordering correctly.
    sorted_primitives = hardware_assisted_sort(gpu_stream)
    
    # 3. Rasterize and Composite
    # Render the sorted primitives to the frame buffer.
    image = rasterize_and_composite(sorted_primitives, camera)
    
    return image
```

## Subtopics

### [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md)
HAVS is a hybrid volume rendering algorithm for unstructured grids that combines object-space sorting on the CPU with image-space fragment sorting on the GPU using a k-buffer. It is chosen when robust visibility handling for non-convex meshes is required, offering a compromise between complexity and quality, though it may produce artifacts if fragments are highly out of order.

## Operating characteristics
*   **Performance:** On a torso dataset, HAVS2 achieves 3.7 FPS and 4.01 M Tet/s. On a fighter dataset, HAVS2 achieves 2.9 FPS and 4.12 M Tet/s [S1, unofficial].
*   **Comparison:** HAVS methods gain approximately 50% in frame rates compared to HAPTQ x HAVS6 configurations and do not require multiple rendering passes [S1, unofficial].
*   **Memory:** HAVS avoids the high memory footprint of methods like HARCn (144 bytes/tet) and HARCp (96 bytes/tet) [S1, unofficial]. It is more efficient than PTINT for large datasets (e.g., turbulent jet sequence) which would require 3 GB of GPU memory to fit entirely [S1, unofficial].
*   **Strengths:** Accurate visibility sorting; flexible pipeline; no multiple passes; handles large volumes via streaming.
*   **Limitations:** Performance is lower than some other methods (e.g., HAPTS 3.9 FPS on torso) but offers better sorting accuracy [S1, unofficial].

## Background
HAVS is an irregular volume renderer that streams volume data. It is one of the most popular irregular volume renderers due to its accurate visibility sorting compared to centroid sort methods [S1, unofficial]. It is related to the HAPT (Hardware-Assisted Projected Tetrahedra) family of algorithms [S1, unofficial].

## Sources
[S1] (PDF) Hardware-Assisted Projected Tetrahedra - Academia.edu, https://www.academia.edu/16536671/Hardware_Assisted_Projected_Tetrahedra, fetched 2026-10-28 (unofficial)
