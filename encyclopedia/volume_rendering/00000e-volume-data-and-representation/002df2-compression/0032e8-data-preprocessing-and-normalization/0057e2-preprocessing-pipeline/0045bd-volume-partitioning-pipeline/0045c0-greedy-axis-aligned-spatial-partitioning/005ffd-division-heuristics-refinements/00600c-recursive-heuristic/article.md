# Recursive Heuristic
Also called: Recursive subdivision scheme
## Overview
The Recursive Heuristic is a load-balancing strategy for parallel volume rendering that recursively partitions the image plane into smaller rectangular regions until each processor manages a single region [S1, unofficial]. It serves as a general superset of simpler 1D horizontal and vertical division schemes, dynamically choosing between horizontal and vertical cuts at each recursion level to optimize load balance [S1, unofficial]. Unlike fixed-axis partitioning, it adapts the division orientation based on the computed load distribution, offering greater scalability flexibility at the cost of increased data exchange and loss of scanline coherency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The current image plane region assigned to a cluster of processors, the total number of processors $P$, and the local load distribution within that region.
- Output: A partitioned image plane where each processor holds a distinct rectangular sub-region, along with the migrated data required to render that sub-region.

**Data Structures:**
- **Image Plane Region:** Defined by bounding coordinates $(x_{min}, y_{min}, x_{max}, y_{max})$.
- **Load Metric:** A scalar value representing the computational cost (e.g., number of active polygons, voxel density) within a region.
- **Processor Set:** A list or array of processor IDs currently responsible for the region.

**Algorithm Steps:**
1. **Calculate Load:** Determine the total computational load $L$ of the current image plane region assigned to the active processor set.
2. **Evaluate Horizontal Cut:** Compute the load distribution if the region were divided horizontally into two equal-area sub-regions. Let $L_{h1}$ and $L_{h2}$ be the loads of the top and bottom halves.
3. **Evaluate Vertical Cut:** Compute the load distribution if the region were divided vertically into two equal-area sub-regions. Let $L_{v1}$ and $L_{v2}$ be the loads of the left and right halves.
4. **Select Best Division:** Compare the balance of the two options. The "best" division is the one that minimizes the difference between the loads of the two resulting sub-regions (i.e., minimizes $|L_{h1} - L_{h2}|$ or $|L_{v1} - L_{v2}|$). Choose the orientation (horizontal or vertical) that yields the better balance.
5. **Data Exchange:** Exchange local data with neighboring processors to migrate geometry and volume data according to the chosen new boundary. This step ensures each processor has the necessary data for its new sub-region.
6. **Partition Processors:** Split the current set of processors into two halves. Assign one half to the first sub-region and the other half to the second sub-region.
7. **Recurse:** Repeat steps 1-6 for each new sub-region with its assigned subset of processors.
8. **Termination:** The recursion stops when each sub-region is assigned to exactly one processor. The total number of recursion passes is $\log_2(P)$, where $P$ is the total number of processors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_heuristic(processors, region, total_processors):
    """
    Recursively partition the image plane and processors.
    
    Args:
        processors: List of processor IDs assigned to this region.
        region: Tuple (x_min, y_min, x_max, y_max) defining the image plane area.
        total_processors: Total number of processors in the system (P).
    """
    # Base case: One processor per region
    if len(processors) == 1:
        return processors[0], region

    # Step 1: Find overall load in the local image plane
    current_load = calculate_load(region, processors)
    
    # Step 2: Divide load horizontally
    # Assume equal area split for simplicity in load estimation
    h_mid_y = (region[1] + region[3]) / 2.0
    region_top = (region[0], region[1], region[2], h_mid_y)
    region_bottom = (region[0], h_mid_y, region[2], region[3])
    load_top = calculate_load(region_top, processors)
    load_bottom = current_load - load_top
    h_balance = abs(load_top - load_bottom)
    
    # Step 3: Divide load vertically
    v_mid_x = (region[0] + region[2]) / 2.0
    region_left = (region[0], region[1], v_mid_x, region[3])
    region_right = (v_mid_x, region[1], region[2], region[3])
    load_left = calculate_load(region_left, processors)
    load_right = current_load - load_left
    v_balance = abs(load_left - load_right)
    
    # Step 4: Compare and choose best
    if h_balance <= v_balance:
        # Choose horizontal division
        chosen_region_1 = region_top
        chosen_region_2 = region_bottom
        balance_diff = h_balance
    else:
        # Choose vertical division
        chosen_region_1 = region_left
        chosen_region_2 = region_right
        balance_diff = v_balance
        
    # Step 5: Exchange local data
    # This step involves communicating with other processors to move geometry/volume data
    # across the new boundary. Implementation depends on the specific parallel framework.
    exchange_data(processors, chosen_region_1, chosen_region_2)
    
    # Step 6: Set half processors to one region, other half to the other
    mid_idx = len(processors) // 2
    processors_1 = processors[:mid_idx]
    processors_2 = processors[mid_idx:]
    
    # Step 7: Recurse
    result_1 = recursive_heuristic(processors_1, chosen_region_1, total_processors)
    result_2 = recursive_heuristic(processors_2, chosen_region_2, total_processors)
    
    return result_1, result_2

def calculate_load(region, processors):
    """
    [G] Placeholder for actual load calculation.
    In practice, this sums the complexity of primitives (polygons, voxels) 
    intersecting the region.
    """
    # Not given in the sources. Typically involves traversing the scene graph 
    # or volume grid to count active elements within the bounding box.
    return 0.0

def exchange_data(processors, region_1, region_2):
    """
    [G] Placeholder for data migration.
    Moves geometry and volume data from processors leaving a region to those 
    entering it.
    """
    # Not given in the sources. Requires specific communication primitives 
    # (e.g., MPI_Send, MPI_Recv) based on the boundary coordinates.
    pass
```

**Parameters:**
- $P$: Total number of processors. Determines the depth of recursion, which is $\log_2(P)$ passes [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Coherency Loss:** Vertical division disturbs intra-scanline coherency, causing loss of active polygon, edge, x-bucket, and z-list information. Horizontal division disturbs inter-scanline coherency, causing loss of active polygon information between scanlines [S1, unofficial]. This necessitates recreation of incremental structures (like insertion-sorted z-lists), increasing processing time [S1, unofficial].
- **Job Approximation Deviation:** The disturbance of coherencies can cause job approximations within a region to deviate drastically, potentially affecting accuracy or requiring additional error correction steps [S1, unofficial].

## Subtopics

### [Rectangular Heuristic](18043-rectangular-heuristic.md)
A load balancing technique that partitions the image plane into rectangular regions by organizing processors into a 2D M x K mesh, dividing the image into M horizontal bands and then K vertical bands within each band to improve scalability over 1D horizontal division.

## Operating characteristics
- **Time Complexity:** The recursion depth is $\log_2(P)$, leading to $O(\log P)$ passes of data exchange and partitioning [S1, unofficial]. Each pass involves load calculation and data migration, which depends on the scene complexity and communication bandwidth.
- **Memory Complexity:** Requires storage for the image plane regions and temporary buffers for data exchange during migration. No significant additional memory overhead beyond the scene data itself is specified [S1, unofficial].
- **Measured Performance:** The method offers greater scalability flexibility compared to fixed-axis division but incurs extra processing time due to the loss of scanline coherency [S1, unofficial]. The trade-off is between the improved load balance (scalability) and the overhead of recreating coherency structures.
- **Strengths:** Dynamic load balancing; adapts to scene complexity; scalable to large numbers of processors.
- **Limitations:** High communication overhead due to data exchange; loss of coherency leads to increased computational work per pixel/voxel; job approximation errors.
- **When to Use:** When the scene load is unevenly distributed and fixed-axis partitioning would lead to significant load imbalance. Suitable for large-scale parallel rendering where scalability is prioritized over minimal coherency overhead.
- **When Not to Use:** When the scene is uniformly distributed or when coherency preservation is critical for performance (e.g., simple scenes where horizontal/vertical division suffices).

## Background
The Recursive Heuristic was introduced as a general subdivision heuristic in the context of parallel volume rendering, specifically as a superset of earlier 1D horizontal and vertical division algorithms [S1, unofficial]. It was developed to address the limitations of fixed-axis partitioning by allowing dynamic choice of division orientation to achieve better load balance [S1, unofficial]. The method is discussed in the context of parallel algorithms for volume rendering, often in conjunction with scanline z-buffer paradigms [S1, unofficial].

## Sources
[S1] ;Г, ,1:. 'i f ,3 Ψ |'̂*i, ̂ t ѴУ t ̂ -.̂ 'j f'\ ' ‘'̂ '· îM. Vl ■I ·' !■' Í; 1 ! é .1 li ifi; .· ̂'■ '’(■H ' ■'“' ’"ΐ ΐ'-ΐ ...., https://repository.bilkent.edu.tr/server/api/core/bitstreams/a7dc03c6-3640-4028-b1b3-d6296aa2d23c/content, fetched 2026-10-30 (unofficial)
