# Space-Leaping 2 (SL2)

Also called: BF-SL2, Coh-SL2, Space-leaping SL2 (SL2), temporal ray sampling intervals

## Overview
Space-Leaping 2 (SL2) is a view-dependent preprocessing strategy for time-varying volume rendering that computes the sampling interval for each pixel as the range between the first and last samples along the ray that possess a relevant value across the entire temporal sequence [S1]. It distinguishes itself from its sibling [Space Leaping](5863-space-leaping.md) by skipping not only empty voxels but also non-selected (irrelevant) voxels, and from [Boundary Pixel Detector Ray Casting](5870-boundary-pixel-detector-ray-casting.md) by relying on temporal coherence rather than spatial boundary detection. While it incurs a higher preprocessing cost than [Hierarchical Box Space Leaping](16042-hierarchical-box-space-leaping.md), it provides more efficient space-leaping for successive frames [S1].

## How it works
**Inputs:**
- A time-varying volume dataset encoded as Temporal Run-Lengths (TRL) [S1].
- A sequence of frames $F_1, \dots, F_N$.
- A camera viewpoint.

**Outputs:**
- A set of sampling intervals $[t_{start}, t_{end}]$ for each pixel in the T-Buffer, defining the range of samples along the ray that contain relevant data [S1].

**Data Structures:**
- **T-Buffer:** Stores the temporal run-lengths for each pixel. The layout corresponds to the TRL encoding of the volume [S1].
- **Sampling Interval Map:** For each pixel, stores the minimum and maximum sample indices (or spatial positions) along the ray that are relevant in any frame of the sequence [S1].

**Steps:**
1. **Ray Traversal:** For each pixel in the image, traverse the ray through the volume data [S1].
2. **Identify Relevant Samples:** Scan all sample codes for the ray across all frames in the sequence to identify the first sample index $i_{min}$ and the last sample index $i_{max}$ that have a relevant value in at least one frame [S1].
   - A sample is "relevant" if it contributes to the final image (e.g., non-background, selected modality) [S1].
3. **Skip Voxels:** During the traversal, skip voxels that are empty or non-selected [S1]. This is possible because the preprocessing knows the temporal extent of relevance [S1].
4. **Compute Interval:** Set the sampling interval for the pixel to $[i_{min}, i_{max}]$ [S1].
5. **Multimodal Handling:** If the dataset is multimodal, compute the sampling intervals separately for each modality. During integration, consult only the TRLs of the modalities corresponding to the current interval [S1].
6. **Apply to Rendering:** Use the computed intervals to limit the ray integration range for all frames in the sequence [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sl2_intervals(volume_data, camera, frames):
    """
    Computes sampling intervals for SL2.
    volume_data: Structure containing TRLs for all frames.
    camera: Camera object with ray generation capability.
    frames: List of frame indices [0, 1, ..., N-1].
    Returns: Dictionary mapping pixel_id to (start_index, end_index).
    """
    intervals = {}
    
    for pixel_id in all_pixels():
        ray = camera.generate_ray(pixel_id)
        
        # Initialize bounds to infinity/null
        min_idx = infinity
        max_idx = -infinity
        
        # Traverse the ray through the volume
        # Note: In TRL encoding, traversal may jump through runs.
        # We must check every run that intersects the ray.
        for run in ray_intersections(volume_data, ray):
            # Check if this run contains any relevant value in any frame
            is_relevant = False
            for frame_idx in frames:
                # Check if the run's value in this frame is relevant
                # Relevance depends on the specific application (e.g., opacity > threshold)
                if is_relevant_value(volume_data.get_value(run, frame_idx)):
                    is_relevant = True
                    break
            
            if is_relevant:
                # Update bounds based on the spatial extent of the run along the ray
                run_start_idx = run.start_sample_index
                run_end_idx = run.end_sample_index
                
                if run_start_idx < min_idx:
                    min_idx = run_start_idx
                if run_end_idx > max_idx:
                    max_idx = run_end_idx
        
        # If no relevant samples were found, the pixel is empty
        if min_idx == infinity:
            intervals[pixel_id] = None
        else:
            intervals[pixel_id] = (min_idx, max_idx)
            
    return intervals
```

**Parameters:**
- **Preprocessing Cost:** Between one-third and one-half the cost of rendering one frame in the sequential method [S1].
- **Image Size:** Affects preprocessing cost, but has no visible influence on the overall rendering cost [S1].

**Edge Cases and Pitfalls:**
- **Viewpoint Change:** Requires recomputation of all intervals, resulting in higher preprocessing cost [S1].
- **Multimodal Data:** Intervals must be computed separately for each modality to ensure correct integration [S1].

## Subtopics

### BF-SL2
A sequential variant of SL2 using the BF (likely "Basic Frame" or similar) rendering strategy. It outperforms BF-SL1 by providing more efficient space-leaping for successive frames [S1].

### Coh-SL2
A coherent variant of SL2. It outperforms Coh-SL1 by leveraging frame-to-frame coherence in addition to the temporal ray sampling intervals [S1].

## Operating characteristics
- **Time Complexity:** Preprocessing is $O(P \cdot R \cdot F)$, where $P$ is pixels, $R$ is ray length (samples), and $F$ is frames. Rendering is $O(P \cdot L)$, where $L$ is the average length of the computed intervals [S1].
- **Memory Complexity:** Requires storage for the T-Buffer and the interval map [S1].
- **Performance:** In datasets with significant empty space, SL2 produces considerable time savings even for a low number of frames [S1]. BF-SL2 outperforms BF-SL1, and Coh-SL2 outperforms Coh-SL1 [S1].
- **Strengths:** More efficient space-leaping than SL1 variants; skips non-selected voxels [S1].
- **Limitations:** Higher preprocessing cost; requires recomputation for viewpoint changes [S1].
- **When to Use:** Time-varying volumes with significant empty or non-selected regions; static viewpoints or infrequent viewpoint changes [S1].
- **When Not to Use:** Dynamic viewpoints requiring frequent interval recomputation; datasets with little empty space where preprocessing cost outweighs rendering savings [S1].

## Background
SL2 was introduced in the context of ray-casting time-varying volume data sets with frame-to-frame coherence [S1]. It builds on the Temporal Run-Length (TRL) encoding, which compresses the voxel model while keeping spatial order and allowing fast prediction of changes along time [S1]. The method is part of a family of three space-leaping strategies designed to exploit temporal coherence in volume animation rendering [S1].

## Sources
[S1] Ray-casting time-varying volume data sets with frame-to-frame coherence, https://doi.org/10.1117/12.649814, fetched 2026-10-26
