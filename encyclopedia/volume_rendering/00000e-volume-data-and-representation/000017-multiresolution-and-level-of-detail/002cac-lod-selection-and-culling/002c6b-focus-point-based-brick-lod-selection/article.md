# Focus Point Based Brick LOD Selection

Also called: Adaptive level-of-detail, Focus Point Based LOD Selection, Focus point based LOD, Focus point oracle, Focus-point based adaptive level-of-detail

## Overview
Focus Point Based Brick LOD Selection is a method to determine the appropriate level-of-detail (LOD) for each brick in a multiscale volume representation based on the distance of the brick's center to a user-defined focus point [S1, S2]. It differs from error-driven siblings like [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) because it relies exclusively on geometric proximity to the focus point, ignoring approximation error [S1]. This allows for arbitrary texture resolutions per brick, unlike methods with strict texture-to-geometry correspondence [S1, S2].

## How it works
**Inputs:**
- A multiscale volume representation composed of bricks.
- A user-defined focus point $P_f$ in world space.
- The initial brick size specification (determined before hierarchy construction) [S2].

**Outputs:**
- An assigned LOD level for each brick.
- Adapted brick boundaries if level transitions occur [S2].

**Data Structures:**
- **Brick:** Defined by its spatial extent and a resolution level $L$. The initial size is fixed prior to construction [S2].
- **Focus Point Oracle:** A function mapping a brick's center position to an LOD level based on distance to $P_f$ [S1, S2].

**Steps:**
1. **Initialization:** Specify the initial size of each brick in advance before constructing the multiscale volume representation [S2].
2. **LOD Assignment:** For each brick, calculate the distance $d$ between its center $C_b$ and the focus point $P_f$ [S1, S2].
3. **Oracle Query:** Use the focus point oracle to determine the appropriate LOD level $L$ based on $d$ [S1, S2]. Closer bricks typically receive higher resolution levels [S1].
4. **Boundary Adaptation:** If the hierarchy changes (e.g., focus point moves) or if a brick is adjacent to a coarser one, adapt the brick boundaries to ensure continuous level transitions [S2].
   - In 3D, up to 26 neighboring bricks may need consideration to handle these transitions [S2].
   - This adaptation is required whenever the hierarchy is changed by moving the focus point [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_lod(bricks, focus_point):
    """
    Assigns LOD levels to bricks based on distance to focus_point.
    bricks: List of Brick objects, each with center C_b and initial_size.
    focus_point: 3D vector (x, y, z).
    """
    for brick in bricks:
        # Calculate distance from brick center to focus point
        distance = vector_length(brick.C_b - focus_point)
        
        # Determine LOD level using the focus point oracle
        # The oracle maps distance to an integer LOD level.
        # Implementation detail: The specific mapping function is not defined in sources.
        # [G] Typical implementation: L = floor(log2(max_resolution / distance)) or similar monotonic function.
        lod_level = focus_point_oracle(distance)
        
        brick.LOD = lod_level

def adapt_boundaries(bricks):
    """
    Adapts brick boundaries to handle level transitions.
    bricks: List of Brick objects with assigned LOD levels.
    """
    # Identify bricks adjacent to coarser ones
    # In 3D, up to 26 neighbors must be checked [S2]
    for brick in bricks:
        neighbors = get_26_neighbors(brick, bricks)
        for neighbor in neighbors:
            if neighbor.LOD < brick.LOD:
                # Adapt boundaries to ensure continuous transition
                # [G] Specific geometric adjustment algorithm is not detailed in sources.
                # [G] Typically involves shifting the shared face position.
                adjust_shared_boundary(brick, neighbor)
```

**Parameters:**
- **Focus Point:** User-defined. No typical values given.
- **Initial Brick Size:** Specified in advance. No typical values given.

**Edge Cases and Pitfalls:**
- **Approximation Error:** The method does not consider approximation error introduced by sampling data at a certain resolution level [S1, S2].
- **High Frequencies:** If high frequencies are present in the data, the approximation may differ significantly from the original signal, causing artifacts between brick boundaries even if continuity is guaranteed [S1, S2].
- **Non-Topology Preserving:** Structures may disappear due to low-pass filtering during coarsification, leading to visual artifacts at brick boundaries if the resampled approximation differs significantly from the original data [S1].
- **Level Transitions:** Level transitions greater than one are not supported in the basic implementation described [S1, S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** In a head data set example, using this approach reduced texture memory usage to 64% (middle image) and 37% (right image) of the original, and reduced texture lookups from 69% to 40% [S1].
- **Strengths:** Allows arbitrary texture resolution for every brick, which is not achievable with techniques requiring strict correspondence between texture tiles and brick geometry [S1, S2].
- **Limitations:** Does not consider approximation error, leading to potential loss of relevant structures and visual artifacts at brick boundaries [S1, S2]. Not topology-preserving [S1].

## Background
- **Origin:** Introduced by Weiler et al. in "Level-Of-Detail Volume Rendering via 3D Textures" (2000) [S1, S2].
- **History:** Part of the development of 3D texture-based volume rendering, extending basic trilinear interpolation and compositing techniques to support multiresolution representations [S2].
- **Usage:** Used in volume rendering systems where user focus is a primary driver for detail allocation, such as interactive visualization tools [S1, S2].

## Sources
[S1] Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
[S2] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
