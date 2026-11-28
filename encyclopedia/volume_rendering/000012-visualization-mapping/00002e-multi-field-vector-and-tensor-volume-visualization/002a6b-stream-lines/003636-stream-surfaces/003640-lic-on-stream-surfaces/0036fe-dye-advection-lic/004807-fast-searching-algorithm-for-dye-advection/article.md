# Fast Searching Algorithm for Dye Advection

Also called: Breadth First Search for dye path, Direct Flow-Back Search, Direct flow-back neighbors, Flow-back Neighbor Search for Dye Advection

## Overview
This algorithm efficiently identifies the set of grid cells affected by a new dye injection in Dye-Advection LIC without scanning the entire volume or storing exhaustive backward streamline histories [S1, unofficial]. It differs from brute-force neighbor lists by constructing a sparse directed graph of "direct flow-back neighbors" and using Breadth First Search (BFS) to locate the affected region, significantly reducing memory overhead [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A 3D vector field discretized on a Cartesian grid [S1, unofficial].
- The convolution length $L$ [S1, unofficial].
- A root cell $a$ where dye is injected [S1, unofficial].
- Pre-computed direct flow-back neighbor relationships for all cells, generated during the initial standard LIC pass [S1, unofficial].

**Outputs:**
- The set $\omega(a)$, a superset of the actually affected cells $\Omega(a)$, containing all cells whose negative streamlines pass through the dyed region within distance $L$ [S1, unofficial].

### Data Structures
The core data structure is a directed graph stored as an adjacency list [S1, unofficial].
- **Nodes:** Grid cells [S1, unofficial].
- **Edges:** Represent the "direct flow-back" relation [S1, unofficial].
- **Storage:** For each cell $c$, a list `direct_flow_back_neighbors[c]` containing indices of cells $\beta$ such that $c \to \beta$ [S1, unofficial].
- **Size:** The number of entries per cell is typically small (average 2–3 in 3D flows), except near critical points [S1, unofficial].

### Definitions and Formulas
The algorithm relies on three hierarchical definitions to approximate the affected region [S1, unofficial]:

1. **Backward Streamline Relation ($\downarrow$):**
   For cells $\alpha$ and $\beta$, $\alpha \downarrow \beta$ if and only if there is a backward streamline flowing through $\alpha$ and stepping into $\beta$ [S1, unofficial].

2. **Affected Set ($\Omega(a)$):**
   The set of cells actually affected by dye injected at cell $a$:
   $$ \Omega(a) = \{p \mid p \downarrow a : d(p, a) < L\} $$
   where $d(p, a)$ is the arc length of the streamline from point $p$ to point $a$, and $L$ is the convolution length [S1, unofficial].

3. **Transitive Backward Relation ($\Lambda$):**
   For cells $\alpha$ and $\beta$, $\alpha \Lambda \beta$ if $\alpha \downarrow \beta$ or there exists a path $\{p_0, p_1, \dots, p_n\}$ such that $\alpha \downarrow p_0, p_0 \downarrow p_1, \dots, p_n \downarrow \beta$ [S1, unofficial].
   *Axiom:* If $\alpha \downarrow \beta$ then $\alpha \Lambda \beta$ [S1, unofficial].

4. **Search Superset ($\omega(a)$):**
   $$ \omega(a) = \{p \mid p \Lambda a, d(p, a) < L\} $$
   *Axiom:* $\omega(a) \supseteq \Omega(a)$ [S1, unofficial]. In practice, the difference in size between $\Omega(a)$ and $\omega(a)$ is very small [S1, unofficial].

5. **Direct Flow-Back Relation ($\to$):**
   A cell $\alpha \to \beta$ if and only if $\alpha \downarrow \beta$ and $\alpha$ is $\beta$'s direct neighbor [S1, unofficial].
   *Direct Neighbors:* Cells directly adjacent in physical space [S1, unofficial].
   - In 2D Cartesian grids: 8 neighbors [S1, unofficial].
   - In 3D Cartesian grids: 26 neighbors [S1, unofficial].

### Algorithm Steps

1. **Pre-computation (During Standard LIC):**
   For every cell $\beta$ in the grid:
   a. Identify all direct neighbors $\alpha$ (up to 26 in 3D) [S1, unofficial].
   b. Trace the backward streamline starting from $\alpha$ [S1, unofficial].
   c. If the backward streamline steps into $\beta$, add $\alpha$ to `direct_flow_back_neighbors[β]` [S1, unofficial].
   d. Store this list. This forms the directed graph where nodes are cells and edges are direct flow-back relationships [S1, unofficial].

2. **Search for Affected Cells (BFS):**
   Given a root cell $a$ and convolution length $L$:
   a. Initialize a queue $Q$ with cell $a$ [S1, unofficial].
   b. Initialize a set `visited` containing $a$ [S1, unofficial].
   c. Initialize a list `affected_cells` containing $a$ [S1, unofficial].
   d. While $Q$ is not empty:
      i. Dequeue cell $u$ [S1, unofficial].
      ii. For each neighbor $v$ in `direct_flow_back_neighbors[u]`:
          1. If $v$ is not in `visited`:
             a. Calculate the arc length $d(v, a)$ along the streamline from $v$ to $a$ [G: Requires pre-computed streamline lengths or on-the-fly tracing during the initial LIC pass].
             b. If $d(v, a) < L$:
                i. Add $v$ to `visited` [S1, unofficial].
                ii. Add $v$ to `affected_cells` [S1, unofficial].
                iii. Enqueue $v$ [S1, unofficial].
   e. Return `affected_cells` as $\omega(a)$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_search_dye_advection(root_cell, convolution_length, direct_flow_back_neighbors, get_streamline_distance):
    """
    Finds the set of cells affected by dye injection at root_cell.
    
    Args:
        root_cell: The index of the cell where dye is injected.
        convolution_length: L, the maximum arc length for dye influence.
        direct_flow_back_neighbors: A dictionary/list where direct_flow_back_neighbors[u] 
                                    returns a list of cells v such that v -> u.
        get_streamline_distance: A function that returns the arc length d(p, a) from point p to root_cell a.
                                 [G: Assumes streamline distance is computable or pre-stored. 
                                  If not, this function must trace the streamline from p to a.]
    
    Returns:
        A list of cell indices in the set omega(a).
    """
    # Initialize BFS structures
    queue = [root_cell]
    visited = {root_cell}
    affected_cells = [root_cell]
    
    # BFS Loop
    while queue:
        current_cell = queue.pop(0)
        
        # Get direct flow-back neighbors of the current cell
        # These are cells that flow directly into current_cell
        neighbors = direct_flow_back_neighbors.get(current_cell, [])
        
        for neighbor in neighbors:
            if neighbor not in visited:
                # Calculate distance from neighbor to root
                # d(neighbor, root_cell) is the arc length along the streamline
                dist = get_streamline_distance(neighbor, root_cell)
                
                if dist < convolution_length:
                    visited.add(neighbor)
                    affected_cells.append(neighbor)
                    queue.append(neighbor)
                # If dist >= convolution_length, we do not add to visited or queue.
                # Pruning at dist >= L is correct for the superset definition.
                
    return affected_cells
```

### Parameters
- **Convolution Length ($L$):** Determines the spatial extent of the dye [S1, unofficial].
- **Direct Neighbor Count:** Fixed by grid topology: 8 in 2D Cartesian grids and 26 in 3D Cartesian grids [S1, unofficial].

### Edge Cases and Pitfalls
- **Critical Points:** Cells near critical points (sources, sinks, saddles) may have a high number of direct flow-back neighbors, potentially negating the memory efficiency gain [S1, unofficial].
- **Distance Calculation:** The algorithm requires $d(p, a)$, the arc length [S1, unofficial]. If this is not pre-computed during the initial LIC pass, computing it on the fly for every neighbor in the BFS can be costly [S1, unofficial].
- **Superset Approximation:** $\omega(a)$ is a superset of $\Omega(a)$ [S1, unofficial]. While the difference is small in practice, it may include cells that are topologically connected via direct neighbors but not directly on the streamline within $L$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{\omega})$, where $N_{\omega}$ is the number of cells in the superset $\omega(a)$ [S1, unofficial]. The BFS visits each cell in $\omega(a)$ once [S1, unofficial]. The cost per cell is proportional to the number of direct flow-back neighbors (average 2–3 in 3D) [S1, unofficial]. This is significantly faster than brute-force scanning of the entire volume or traversing long streamline lists [S1, unofficial].
- **Memory Complexity:** $O(N \cdot k)$, where $N$ is the total number of cells and $k$ is the average number of direct flow-back neighbors per cell [S1, unofficial]. $k$ is typically 2–3, compared to potentially hundreds or thousands for brute-force lists [S1, unofficial]. This makes the method highly memory efficient [S1, unofficial].
- **Scaling:** Scales well with volume size because the search is localized to the dyed region's upstream influence [S1, unofficial]. Memory efficiency is negated near critical points due to a high number of direct flow-back neighbors [S1, unofficial].
- **Strengths:** Low memory footprint, fast search time, simple implementation (BFS on pre-computed graph) [S1, unofficial].
- **Limitations:** Requires pre-computation during initial LIC [S1, unofficial]. Less efficient near critical points [S1, unofficial]. Approximates the affected set (superset) [S1, unofficial].

## Background
The algorithm was proposed in the context of Dye-Advection LIC, an extension of Line Integral Convolution (LIC) that simulates the advection of dye in a vector field [S1, unofficial]. The standard LIC method convolves a noise texture along streamlines [S1, unofficial]. Dye advection allows for dynamic visualization of flow features by injecting dye at specific points [S1, unofficial]. The fast searching algorithm was introduced to solve the problem of efficiently identifying which cells need to be updated when new dye is injected, avoiding the prohibitive memory costs of storing full backward streamline histories [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970011183: Visualizing Vector Fields Using Line Integral Convolution and Dye Advection, https://archive.org/details/NASA_NTRS_Archive_19970011183, fetched 2026-10-26 (unofficial)
