# Galvanometer Scanner Volume Rendering

Also called: DSS color volumetric display rendering, Drawing of multicolor volumetric graphics, LC Color Filter Volume Rendering, LC filter synchronized voxel generation, Multicolor Volumetric Graphics, Point Cloud Scanner Volume Rendering, Temporal color-synchronized volumometric rendering

## Overview
This method renders volumetric images with arbitrary per-voxel colors by temporally synchronizing the spatial generation of voxels with the switching of a Liquid Crystal (LC) color filter [S1, S2]. It solves the problem of multicolor volumetric display by mapping 3D voxel data to control signals for a galvanometer scanner, a varifocal lens, and an LC filter [S2]. Unlike monochromatic scanning methods, it explicitly calculates optimal scanning routes for color layers to minimize mechanical travel time while ensuring color fidelity through temporal multiplexing [S1]. It builds on [Drawing Space Separation (DSS)](5290-drawing-space-separation-dss.md) by adding a color-synchronization layer to the spatial mapping process.

## How it works
**Inputs and Outputs**
*   **Input:** A 3D point cloud $V$ consisting of $M$ voxels. Each voxel $v_i$ is defined by its spatial coordinates and RGB color values: $v_i = (x_i, y_i, z_i, r_i, g_i, b_i)$ [S1, S2].
*   **Output:** Application orders (sequences) of 3D coordinates and color patterns for the beam scanner and LC color filter, specifically $(x, y)$ values for the galvanometer scanner, $z$ values for the varifocal lens, and $(r, g, b)$ values for the LC color filter [S2].

**Data Structures**
*   **Voxel Set:** $V = \{v_1, v_2, \ldots, v_M\}$ [S1, S2].
*   **Layer Set:** For a quantized $z$-axis, a layer $L_z$ contains a set of vertices $P = \{p_1, p_2, \ldots, p_N\}$ [S2]. Each vertex $p_j$ in the layer has 2D spatial position and RGB values: $p_j = (x_j, y_j, r_j, g_j, b_j)$ [S2].

**Parameters**
*   **LC Color Filter Refresh Rate:** 60 Hz [S1].
*   **Laser Repetition Frequency:** 1 kHz [S1].
*   **Color Extraction Interval:** Different colors are extracted every 17 pulses given the above rates [S1].
*   **Quantization Layers:** The number of $z$-layers is a user-defined parameter (e.g., 30 layers used in demonstrations) [S2].

**Steps**
1.  **Quantization:** Quantize the $z$ values of $V$ to the desired number of layers. The $z$-axis must match the optical axis direction of the display system [S2].
2.  **Layer Partitioning:** For each quantized layer $z$, define the set of vertices $P$ present in that layer [S2].
3.  **Route Optimization:** For each layer $P$, calculate the optimal scanning order that minimizes the total route length. The objective function is:
    $$ \sum_{j=1}^{N-1} d(p_j, p_{j+1}) + d(p_N, p_1) $$
    where $d(p_j, p_{j+1})$ is the Euclidean distance between the 2D coordinates $(x_j, y_j)$ and $(x_{j+1}, y_{j+1})$ of adjacent vertices in the sequence [S2].
4.  **Algorithm Application:** Apply the Christofides algorithm to all layers to find the shortest Hamiltonian cycle (scanning route) for each layer [S2].
5.  **Signal Generation:** Generate the application orders for the display system based on the optimized routes. This yields the synchronized $(x, y)$ commands for the galvanometer scanner, $z$ commands for the varifocal lens, and $(r, g, b)$ commands for the LC color filter [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def galvanometer_scanner_volume_rendering(V, num_layers, laser_freq_hz, filter_refresh_hz):
    """
    V: List of voxels, each voxel is a tuple (x, y, z, r, g, b)
    num_layers: Integer, number of quantized z-layers
    laser_freq_hz: Integer, laser repetition frequency (e.g., 1000)
    filter_refresh_hz: Integer, LC filter refresh rate (e.g., 60)
    """
    # Step 1: Quantize z-values
    # Determine z-bounds and bin size
    z_min = min(v[2] for v in V)
    z_max = max(v[2] for v in V)
    z_range = z_max - z_min
    if z_range == 0:
        z_bin_size = 1.0
    else:
        z_bin_size = z_range / num_layers
    
    # Create layers
    layers = [[] for _ in range(num_layers)]
    for v in V:
        x, y, z, r, g, b = v
        # Calculate layer index
        layer_idx = int((z - z_min) / z_bin_size)
        # Clamp to valid range
        layer_idx = max(0, min(layer_idx, num_layers - 1))
        # Store 2D position and color for this layer
        layers[layer_idx].append((x, y, r, g, b))

    # Step 2 & 3: Optimize scanning routes per layer
    optimized_routes = []
    for layer in layers:
        if not layer:
            optimized_routes.append([])
            continue
        
        # Extract 2D coordinates for distance calculation
        points_2d = [(p[0], p[1]) for p in layer]
        n = len(points_2d)
        
        # Calculate distance matrix
        dist_matrix = [[0.0] * n for _ in range(n)]
        for i in range(n):
            for j in range(n):
                dx = points_2d[i][0] - points_2d[j][0]
                dy = points_2d[i][1] - points_2d[j][1]
                dist_matrix[i][j] = (dx**2 + dy**2)**0.5
        
        # Step 4: Apply Christofides algorithm to find shortest Hamiltonian cycle
        # Note: Christofides algorithm implementation is assumed available or standard
        # It returns a permutation of indices representing the optimal order
        route_indices = christofides_algorithm(dist_matrix)
        
        # Map indices back to full voxel data (x, y, r, g, b)
        route = [layer[i] for i in route_indices]
        optimized_routes.append(route)

    # Step 5: Generate application orders
    # The output is a sequence of commands for the hardware
    # Format: (x, y, z_layer, r, g, b)
    # Note: z is constant for all voxels in a specific layer's route
    application_orders = []
    for layer_idx, route in enumerate(optimized_routes):
        # Determine the representative z for this layer
        # In practice, this might be the center of the quantized bin
        z_val = z_min + (layer_idx + 0.5) * z_bin_size
        
        for point in route:
            x, y, r, g, b = point
            application_orders.append((x, y, z_val, r, g, b))
            
    return application_orders

def christofides_algorithm(dist_matrix):
    """
    Placeholder for the Christofides algorithm implementation.
    Returns a list of indices representing the optimal tour.
    """
    # Implementation details of Christofides algorithm (Minimum Spanning Tree,
    # Minimum Weight Perfect Matching on odd-degree vertices, Eulerian tour
    # conversion to Hamiltonian cycle) are not provided in the source.
    # [G] Standard graph theory implementation required.
    pass
```

**Edge Cases and Pitfalls**
*   **Color Switching Limit:** The color expression is limited by the LC filter's switching speed relative to the laser repetition rate. With a 60 Hz filter and 1 kHz laser, only one color can be extracted every 17 pulses [S1].
*   **Mitigation:** Use faster color extraction devices (non-LC) or develop scanning methods that generate voxels per color to expand color expression [S1].
*   **Spatial vs. Temporal:** The method relies on temporal synchronization. If the mechanical latency of the scanner or lens exceeds the timing precision required for the 17-pulse color window, color bleeding or misregistration may occur [S1, S2].

## Subtopics

### [Volumetric Display Scanning Route Optimization](5353-volumetric-display-scanning-route-optimization.md)
Optimizes the traversal path of the beam scanner within each depth layer to minimize total travel distance, reducing rendering latency and mechanical wear. It solves the Traveling Salesman Problem (TSP) for 2D voxel coordinates using the Christofides algorithm, balancing optimality with deterministic execution time compared to nearest-neighbor heuristics or exact solvers.

## Operating characteristics
*   **Time Complexity:** The dominant computational cost is the route optimization for each layer. Using the Christofides algorithm, the complexity is $O(N^3)$ for a layer with $N$ voxels, where $N$ is the number of voxels per layer [S2].
*   **Memory Complexity:** Linear with respect to the number of voxels $M$ to store the point cloud and layer partitions [S2].
*   **Performance:** Demonstrated with 30 quantized layers [S2]. The system uses a laser repetition frequency of 1 kHz and an LC filter refresh rate of 60 Hz [S1].
*   **Strengths:** Enables arbitrary per-voxel color rendering in a volumetric display based on DSS [S1]. The use of Xenon (Xe) as a drawing space reduces pulse energy requirements and improves brightness [S1].
*   **Limitations:** Color depth and switching speed are constrained by the LC filter's refresh rate (60 Hz) and the laser repetition rate (1 kHz), limiting color changes to every 17 pulses [S1].
*   **When to Use:** When multicolor volumetric graphics are required in a DSS-based system and the hardware supports galvanometer scanning with a varifocal lens and an LC color filter [S1, S2].
*   **When Not to Use:** When real-time color switching faster than the LC filter's refresh rate is required, or when the hardware lacks a color-filtering mechanism [S1].

## Background
This method was introduced in the context of Drawing Space Separation (DSS) volumetric displays. It was developed to enable coloring on a voxel-by-voxel basis by temporally synchronizing voxel generation positions with the color switching of an LC filter [S1]. The research was published in 2021, demonstrating graphics such as Lissajous figures, the Armadillo model, the Stanford Bunny, and a knight with a light saber [S1]. The approach leverages the high brightness and low pulse energy requirements of Xe-based drawing spaces [S1].

## Sources
[S1] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation., https://doi.org/10.1038/s41598-021-02107-3, fetched 2026-10-26
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation, https://www.youtube.com/watch?v=W-CbMq23NKk, fetched 2026-10-28
