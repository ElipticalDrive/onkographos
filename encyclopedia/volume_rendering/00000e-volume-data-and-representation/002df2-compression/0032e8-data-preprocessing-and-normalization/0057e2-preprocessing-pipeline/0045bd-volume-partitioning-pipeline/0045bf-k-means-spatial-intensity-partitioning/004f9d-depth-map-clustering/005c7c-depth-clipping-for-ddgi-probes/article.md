# Depth Clipping for DDGI Probes

Also called: Dead Probe Clipping

## Overview
Depth Clipping for DDGI Probes is a compression technique for Deferred Dynamic Global Illumination (DDGI) light probe depth maps that reduces the dynamic range of stored depth values by clipping them to the maximum distance between adjacent probes [S1, unofficial]. This allows probes with no nearby geometry to be identified as 'dead' and compressed efficiently, serving as a preprocessing step before dictionary compression [S1, unofficial]. It sets itself apart from its sibling [Depth Map Clustering](14006-depth-map-clustering.md) by operating on the radial depth range per probe rather than reducing depth layers within a hologram, and from [Dictionary Compression with Look-Back Window](#dictionary-compression-with-look-back-window) by being a lossless range-reduction step that enables subsequent lossy matching [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 16x16 depth map for a single DDGI light probe, containing true depth values.
*   **Input:** `max_dist_from_probe_to_probe` ($N$), the maximum distance possible from one probe to another during interpolation.
*   **Output:** A compressed representation of the depth map. If the probe is "dead," a label and the constant value $N$ are stored. Otherwise, the clipped depth values are stored.

**Data Structures:**
*   **Depth Map:** A 16x16 grid of depth values. The source does not specify the exact bit-depth or floating-point format of the input depth values, only that they are "true depth values" [S1, unofficial].
*   **Probe State:** A boolean or enum flag indicating if the probe is "dead" [S1, unofficial].

**Parameters:**
*   `max_dist_from_probe_to_probe` ($N$): The upper bound for depth clipping. This value is derived from the uniform probe density and ensures that any shading point will have a closer probe available if the current one is too far [S1, unofficial].

**Steps:**
1.  For each texel $(u, v)$ in the 16x16 depth map, retrieve the true depth value $d_{true}(u, v)$.
2.  Calculate the clipped depth value $d_{clip}(u, v)$ using the formula:
    $$ d_{clip}(u, v) = \min(d_{true}(u, v), N) $$
3.  Check if all texels in the block have the clipped depth value equal to $N$. That is, verify if:
    $$ \forall (u, v) \in [0, 15] \times [0, 15]: d_{clip}(u, v) = N $$
4.  If the condition in Step 3 is true, mark the probe as "dead." Store only a label indicating this state. During runtime queries, always return the constant depth value $N$ for all directions [S1, unofficial].
5.  If the condition in Step 3 is false, store the clipped depth map $d_{clip}$ for the probe [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_probe_depth_map(true_depth_map: List[List[float]], max_dist: float) -> dict:
    """
    Compress a 16x16 DDGI probe depth map using depth clipping.
    
    Args:
        true_depth_map: A 16x16 list of lists containing true depth values.
        max_dist: The maximum distance between probes (N).
        
    Returns:
        A dictionary representing the compressed state.
    """
    # Step 1 & 2: Clip depth values
    clipped_map = []
    is_dead = True
    
    for y in range(16):
        row = []
        for x in range(16):
            d_true = true_depth_map[y][x]
            d_clip = min(d_true, max_dist)
            row.append(d_clip)
            
            # Step 3: Check if any texel is not equal to max_dist
            if d_clip != max_dist:
                is_dead = False
        clipped_map.append(row)
        
    # Step 4 & 5: Determine storage format
    if is_dead:
        # Probe is dead: store label and constant value
        return {
            "state": "dead",
            "constant_depth": max_dist
        }
    else:
        # Probe is alive: store clipped depth map
        return {
            "state": "alive",
            "depth_map": clipped_map
        }
```

**Edge Cases and Pitfalls:**
*   **Precision:** The source does not specify floating-point precision issues, but clipping to $N$ reduces the effective dynamic range, which may impact quantization if the depth map is stored in a fixed-point format [S1, unofficial].
*   **Boundaries:** The method assumes a uniform probe density such that a closer probe always exists if the current one is too far [S1, unofficial]. Non-uniform density may break this guarantee.

## Subtopics

### Dictionary Compression with Look-Back Window
A lossy dictionary compression method for DDGI depth map blocks that matches new 16x16 depth blocks against a history of previously processed blocks using a peak pixel difference threshold, storing indices to matches instead of raw data [S1, unofficial]. It is chosen when further compression is needed after depth clipping. It differs from Depth Clipping by being lossy and relying on spatial coherence of probe indices rather than radial depth range reduction [S1, unofficial].
*   **Definition:** Matches new 16x16 depth blocks against a look-back window using peak pixel difference thresholding [S1, unofficial].
*   **Steps:**
    1.  Queue light probe baking requests and process them sequentially, generating a 16x16 depth block per request [S1, unofficial].
    2.  Compare the new depth block against blocks in the look-back window using peak pixel difference thresholding. If the maximum per-texel difference is below 0.01, accept the match [S1, unofficial].
    3.  If a match is found, output the index of the matching block. If no match is found, add the new block to the depth block list and output its new index [S1, unofficial].
*   **Parameters:** Look-back window size: 10,000 blocks [S1, unofficial].
*   **Pitfalls:** The compression is lossy because it accepts matches with non-zero difference (up to 0.01) rather than requiring exact zero difference [S1, unofficial].
*   **Performance:** Final compression ratios: 4.5% of original size with 0.5m probe density and 3m dilation; 9% of original size with 1.0m probe density and 1.5m dilation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The source notes that a single-threaded implementation is slow, but does not provide asymptotic complexity [S1, unofficial].
*   **Memory Complexity:** Reduces storage size by identifying "dead" probes. The source reports a 50% reduction in storage size (from 2.4GB to 1.2GB) using this clipping method alone [S1, unofficial].
*   **Measured Performance:** Achieves 50% compression ratio (1.2GB from 2.4GB) on the tested dataset [S1, unofficial].
*   **Scaling:** Compression ratio improves as probe dilation increases, due to more "dead" probes being generated [S1, unofficial].
*   **Strengths:** Simple to implement; effectively compresses probes with no nearby geometry; enables further compression via dictionary methods [S1, unofficial].
*   **Limitations:** Only effective when probes are uniformly distributed and geometry is sparse enough to create "dead" probes; does not compress non-dead probes [S1, unofficial].
*   **When to use:** When DDGI probes are uniformly distributed and many probes are far from geometry.
*   **When not to use:** When probe density is non-uniform or geometry is dense everywhere, preventing the creation of "dead" probes [S1, unofficial].

## Background
Originated from a blog post by an unnamed author on Handmade Network in 2020, discussing DDGI light probe depth data compression [S1, unofficial]. The method is part of a pipeline that includes Depth Map Clustering and Dictionary Compression, aimed at reducing the memory footprint of DDGI probes in real-time rendering engines [S1, unofficial].

## Sources
[S1] Engine Work: DDGI Light Probe Depth Data Compression | Handmade Network, https://handmade.network/p/75/monter/blog/p/7421-engine_work__ddgi_light_probe_depth_data_compression, fetched 2026-10-28 (unofficial)
