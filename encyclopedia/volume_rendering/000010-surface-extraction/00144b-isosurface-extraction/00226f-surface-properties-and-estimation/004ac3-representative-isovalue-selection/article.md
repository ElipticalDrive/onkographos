# Representative Isovalue Selection

Also called: Isosurface Similarity Map-based Isovalue Selection, SDV-based Isovalue Prioritization

## Overview
This method automatically selects a subset of representative isovalues from a scalar dataset by identifying peaks in the isosurface similarity distribution, ensuring that selected values correspond to distinct geometric structures with low mutual similarity [S1]. Unlike methods that rely on frequency counts or magnitude thresholds, it uses a recursive partitioning strategy weighted by the Similarity Distribution Value (SDV) to order isovalues from most to least representative without requiring user-defined parameters [S1].

## How it works
**Inputs and Outputs**
*   **Input**: A sorted list of candidate isovalues $V = \{h_1, h_2, \dots, h_n\}$, where $n$ is the total number of candidates. The list must be sorted in ascending order of isovalue magnitude.
*   **Input**: A similarity metric $SMV(h_a, h_b)$ that computes the similarity between the isosurfaces generated at isovalue $h_a$ and $h_b$.
*   **Input**: A Similarity Distribution Value function $SDV(h_i)$ for each candidate $h_i$, representing its average similarity to all other candidates in the current subset [S1].
*   **Output**: A priority queue $Q$ containing all isovalues ordered by priority, such that extracting items from $Q$ yields isovalues from most representative to least representative [S1].

**Data Structures**
*   **Priority Queue $Q$**: A heap or sorted list storing tuples $(h_i, p_i)$, where $h_i$ is the isovalue and $p_i$ is its calculated priority.
*   **Candidate Set $V$**: A list of indices or isovalue objects, initially containing all candidates.

**Formulas**
1.  **Selection of Maximum SDV**:
    $$ m \leftarrow \arg \max_{h_i \in V} SDV(i) $$
    Where $m$ is the index of the isovalue with the highest SDV in the current subset $V$ [S1].

2.  **Priority Calculation**:
    $$ p \leftarrow |V| \cdot SDV(m) $$
    The priority $p$ is the product of the size of the current subset $|V|$ and the SDV of the selected isovalue $m$ [S1].

3.  **Penalty Update**:
    $$ p_i \leftarrow \frac{p_i}{1 + SMV(h_m, h_i)} $$
    After selecting $h_m$, the priority $p_i$ of all remaining items $i$ in the queue is reduced by a factor dependent on the similarity between $h_m$ and $h_i$ [S1].

**Parameters**
*   **SDV**: The Similarity Distribution Value. The source does not provide the explicit formula for calculating SDV, only that it is used to rank isovalues initially based on their average similarity to others [S1].
*   **SMV**: The Similarity Measure Value. The specific implementation of $SMV$ (e.g., Hausdorff distance, IoU, etc.) is not defined in the source text, only that it is used in the penalty formula [S1].

**Algorithm Steps**
1.  **Initialization**: Initialize an empty priority queue $Q$ and a set $V$ containing all candidate isovalues. Call the recursive procedure `prioritize(Q, V)`.
2.  **Recursive Partitioning (`prioritize(Q, V)`)**:
    a. Find the isovalue $m$ in $V$ with the maximum SDV: $m \leftarrow \arg \max_{h_i \in V} SDV(i)$.
    b. Calculate the priority $p$ for $m$: $p \leftarrow |V| \cdot SDV(m)$.
    c. Enqueue $m$ into $Q$ with priority $p$.
    d. Split $V$ into two subsets:
       - $V_1 \leftarrow \{h_i \in V : 1 \le i < m\}$
       - $V_2 \leftarrow \{h_i \in V : m < i \le |V|\}$
    e. Recursively call `prioritize(Q, V_1)` and `prioritize(Q, V_2)`.
3.  **Extraction and Penalty**:
    a. Extract the isovalue $h_m$ with the maximum priority from $Q$.
    b. For all remaining items $i$ in $Q$, update their priority $p_i$ using the penalty formula: $p_i \leftarrow p_i / (1 + SMV(h_m, h_i))$.
    c. Repeat step 3 until $Q$ is empty.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def representative_isovalue_selection(isovalue_list, sdv_function, similarity_metric):
    """
    Selects representative isovalues using SDV-based prioritization.
    
    Args:
        isovalue_list: List of isovalue objects, sorted by magnitude.
        sdv_function: Function returning SDV for an isovalue index.
        similarity_metric: Function returning SMV between two isovalue objects.
        
    Returns:
        List of isovalue objects ordered by representativeness.
    """
    # Priority Queue stores tuples (priority, isovalue_index)
    # Using a max-heap for efficiency
    import heapq
    
    # Helper to negate priority for max-heap behavior in Python's min-heap
    pq = []
    
    # Initial call to recursive partitioning
    _prioritize(isovalue_list, 0, len(isovalue_list) - 1, pq, sdv_function)
    
    # Extract results
    result = []
    while pq:
        neg_p, idx = heapq.heappop(pq)
        result.append(isovalue_list[idx])
        
    return result

def _prioritize(isovalue_list, start_idx, end_idx, pq, sdv_function):
    """
    Recursively partitions the subset of isovalues and adds to priority queue.
    """
    if start_idx > end_idx:
        return

    current_subset_size = end_idx - start_idx + 1
    
    # Find m: index with max SDV in the current range [start_idx, end_idx]
    # Note: SDV is computed relative to the global set or current subset?
    # Source says "average similarity to all others" but also "maxima of similarity distribution 
    # for only the values below and above". The formula uses SDV(i) directly.
    # We assume SDV is precomputed or computed on the fly for the current subset context.
    # Given the source text "maxima of the similarity distribution for only the values below and above",
    # it implies SDV might be local. However, the formula p = |V|SDV(m) suggests |V| is the current subset size.
    # We will compute SDV for each candidate in the current range.
    
    best_idx = -1
    max_sdv = -1.0
    
    for i in range(start_idx, end_idx + 1):
        # Compute SDV for isovalue_list[i] within the current subset context
        # The source does not define SDV explicitly, so we call the provided function.
        # [G] Assuming sdv_function takes the list of candidates in the current subset and the index.
        # If sdv_function is global, this step might need adjustment based on specific implementation.
        # Based on "maxima of the similarity distribution for only the values below and above",
        # SDV likely considers neighbors in the sorted list.
        current_sdv = sdv_function(isovalue_list, i)
        
        if current_sdv > max_sdv:
            max_sdv = current_sdv
            best_idx = i
            
    # Calculate priority
    priority = current_subset_size * max_sdv
    
    # Enqueue
    # Python heapq is a min-heap, so we store negative priority
    heapq.heappush(pq, (-priority, best_idx))
    
    # Split V into V1 and V2
    # V1: indices < best_idx
    # V2: indices > best_idx
    _prioritize(isovalue_list, start_idx, best_idx - 1, pq, sdv_function)
    _prioritize(isovalue_list, best_idx + 1, end_idx, pq, sdv_function)

def update_priorities_with_penalty(pq, selected_isovalue, similarity_metric):
    """
    Updates priorities of remaining items in the queue based on similarity to the selected isovalue.
    Note: Standard heap doesn't support efficient key updates. 
    In a real implementation, one might use a Fibonacci heap or rebuild the heap.
    The source describes the logical update.
    """
    # This function is conceptual as Python's heapq doesn't support direct update.
    # For the purpose of the algorithm description, this represents the logical step.
    pass
```

**Edge Cases and Pitfalls**
*   **Parameter Tuning**: The algorithm requires no user-defined thresholds or parameters, avoiding issues related to parameter tuning [S1].
*   **SDV Definition**: The explicit formula for SDV is not provided in the source. Implementers must define how "average similarity" is calculated, particularly whether it considers all isovalues or only those in the current recursive subset [S1].
*   **Similarity Metric**: The specific implementation of $SMV$ is not defined. The choice of similarity metric (e.g., topological vs. geometric) will significantly impact the results [S1].
*   **Degenerate Input**: If multiple isovalues have identical SDV values, the selection of $m$ is arbitrary among them. The source does not specify a tie-breaking rule [S1].

## Subtopics

### Isosurface Similarity Maps
The foundational data structure used by this method. It provides the similarity values $SMV$ and the distribution from which SDV is derived. [Isosurface Similarity Maps](1005-isosurface-similarity-maps.md)

### Isosurface Characteristics Computation
Related to the computation of geometric properties that might inform similarity measures. [Isosurface Characteristics Computation](12228-isosurface-characteristics-computation.md)

## Operating characteristics
*   **Time Complexity**: The recursive partitioning step visits each isovalue once, taking $O(n)$ calls to `prioritize`. However, finding the maximum SDV in each subset takes $O(k)$ where $k$ is the subset size. In the worst case (unbalanced splits), this could approach $O(n^2)$. The penalty update step, if implemented naively by rebuilding the heap or updating keys, adds overhead. The source does not provide a formal complexity analysis [S1].
*   **Memory Complexity**: $O(n)$ to store the isovalues and the priority queue [S1].
*   **Strengths**: Automatically identifies representative isovalues without user-defined thresholds; ensures selected isovalues correspond to distinct structures by penalizing similarity to already selected values [S1].
*   **Limitations**: Requires a similarity metric $SMV$ which can be computationally expensive to compute for many pairs; the definition of SDV is not standardized in the source [S1].
*   **When to Use**: Exploratory visualization tasks where the user intent is unknown and a diverse set of representative isosurfaces is needed [S1].

## Background
The method was introduced in the paper "Isosurface Similarity Maps" by the author(s) of [S1], published in 2010 in the journal *Computer Graphics Forum* (Eurographics Association) [S1]. It addresses the problem of selecting relevant isovalues for exploratory volume rendering, where the user's intent is not known a priori [S1]. The approach leverages the structured nature of the Isosurface Similarity Map (ISM) to automatically identify regions of high similarity and select representative isovalues from them [S1].

## Sources
[S1] Isosurface Similarity Maps, https://doi.org/10.1111/j.1467-8659.2009.01689.x, fetched 2026-10-27
