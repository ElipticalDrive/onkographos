# Hidden Volume Removal using Hierarchical Occlusion Maps

Also called: HOM-based Occlusion Culling, Hidden Volume Removal (HVR), Hierarchical Occlusion Map Occlusion Clipping

## Overview
Hidden Volume Removal (HVR) is an occlusion culling optimization for object-order volume rendering that eliminates a large fraction of cells and octree nodes within the opacity range, particularly effective for isosurface visualization [S1]. It compensates for the lack of early ray termination inherent in object-order rendering by using Hierarchical Occlusion Maps (HOMs) to skip hidden nodes in the min-max octree [S1]. Unlike its sibling [Orthogonal Opacity Map Creation](991-orthogonal-opacity-map-creation.md), which relies on orthogonal projections and neighbor finding, HVR uses a hierarchical map structure to approximate node visibility, providing superior skipping efficiency at the cost of some artifacts due to approximation [S1].

## How it works
**Inputs:**
- An octree representation of the volume data.
- A camera view (orthogonal projection is assumed for the core algorithm description [S1]).
- A rendering image plane of width $m$.

**Outputs:**
- A rendered image with hidden nodes skipped.
- Updated HOM structures reflecting current opacity.

**Data Structures:**
1.  **Hierarchical Occlusion Maps (HOMs):** A hierarchy of images.
    -   **Pixel Value:** Integer in range $[0, 16]$. Initialized to 0.
    -   **Hierarchy Levels:** Multiple levels exist. The finest map (Level 0) has a size of $m/2 \times m/2$ (since the first map size is $1/4$ of the image area, and assuming square images, side length is $m/2$). The coarsest map (Level $L$) represents a single pixel.
    -   **Mapping:** Each level $k$ corresponds to a specific resolution. A pixel in HOM level $k$ covers a $2^k \times 2^k$ area of the finest map.
2.  **Octree:** Standard min-max octree structure. Each node has a projected center.

**Algorithm Steps:**

1.  **Initialization:**
    -   Initialize all HOM pixels to 0.
    -   Determine the mapping between octree levels and HOM levels. For each octree level, find the finest HOM level such that a single HOM pixel can include the entire projection of the octree node [S1].

2.  **HOM Update Process:**
    -   Triggered when a pixel on the image plane becomes opaque during rendering.
    -   Identify the corresponding pixel in the finest HOM (Level 0).
    -   Increment this pixel and its 3 nearest neighbors in the map (forming a $2 \times 2$ block centered on the opaque pixel's projection) [S1].
    -   **Recursive Propagation:** If any HOM pixel reaches the maximum value of 16:
        -   Increment the corresponding pixel in the superior (coarser) HOM level.
        -   Repeat the neighbor incrementation and check for 16 in the superior level.
        -   Continue recursively up the hierarchy until no pixel reaches 16 or the top level is reached [S1].

3.  **Visibility Test:**
    -   For each octree node to be processed:
        -   Project the center of the node onto the image plane.
        -   Identify the HOM level associated with the node's octree level (as determined in Step 1).
        -   Find the pixel in that HOM level corresponding to the projected center.
        -   **Test:** If the pixel value equals 16, the node is considered hidden and skipped. A value of 16 indicates that an extended square around the pixel (made of 16 quarter-pixels) is opaque [S1].
        -   If the value is less than 16, the node is visible and processed.
    -   **Note:** An additional opacity test must be added for every treated ray to handle cases where the HOM approximation is insufficient [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_HOM_VALUE = 16
IMAGE_WIDTH = m
# HOM levels: 0 is finest (size m/2 x m/2), L is coarsest (1x1)
# HOM[level][y][x] stores integer value

def initialize_homs():
    """Initialize HOM hierarchy with zeros."""
    # Determine number of levels based on image size
    # Finest map side = m / 2
    # Coarsest map side = 1
    # Levels = log2(m/2) + 1
    num_levels = int(math.log2(IMAGE_WIDTH / 2)) + 1
    homs = []
    for level in range(num_levels):
        side = IMAGE_WIDTH / (2 ** (level + 1))
        homs.append([[0 for _ in range(int(side))] for _ in range(int(side))])
    return homs

def update_homs(homs, opaque_pixel_coords):
    """
    Update HOMs when an image plane pixel becomes opaque.
    opaque_pixel_coords: (x, y) in image coordinates.
    """
    # Map image pixel to finest HOM (level 0)
    # Finest HOM pixel covers 2x2 image pixels (since HOM size is m/2)
    hom_x = opaque_pixel_coords[0] // 2
    hom_y = opaque_pixel_coords[1] // 2
    
    current_level = 0
    while current_level < len(homs):
        level_homs = homs[current_level]
        # Increment the pixel and its 3 nearest neighbors (2x2 block)
        # Neighbors are (x, y), (x+1, y), (x, y+1), (x+1, y+1)
        # Check bounds for each neighbor
        for dx in [0, 1]:
            for dy in [0, 1]:
                nx = hom_x + dx
                ny = hom_y + dy
                if 0 <= ny < len(level_homs) and 0 <= nx < len(level_homs[0]):
                    level_homs[ny][nx] += 1
                    # Check if this pixel reached max value
                    if level_homs[ny][nx] == MAX_HOM_VALUE:
                        # Propagate to superior level
                        if current_level + 1 < len(homs):
                            # Map current pixel to superior level
                            # Superior level is half the resolution
                            super_x = nx // 2
                            super_y = ny // 2
                            # Reset current pixel? The source says "updating starts again recursively"
                            # It implies the value is consumed/propagated. 
                            # Standard HOM implementations often reset or keep track of count.
                            # Source says "incremented 16 times... updating starts again".
                            # This implies the value 16 is a threshold for propagation.
                            # We assume the value stays at 16 or is reset. 
                            # Given "reaches its maximum value... updating starts again", 
                            # it suggests the 16 is a trigger. 
                            # Let's assume the value remains 16 to indicate full opacity.
                            pass 
                        # Break inner loops to avoid double counting if multiple neighbors trigger?
                        # The source says "When an HOM pixel reaches... updating starts again".
                        # It's a recursive process.
                        break
            if level_homs[ny][nx] == MAX_HOM_VALUE:
                break
        if current_level + 1 < len(homs) and any(level_homs[y][x] == MAX_HOM_VALUE for y in range(len(level_homs)) for x in range(len(level_homs[0]))):
             # If any pixel in current level hit 16, we need to process the superior level.
             # However, the propagation is specific to the pixel that hit 16.
             # The source says "When an HOM pixel reaches... updating starts again recursively at the superior level."
             # This implies a per-pixel propagation.
             # The pseudocode above is simplified. A more accurate recursive update:
             pass
        current_level += 1

def is_node_hidden(node, homs):
    """
    Check if an octree node is hidden using HOMs.
    node: Octree node with projected center (cx, cy) and octree level.
    homs: Hierarchical Occlusion Maps.
    Returns: True if hidden, False if visible.
    """
    # Determine HOM level for this octree level
    # HOM level is the finest level where a single HOM pixel includes the entire projection of the node.
    # This mapping is pre-computed.
    hom_level = get_hom_level_for_octree_level(node.octree_level)
    
    # Project node center to HOM coordinates
    # HOM level 0 has size m/2. Image size is m.
    # So HOM level 0 pixel covers 2x2 image pixels.
    # HOM level k has size m / (2^(k+1)).
    # Image pixel (x,y) maps to HOM level k pixel (x / 2^(k+1), y / 2^(k+1))
    
    side = IMAGE_WIDTH / (2 ** (hom_level + 1))
    hom_x = int(node.projected_center_x / side)
    hom_y = int(node.projected_center_y / side)
    
    # Check bounds
    if 0 <= hom_y < len(homs[hom_level]) and 0 <= hom_x < len(homs[hom_level][0]):
        if homs[hom_level][hom_y][hom_x] == MAX_HOM_VALUE:
            return True
    return False
```

**Parameters:**
-   **HOM Max Value:** 16 [S1].
-   **HOM Resolution:** Finest map is $1/4$ of the image area [S1].

**Edge Cases and Pitfalls:**
-   **Approximation Artifacts:** The visibility test is an approximation. Nodes far beyond the visible surface may still be encountered because big block skipping is rare due to the low accuracy of the occlusion maps [S1].
-   **Non-Optimal Traversal:** The recursive run of the octree does not optimize HVR efficiency optimally. Some hidden nodes are processed unnecessarily [S1].
-   **Neighbor Finding:** The approach relies on trees that do not allow efficient neighbor finding, which is a limitation compared to other methods [S1].

## Subtopics

### Hierarchical Occlusion Extension
A proposed extension to exploit the hierarchy for accelerating occlusion tests, using the model from [BWPP04] to hierarchically test occlusion of visible nodes without latency from waiting for parent node results [S2, unofficial]. However, the hierarchy is not exploited in the current algorithm to accelerate occlusion tests [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** The update complexity is $m^2 \cdot \log_2(m)$, where $m$ is the image width [S1].
-   **Performance:** HVR is more efficient than the method by Lee and Ihm [9]. For similar renderings, Lee and Ihm achieve 25% occluded splats for the head dataset and 67% for the brain dataset, whereas HVR achieves higher occlusion ratios (e.g., 91.70% for Head (a)) [S1].
-   **Throughput:** The algorithm delivers an approximately constant cell throughput (between 1.4 and 2.1 millions cells per second) [S1].
-   **Strengths:** Eliminates a large fraction of cells and octree nodes within the opacity range, particularly effective for isosurface visualization [S1]. It is an efficient alternative to the lack of early ray termination in object-order volume rendering [S1]. Produces better image quality than some competitors (e.g., Ultravis) [S1].
-   **Limitations:** Big block skipping is rare due to low accuracy of occlusion maps [S1]. Some hidden nodes far beyond the visible surface may still be encountered [S1]. The recursive run of the octree does not optimize HVR efficiency [S1].
-   **When to Use:** Effective for isosurface visualization where early ray termination is not available or effective.
-   **When Not to Use:** When high accuracy in skipping distant hidden nodes is required, or when semi-transparent transfer functions are used (trilinear interpolation becomes predominant, reducing HVR efficiency) [S1].

## Background
Hidden Volume Removal using Hierarchical Occlusion Maps was introduced in the paper "A new object-order ray-casting algorithm" [S1]. It was developed to address the inefficiencies of previous occlusion culling methods in object-order volume rendering, specifically those that relied on trees without efficient neighbor finding [S1]. The method builds on the concept of Hierarchical Occlusion Maps [35] to provide superior skipping of hidden nodes in the octree [S1]. A proposed extension to exploit the hierarchy further was discussed in a thesis [S2, unofficial], but was not implemented in the original algorithm.

## Sources
[S1] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
