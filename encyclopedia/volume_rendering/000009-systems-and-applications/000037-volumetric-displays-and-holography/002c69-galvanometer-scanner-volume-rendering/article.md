# Galvanometer Scanner Volume Rendering

Also called: DSS color volumetric display rendering, Drawing of multicolor volumetric graphics, LC Color Filter Volume Rendering, LC filter synchronized voxel generation, Miyazaki Galvano Scanner System, Multicolor Volumetric Graphics, Point Cloud Scanner Volume Rendering, Temporal color-synchronized volumetric rendering

## Overview
Galvanometer Scanner Volume Rendering is a method for rendering volumetric images with arbitrary per-voxel colors by temporally synchronizing the spatial generation of voxels with the switching of a Liquid Crystal (LC) color filter [S1, S2]. It solves the problem of multicolor volumetric display by generating application orders of 3D coordinates and color patterns for a beam scanner and an LC color filter, respectively [S2]. Unlike [Rotational Swept-Screen Projection](8589-rotational-swept-screen-projection.md) which relies on persistence of vision in a rotating medium, or [Drawing Space Separation (DSS)](5290-drawing-space-separation-dss.md) which focuses on the optical path separation, this method specifically addresses color expression by mapping voxel generation positions to color switching states [S1]. It builds on [Drawing Space Separation (DSS)](5290-drawing-space-separation-dss.md) by adding a temporal color-synchronization layer [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D point cloud $V$ consisting of $M$ voxels. Each voxel $v_i$ is defined by spatial position $(x_i, y_i, z_i)$ and RGB color values $(r_i, g_i, b_i)$ [S1, S2].
*   **Output:** A sequence of application orders for the display system, yielding $(x, y)$ values for the galvanometer scanner, $z$ values for the varifocal lens, and $(r, g, b)$ values for the LC color filter [S2].

**Data Structures**
*   **Voxel Cloud ($V$):** A list or array of $M$ objects. Each object contains fields: `x`, `y`, `z`, `r`, `g`, `b`.
*   **Layer ($P$):** A subset of $V$ containing $N$ voxels that share the same quantized $z$-coordinate. Each object contains fields: `x`, `y`, `r`, `g`, `b`.
*   **Scan Path:** A list of indices or voxel references defining the traversal order for each layer.

**Parameters**
*   **LC Color Filter Refresh Rate:** 60 Hz [S1].
*   **Laser Repetition Frequency:** 1 kHz [S1].
*   **Color Extraction Interval:** Different colors are extracted every 17 pulses given the above rates [S1].
*   **Quantization Layers:** The number of discrete $z$-layers. The sources provide an example of 30 layers [S2]. The exact number is not fixed but depends on the desired depth resolution [S2].

**Steps**
1.  **Represent Point Cloud:** Initialize $V = \{v_1, v_2, \ldots, v_M\}$ where $v_i = (x_i, y_i, z_i, r_i, g_i, b_i)$ [S1, S2].
2.  **Quantize Z-Axis:** Quantize the $z$ values of $V$ into a desired number of layers. The $z$ axis must match the optical axis direction of the display system [S2].
3.  **Slice Layers:** For each quantized $z$-layer, define the set of vertices $P = \{p_1, p_2, \ldots, p_N\}$, where each $p_j$ has 2D position $(x_j, y_j)$ and RGB values $(r_j, g_j, b_j)$ [S2].
4.  **Calculate Optimal Scanning Order:** For each layer $P$, calculate the optimal scanning order that minimizes the total route length. The route length is defined as:
    $$ \sum_{j=1}^{N-1} d(p_j, p_{j+1}) + d(p_N, p_1) $$
    where $d(p_j, p_{j+1})$ is the Euclidean distance between the 2D coordinates of vertices $p_j$ and $p_{j+1}$ [S2]. The Christofides algorithm is applied to all layers to solve this Traveling Salesman Problem (TSP) [S1, S2].
5.  **Generate Application Orders:** Provide the shortest scanning order obtained for each $z$ to the display system. This yields the $(x, y)$ values for the galvanometer scanner, $z$ values for the varifocal lens, and $(r, g, b)$ values for the LC color filter [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def euclidean_distance_2d(p1, p2):
    """Calculate Euclidean distance between two 2D points."""
    return math.sqrt((p1['x'] - p2['x'])**2 + (p1['y'] - p2['y'])**2)

def christofides_algorithm(points):
    """
    Placeholder for Christofides algorithm implementation.
    Returns a list of indices representing the optimal tour order.
    Note: A full implementation requires graph construction, minimum spanning tree,
    minimum weight perfect matching, and Eulerian tour conversion.
    """
    # [G] Standard Christofides implementation details omitted for brevity.
    # In a real implementation, this function would return the permutation
    # of indices that minimizes the total path length.
    pass

def render_galvanometer_volumetric(voxel_cloud):
    """
    Renders volumetric images with arbitrary per-voxel colors.
    
    Args:
        voxel_cloud (list): List of dicts, each with keys 'x', 'y', 'z', 'r', 'g', 'b'.
        
    Returns:
        list: A list of layers, where each layer is a list of voxel dicts in scan order.
    """
    # Step 1 & 2: Quantize Z values
    # Determine unique z levels and quantize
    z_values = sorted(set(v['z'] for v in voxel_cloud))
    # [G] Quantization logic: map continuous z to discrete indices or bins.
    # For simplicity, we assume exact matches or binning is handled.
    # Let's assume we group by exact z for this example, or use a quantization function.
    # In practice, z is quantized to the desired number of layers.
    
    # Group voxels by quantized z-layer
    layers = {}
    for v in voxel_cloud:
        # Quantize z to nearest layer index or bin
        # [G] Specific quantization method (e.g., floor, round) depends on hardware.
        # Assuming z is already quantized or we bin it.
        z_bin = round(v['z']) # Placeholder for specific quantization
        if z_bin not in layers:
            layers[z_bin] = []
        layers[z_bin].append(v)
        
    scan_orders = []
    
    # Step 3 & 4: Process each layer
    for z_bin, points in layers.items():
        if not points:
            continue
            
        # Step 3: Define set P
        # Points already in 'points' list
        
        # Step 4: Calculate optimal scanning order using Christofides
        # We need to pass points to the TSP solver
        # The solver needs to minimize sum of distances
        n = len(points)
        if n == 0:
            continue
        elif n == 1:
            scan_orders.append(points)
            continue
        elif n == 2:
            # Simple case: visit both
            scan_orders.append(points)
            continue
        else:
            # Use Christofides algorithm
            # The algorithm returns a permutation of indices
            tour_indices = christofides_algorithm(points)
            
            # Reorder points according to the tour
            ordered_points = [points[i] for i in tour_indices]
            scan_orders.append(ordered_points)
            
    return scan_orders
```

**Edge Cases and Pitfalls**
*   **Color Switching Speed:** Color expression is limited by the LC filter's switching speed relative to the laser repetition rate. With a 60 Hz filter and 1 kHz laser, different colors are extracted every 17 pulses [S1]. If the scan path requires more than 17 voxels per color state, color fidelity degrades [S1].
*   **Mitigation:** Use faster color extraction devices (non-LC) or develop scanning methods that generate voxels per color [S1].
*   **Boundary Conditions:** The quantization of $z$ must align with the optical axis direction of the display system [S2]. Misalignment causes depth errors.
*   **TSP Complexity:** The Christofides algorithm is used to balance optimality with deterministic execution time, avoiding the prohibitive cost of exact solvers for real-time systems [S1].

## Subtopics

### [Volumetric Display Scanning Route Optimization](5353-volumetric-display-scanning-route-optimization.md)
This method optimizes the traversal path of a beam scanner in a galvanometer-based volumetric display to minimize the total travel distance between voxels within each depth layer, thereby reducing rendering latency and mechanical wear. It solves the Traveling Salesman Problem (TSP) for the 2D coordinates of voxels in each quantized z-layer using the Christofides algorithm, which provides a guaranteed worst-case performance bound for the heuristic. Unlike simpler heuristics (e.g., nearest neighbor) that may produce suboptimal paths, or exact solvers that are computationally prohibitive for real-time systems, this approach balances optimality with deterministic execution time.

## Operating characteristics
*   **Time Complexity:** The dominant cost is solving the TSP for each layer. The Christofides algorithm has a time complexity of $O(N^3)$ for $N$ voxels per layer [S1, S2]. With $L$ layers, the total complexity is $O(L \cdot N^3)$ where $N = M/L$.
*   **Memory Complexity:** $O(M)$ to store the voxel cloud and scan paths [S1, S2].
*   **Measured Performance:** Demonstrated with a 60 Hz LC filter and 1 kHz laser repetition frequency [S1]. Colors are extracted every 17 pulses [S1].
*   **Strengths:** Enables arbitrary per-voxel colors [S1]. Uses Christofides algorithm for near-optimal scan paths with deterministic time [S1]. Compatible with fluorescent glass displays [S3, unofficial].
*   **Limitations:** Color expression is limited by LC filter switching speed [S1]. Requires precise synchronization between scanner, varifocal lens, and color filter [S2].
*   **When to Use:** When multicolor volumetric graphics are required and the display system supports temporal color switching [S1].
*   **When Not to Use:** When higher color update rates are needed than the LC filter can provide, or when real-time performance is critical and $O(N^3)$ TSP solving is too slow [S1].

## Background
The method originates from research on volumetric displays using Drawing Space Separation (DSS) [S1, S2]. It was demonstrated by Miyazaki et al. in 2005 using three galvano scanners to draw volumetric images into fluorescent glass, showing that a beam can excite fluorescent voxels [S3, unofficial]. The specific rendering method for multicolor graphics was detailed in a 2021 paper by Wakunami et al. [S1, S2]. The system uses a 3D scanning optical system with three galvano scanners [S3, unofficial].

## Sources
[S1] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation, https://www.youtube.com/watch?v=W-CbMq23NKk, fetched 2026-10-28
[S3] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
