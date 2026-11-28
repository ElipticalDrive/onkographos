# Greedy-based Box Packing

Also called: Greedy-based Approach

## Overview
Greedy-based Box Packing is a volume rendering pre-processing method that partitions a 3D volume into variably-sized bounding boxes based on alpha values and packs these boxes into 2D texture slices to bypass 3D texture padding waste [S1, unofficial]. It distinguishes itself from its sibling [Factoring Tree Construction](9647-factoring-tree-construction.md) by explicitly compacting isolated non-empty voxels into 2D texture slices, thereby avoiding the severe 3D texture padding waste of Adaptive Mesh Refinement (AMR) structures and the complex decoding overheads of out-of-core streaming [S1, unofficial]. Unlike [Li03a], which partitions based on density and gradient magnitude, this method applies only to alpha values as density magnitude for a streamlined process [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset containing alpha values, a user-defined density threshold ($T_{density}$), and a user-defined sparsity threshold ($T_{sparsity}$) [S1, unofficial].
*   **Output:** A set of 2D texture slices containing packed rectangles, along with a lookup table storing the coordinates and orientation of each packed rectangle for GPU sampling [S1, unofficial].

**Data Structures:**
1.  **Seed Map:** A binary map of the same dimensions as the input volume, where each voxel is marked as a seed (1) if its alpha value $\alpha > T_{density}$, indicating an important voxel; otherwise, it is marked as unimportant (0) [S1, unofficial].
2.  **Bounding Boxes:** A collection of 3D axis-aligned bounding boxes, each defined by its minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$ [S1, unofficial]. Initially, each box is $1 \times 1 \times 1$ [S1, unofficial].
3.  **2D Rectangles:** For each 2D slice, a list of 2D rectangles derived from the intersection of the 3D bounding boxes with that slice [S1, unofficial]. Each rectangle is defined by position and dimensions [S1, unofficial]. The lookup table additionally stores the orientation of each packed rectangle [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_box_packing(volume, X, Y, Z, T_density, T_sparsity):
    # 1. Seed Map Creation
    seed_map = [[[False]*Z for _ in range(Y)] for _ in range(X)]
    for x in range(X):
        for y in range(Y):
            for z in range(Z):
                if volume[x][y][z] > T_density:
                    seed_map[x][y][z] = True
                else:
                    seed_map[x][y][z] = False

    # 2. Sparse Check
    total_voxels = X * Y * Z
    unimportant_voxels = sum(1 for x in range(X) for y in range(Y) for z in range(Z) if not seed_map[x][y][z])
    sparsity = unimportant_voxels / total_voxels
    if sparsity < T_sparsity:
        return greedy_approach(seed_map, X, Y, Z)
    else:
        return regular_2d_packing_approach(seed_map, X, Y, Z) # Details in [Hybrid Greedy/Regular 2D Packing](11866-hybrid-regular-2d-packing.md)

def greedy_approach(seed_map, X, Y, Z):
    boxes = []
    taken = [[[False]*Z for _ in range(Y)] for _ in range(X)]
    for x in range(X):
        for y in range(Y):
            for z in range(Z):
                if seed_map[x][y][z] and not taken[x][y][z]:
                    # Initialize 1x1x1 box
                    box = {'min': (x, y, z), 'max': (x, y, z)}
                    taken[x][y][z] = True
                    # Greedy expansion
                    while True:
                        best_dir = None
                        best_profit = 0
                        # Evaluate 6 directions
                        for dx, dy, dz in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]:
                            nx, ny, nz = box['max'][0]+dx, box['max'][1]+dy, box['max'][2]+dz
                            if 0 <= nx < X and 0 <= ny < Y and 0 <= nz < Z:
                                new_seeds = 1 if seed_map[nx][ny][nz] else 0
                                empty = 0 if seed_map[nx][ny][nz] else 1
                                profit = new_seeds - empty
                                if profit > best_profit:
                                    best_profit = profit
                                    best_dir = (dx, dy, dz)
                        # Expand if profitable
                        if best_profit > 0:
                            dx, dy, dz = best_dir
                            nx, ny, nz = box['max'][0]+dx, box['max'][1]+dy, box['max'][2]+dz
                            box['max'] = (nx, ny, nz)
                            taken[nx][ny][nz] = True
                        else:
                            break
                    boxes.append(box)

    # 3. 2D Slice Intersection & Packing
    slices = []
    lookup_table = []
    for k in range(Z):
        rects = []
        for box in boxes:
            if box['min'][2] <= k <= box['max'][2]:
                rects.append({
                    'x': box['min'][0], 'y': box['min'][1],
                    'w': box['max'][0] - box['min'][0] + 1,
                    'h': box['max'][1] - box['min'][1] + 1
                })
        # Sort rectangles by area descending
        rects.sort(key=lambda r: r['w'] * r['h'], reverse=True)
        
        # 2D Greedy Packing
        packed_rects = []
        # Initial free region covers the slice plane [G]
        free_regions = [{'x': 0, 'y': 0, 'w': 1024, 'h': 1024}] 
        for r in rects:
            best_scenario = None
            best_growth = float('inf')
            # Evaluate 4 placement scenarios for each free region [G]
            for i, region in enumerate(free_regions):
                scenarios = [
                    ('right', region['x'] + region['w'], region['y'], r['w'], r['h']),
                    ('top', region['x'], region['y'] + region['h'], r['w'], r['h']),
                    ('right-transposed', region['x'] + region['w'], region['y'], r['h'], r['w']),
                    ('top-transposed', region['x'], region['y'] + region['h'], r['h'], r['w'])
                ]
                for name, px, py, pw, ph in scenarios:
                    if px + pw <= 1024 and py + ph <= 1024: # Boundary check [G]
                        # Calculate growth in total bounding area [G]
                        current_area = region['w'] * region['h']
                        new_area = max(px + pw, region['x'] + region['w']) * max(py + ph, region['y'] + region['h'])
                        growth = new_area - current_area
                        if growth < best_growth:
                            best_growth = growth
                            best_scenario = (name, px, py, pw, ph, i)
            if best_scenario:
                name, px, py, pw, ph, reg_idx = best_scenario
                packed_rects.append({'x': px, 'y': py, 'w': pw, 'h': ph, 'orient': name})
                # Update free regions by splitting the used region [G]
                free_regions[reg_idx] = {'x': px + pw, 'y': py, 'w': region['w'] - pw, 'h': ph}
                free_regions.append({'x': px, 'y': py + ph, 'w': pw, 'h': region['h'] - ph})
                lookup_table.append({'slice': k, 'rect': packed_rects[-1]})
        slices.append(packed_rects)
    return slices, lookup_table
```

**Formulas:**
*   **Profit:** $\text{Profit} = N_{\text{new\_seeds}} - N_{\text{empty}}$, where $N_{\text{new\_seeds}}$ is the number of new seed voxels encompassed by the expansion, and $N_{\text{empty}}$ is the number of empty (unimportant) voxels encompassed [S1, unofficial]. The direction with the most positive profit is chosen [S1, unofficial].

**Parameters:**
*   $T_{density}$: User-defined density threshold for alpha values [S1, unofficial]. Typical values mentioned in sources include 0.05 and 0.5 [S1, unofficial].
*   $T_{sparsity}$: User-defined sparsity threshold to switch between greedy and regular packing [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The profit calculation relies on integer counts of voxels, avoiding floating-point precision issues during expansion [S1, unofficial].
*   **Boundaries:** Expansion is limited by the volume boundaries [S1, unofficial].
*   **Degenerate Input:** If no voxels exceed $T_{density}$, no boxes are generated [S1, unofficial]. If all voxels are seeds, the greedy algorithm may produce large boxes, potentially reducing packing efficiency if the sparsity check is not tuned correctly [S1, unofficial].

## Subtopics

### [Hybrid Greedy/Regular 2D Packing](11866-hybrid-greedy-regular-2d-packing.md)
Hybrid Greedy/Regular 2D Packing is a volume rendering pre-processing method that partitions a 3D volume into non-empty bounding boxes and packs them into 2D texture slices to bypass 3D texture padding waste. It distinguishes itself from sibling methods by dynamically selecting between a recursive greedy packing strategy and a regular 2D packing strategy based on dataset sparsity, followed by post-packing rotation optimization to minimize memory overhead. Unlike [Li03a], which partitions based on density and gradient magnitude, this method applies only to alpha values as density magnitude for a streamlined process.

### Maximal Rectangles Best Short Side Fit (MAXRECTS-BSSF)
Maximal Rectangles Best Short Side Fit (MAXRECTS-BSSF) is a 2D packing algorithm identified as a top-tier variant for empty-space management, used to resolve spatial inefficiency in 2D rectangle packing by managing maximal rectangles [S2]. It translates theoretical 2D packing algorithms into a practical 3D volume rendering pipeline by applying fast 2D packing strategies directly to non-empty voxels [S2]. Exact solvers guarantee optimal bounds but have massive computational cost, making them impractical for real-time systems [S2]. Simple Shelf algorithms run extremely fast but waste significant surface area whenever a rectangle's height does not match the shelf ceiling [S2].

## Operating characteristics
*   **Time Complexity:** The greedy partitioning step is linear with respect to the number of seed voxels, as each voxel is visited and marked once [S1, unofficial]. The 2D packing step is NP-Hard in the general case, but the greedy heuristic (sorting by area and evaluating four placements) provides a practical approximation suitable for real-time systems [S1, unofficial][S2].
*   **Memory Complexity:** Memory usage is proportional to the number of non-empty voxels and the resulting bounding boxes, significantly less than a full regular grid or AMR structure for sparse volumes [S1, unofficial].
*   **Measured Performance:** On a 128x128 slice, the method condensed the original slice into a smaller dimension (e.g., 54x196), demonstrating effective space compression [S1, unofficial].
*   **Strengths:** Bypasses 3D texture padding waste; avoids complex decoding overheads of out-of-core streaming; streamlined process using only alpha values [S1, unofficial].
*   **Limitations:** The 2D packing is an approximation; the choice of $T_{density}$ and $T_{sparsity}$ requires tuning; the greedy expansion may not produce globally optimal box shapes [S1, unofficial].
*   **When to use:** Suitable for sparse 3D volumes where memory bandwidth is a bottleneck and real-time performance is required [S1, unofficial].
*   **When not to use:** Not suitable for dense volumes where the overhead of packing may outweigh the benefits, or where exact optimal packing is required (though exact solvers are too slow) [S2].

## Background
The Greedy-based Box Packing method was proposed in a paper titled "A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data" [S1, unofficial][S2]. The work addresses the limitations of AMR structures and out-of-core streaming in volume rendering [S1, unofficial]. It builds on the observation that 2D rectangle packing is an NP-Hard problem and leverages the MAXRECTS-BSSF algorithm as a reference for efficient empty-space management [S2]. The method was developed to bridge the gap between theoretical 2D packing algorithms and practical 3D volume rendering pipelines [S1, unofficial].

## Sources
[S1] A Hybrid CPU-GPU Approach for Memory-Efficient Volume ..., https://www.researchgate.net/publication/408581180_A_Hybrid_CPU-GPU_Approach_for_Memory-Efficient_Volume_Rendering_of_Medical_Image_Data, fetched 2026-10-26 (unofficial)
[S2] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://doi.org/10.24132/csrn.2026-d53, fetched 2026-10-26
