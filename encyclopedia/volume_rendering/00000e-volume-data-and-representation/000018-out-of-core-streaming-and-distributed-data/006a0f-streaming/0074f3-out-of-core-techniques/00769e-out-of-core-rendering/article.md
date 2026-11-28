# Out-of-Core Rendering

Also called: Out-of-Core Processing, Out-of-Core Strategies

## Overview
Out-of-core rendering is a strategy for visualizing volumetric datasets that exceed the capacity of main memory by subdividing the data in space and time into manageable blocks [S1]. It solves the I/O bottleneck inherent in large-scale volume rendering by ensuring that only necessary data blocks are loaded from disk, while simultaneously improving the efficiency of space-leaping techniques by allowing entire blocks to be skipped based on pixel opacity or temporal inactivity [S1]. Unlike in-core methods, it is not interactive and is designed for batch processing of large sequences [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A time-varying volumetric dataset larger than available RAM.
*   **Output:** A sequence of rendered frames composed into a Frame Buffer (FTB).

**Data Structures:**
*   **Octree Hierarchy:** Used only during the preprocessing/creation phase to determine block boundaries. It is not stored or traversed during rendering [S1].
*   **Disk Storage:** Only leaf nodes (blocks) are stored on disk. Files are labeled according to the block's position in the octree to allow sorted traversal [S1].
*   **Memory Buffer:** Holds the current set of active leaf nodes (blocks) required for the current frame or time span [S1].

**Algorithm Steps:**
1.  **Temporal Subdivision:** Subdivide long time sequences into smaller time spans [S1].
2.  **Spatial Subdivision (BON):** Subdivide the model in space using the Branch-On-Need (BON) octree strategy [S1].
3.  **Node Division Criterion:** Divide octree nodes based on memory occupancy. Nodes with higher temporal variation in their voxels are subdivided more deeply [S1].
4.  **Disk Labeling:** Store only the leaf nodes (blocks) on disk. Label each file with the block's octree position to enable sorted access [S1].
5.  **Rendering Loop (Ordered Processing):**
    a. Process leaf nodes in sorted order [S1].
    b. Read the required block from disk into memory [S1].
    c. Render the block.
    d. Compose the resulting subimage into the Frame Buffer (FTB) [S1].

**Rendering Strategies:**
Two algorithms are defined for handling the data flow:
1.  **Simultaneous Algorithm:** Each block of a time span is stored in a single file and read only once for the entire span [S1].
2.  **Frame-to-Frame Algorithm:** Blocks are stored in various files such that only the current voxel codes for the specific frame are read from disk at each frame step [S1].

**Pruning and Optimization:**
*   **Pre-loading Pruning:** Data blocks are pruned before loading into memory by applying ray and temporal coherency checks [S1].
*   **Space-Leaping Integration:** The method improves time and space leaping. A block is skipped if:
    *   Pixels in the block's subimage have already reached maximum opacity [S1].
    *   No pixels in the subimage change between frames (temporal coherency) [S1].
*   **Note:** This strategy is incompatible with the SL2 space-leaping method [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def out_of_core_rendering(dataset, time_spans, octree_blocks):
    """
    Renders a large time-varying volume dataset using out-of-core techniques.
    
    Args:
        dataset: The full volumetric data (used for preprocessing to generate blocks).
        time_spans: List of time span intervals.
        octree_blocks: List of leaf node blocks, labeled by octree position.
        
    Returns:
        frames: List of rendered frame images.
    """
    # Preprocessing: Create disk files for leaf nodes
    # This step is assumed to be completed before rendering starts.
    # Files are labeled by octree position for sorted traversal.
    disk_files = create_disk_files(octree_blocks)
    
    frames = []
    
    for span in time_spans:
        # Determine which blocks are needed for this span
        # Blocks are sorted by octree position
        active_blocks = get_blocks_for_span(span, octree_blocks)
        
        # Initialize Frame Buffer for the current span or frame
        # Depending on algorithm: Simultaneous or Frame-to-Frame
        if algorithm == "simultaneous":
            # Read each block once for the whole span
            for block in active_blocks:
                block_data = read_from_disk(block.file_path)
                render_block(block_data, span, frame_buffer)
        elif algorithm == "frame_to_frame":
            for frame_idx in span.frames:
                # Read only current voxel codes for this frame
                current_frame_buffer = initialize_frame_buffer()
                
                for block in active_blocks:
                    # Check pruning conditions before loading
                    if is_pruned(block, frame_idx, current_frame_buffer):
                        continue
                        
                    block_data = read_from_disk(block.file_path, frame_idx)
                    render_block(block_data, frame_idx, current_frame_buffer)
                
                frames.append(current_frame_buffer)
                
    return frames

def is_pruned(block, frame_idx, frame_buffer):
    """
    Checks if a block can be skipped based on space and temporal leaping.
    """
    # Space leaping: if all pixels in subimage are max opacity
    if is_max_opacity(block.subimage, frame_buffer):
        return True
        
    # Temporal leaping: if no pixels in subimage change
    if is_temporally_static(block, frame_idx, frame_buffer):
        return True
        
    return False
```

**Parameters:**
*   **Time Span Size:** Not specified in sources. Determined by dataset characteristics.
*   **Octree Depth:** Determined by memory occupancy and temporal variation criteria. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Incompatibility:** The out-of-core strategy is incompatible with the SL2 space-leaping method [S1].
*   **Performance Scaling:** Performance ratios improve further in large datasets compared to small datasets due to pre-loading pruning [S1].
*   **Interactivity:** The method is not interactive; it is suitable for batch generation [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of blocks, frames, and the efficiency of pruning. The method reduces computational cost by skipping blocks that do not change or are fully opaque [S1].
*   **Memory Complexity:** Bounded by the size of the largest block plus the frame buffer, allowing datasets larger than main memory to be processed [S1].
*   **Measured Performance:**
    *   Tests were conducted on a Card Nvidia FX5600 [S1].
    *   Ratios of selection ($r_{sel}$), coherency ($r_{coh}$), and occupancy ($r_{occ}$) were measured for various datasets (see Table 1 in [S1]).
    *   For example, on the LRabbit dataset (387x513x640, 16 frames), $r_{sel}=0.86$, $r_{coh}=0.17$, $r_{occ}=0.44$ [S1].
*   **Strengths:** Handles datasets larger than main memory; improves space and time leaping; reduces I/O by reading blocks only when necessary [S1].
*   **Limitations:** Not interactive; incompatible with SL2 space-leaping; requires preprocessing to create octree-based block files [S1].
*   **When to use:** When rendering large, time-varying volumetric datasets that exceed RAM capacity, and interactivity is not required [S1].
*   **When not to use:** When interactive frame rates are needed; when using SL2 space-leaping [S1].

## Background
*   **Origin:** The method is described in [S1] "Ray-casting time-varying volume data sets with frame-to-frame coherence" (2006).
*   **Development:** It was developed to handle multimodal, time-varying datasets where even compressed models exceed main memory [S1]. It builds on the Branch-On-Need (BON) octree strategy [30] and Temporal Run-Length Encoding [S1].
*   **Usage:** Used in batch processing of large scientific visualizations, such as dynamic SPECT/MR brain data and biomaterial implant evolution [S1].

## Sources
[S1] Ray-casting time-varying volume data sets with frame-to-frame coherence, https://doi.org/10.1117/12.649814, fetched 2026-10-26
