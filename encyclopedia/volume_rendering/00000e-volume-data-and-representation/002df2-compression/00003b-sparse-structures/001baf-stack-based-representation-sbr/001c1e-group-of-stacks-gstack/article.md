# Group of Stacks (gstack)
Also called: Gstack

## Overview
A gstack is a compact data structure that represents a rectangular region of vertical stacks in a voxel grid, where all stacks share an identical sequence of attribute values (intervals). It solves the problem of redundant storage for attribute information in sparse volume representations by grouping spatially adjacent stacks with identical semantic content. Unlike standard spatial decompositions that subdivide solely based on geometric variation, gstacks compress attribute information while storing geometry as a set of non-intersecting heightfields, allowing for separate compression of the height data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A rectangular region of stacks $S$ defined by coordinates $[x_{min}, y_{min}, x_{max}, y_{max}]$ in the $xy$-plane, where every stack $S_{x,y}$ in this region has the same number of intervals $n$ and the same sequence of attribute values $a_1, a_2, \dots, a_n$.
*   **Output:** A gstack structure $G$ encoding this region.

**Data Structures:**
A gstack is encoded as a sequence of $n$ intervals $i_1, i_2, \dots, i_n$. Each interval $i_k$ is a tuple:
$$ i_k = \langle a_k, H_k \rangle $$
Where:
*   $a_k$: The attribute value (scalar or vector) common to all stacks in the region for the $k$-th interval.
*   $H_k$: A heightfield (2D array) with dimensions $(x_{max} - x_{min} + 1) \times (y_{max} - y_{min} + 1)$.

**Layout and Mapping:**
The heightfield $H_k$ stores the geometric height of the $k$-th interval for each stack in the region. The height $h_{x,y}$ of a specific stack $S_{x,y}$ is mapped to the heightfield coordinate relative to the region's minimum:
$$ H_k[x - x_{min}, y - y_{min}] = h_{x,y}^{(k)} $$
where $h_{x,y}^{(k)}$ is the height of the $k$-th interval in stack $S_{x,y}$.

**Constraints:**
The intervals are consistently ordered for all stacks in the region. Consequently, the heightfields never intersect. For any coordinate $(i, j)$ within the heightfield dimensions, and for any $k$ such that $1 \le k < n$:
$$ H_k[i, j] < H_{k+1}[i, j] $$
This strict ordering ensures that the intervals are vertically stacked without overlap [S1, unofficial].

**Algorithm: Interval Propagation (for merging gstacks):**
When combining gstacks (e.g., during QuadStack construction), an algorithm finds a common mapping to maximize terminal intervals.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interval_propagation(gstack1, gstack2):
    """
    Finds a matching sequence of interval attribute values for two gstacks.
    Input: Two gstacks, each with a list of intervals.
    Output: Two new gstacks with matching terminal intervals and * -intervals for gaps.
    """
    # Base case: if either gstack has no intervals, return empty results
    if not gstack1.intervals or not gstack2.intervals:
        return [], []

    best_solution = None
    max_terminal_count = -1

    # Iterate over all possible pairs of intervals to find matches
    for s in range(len(gstack1.intervals)):
        for t in range(len(gstack2.intervals)):
            i1_s = gstack1.intervals[s]
            i2_t = gstack2.intervals[t]

            # Check if attribute values match
            if i1_s.attribute == i2_t.attribute:
                # Create result lists
                res1_intervals = []
                res2_intervals = []

                # Add the matching terminal intervals
                res1_intervals.append(i1_s)
                res2_intervals.append(i2_t)

                # Handle predecessors if they exist (s > 0 and t > 0)
                if s > 0 and t > 0:
                    # Group predecessors into wildcard intervals (*)
                    # Associated with the heightfields of the top intervals i1_1 and i2_1
                    # Note: The specific heightfield mapping for * intervals is handled
                    # by the parent context or subsequent packing steps.
                    # Here we mark them as wildcard intervals.
                    pred1 = WildcardInterval(heightfield=i1_s.heightfield) # Placeholder for logic
                    pred2 = WildcardInterval(heightfield=i2_t.heightfield) # Placeholder for logic
                    res1_intervals.insert(0, pred1)
                    res2_intervals.insert(0, pred2)

                # Recursively process remaining intervals
                remaining_g1 = Gstack(intervals=gstack1.intervals[s+1:])
                remaining_g2 = Gstack(intervals=gstack2.intervals[t+1:])
                
                # Recursive call
                sub_res1, sub_res2 = interval_propagation(remaining_g1, remaining_g2)
                
                # Combine results
                current_res1 = res1_intervals + sub_res1
                current_res2 = res2_intervals + sub_res2

                # Count terminal intervals (non-wildcard)
                count1 = sum(1 for i in current_res1 if not i.is_wildcard)
                count2 = sum(1 for i in current_res2 if not i.is_wildcard)
                total_terminals = count1 + count2

                # Keep the best solution
                if total_terminals > max_terminal_count:
                    max_terminal_count = total_terminals
                    best_solution = (current_res1, current_res2)

    return best_solution if best_solution else ([], [])
```
*Note: The theoretical complexity is $O((m!)^4)$ for four stacks with $m$ intervals, but heuristics improve practical performance [S1, unofficial].*

**Edge Cases and Pitfalls:**
*   **Non-square Quadrants:** In hierarchical constructions (like QuadStack), gstack regions are not guaranteed to be square or powers of two. Heightfield dimensions must be calculated exactly as $(x_{max} - x_{min} + 1) \times (y_{max} - y_{min} + 1)$ [S1, unofficial].
*   **Wildcard Intervals:** When intervals are propagated up or down a hierarchy, gaps are represented by wildcard intervals. The heightfield of a wildcard interval $H_w$ must correctly map to quadrants of descendant or ancestor heightfields. If $i_w$ represents intervals $i_k \dots i_{k+j}$ in a descendant, $H_w$ combines the heightfields of the top intervals of all sets in descendants grouped by $i_w$ [S1, unofficial].

## Subtopics

### Wildcard Interval (*-interval)
A special type of interval used in internal nodes of hierarchical structures (like QuadStack) to represent gaps between terminal intervals. It corresponds to intervals stored elsewhere (in descendant or ancestor nodes). Defined as $i_w = \langle *, H_w \rangle$, it has an associated heightfield $H_w$. If $i_w$ represents intervals in a descendant, $H_w$ combines the heightfields of the top intervals of those descendants. If it refers to an ancestor, $H_w$ is a quadrant of the ancestor's heightfield. This allows flexible gstacks that combine terminal intervals with areas lacking information at the current level [S1, unofficial].

### Interval Propagation Algorithm
A brute-force algorithm used to find a common mapping for two gstacks that maximizes the number of terminal intervals. It tests every pair of intervals from both gstacks for attribute matches. If a match is found, the intervals are added to the result, and the function recurses on the remaining intervals. Predecessor intervals are grouped into wildcard intervals if they are not at the start. The solution with the highest number of terminal intervals is returned. While theoretically exponential, it performs well with heuristics [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The core gstack encoding is linear in the number of intervals $n$ and the area of the region. The Interval Propagation algorithm has a theoretical complexity of $O((m!)^4)$ for merging four stacks with $m$ intervals, but practical performance is significantly better due to heuristics [S1, unofficial].
*   **Memory Complexity:** Space-efficient for attribute information. The size is proportional to $n \times (\text{size}(a) + \text{size}(H))$, where $\text{size}(H)$ is the area of the heightfield. Geometry is stored as non-intersecting heightfields, which can be further compressed using standard heightfield encoding methods [S1, unofficial].
*   **Strengths:** Compresses redundant attribute data across spatially adjacent stacks. Allows separate compression of geometry (heightfields). Flexible integration with hierarchical structures via wildcard intervals.
*   **Limitations:** Requires identical attribute sequences for all stacks in the region. Merging gstacks can be computationally expensive without heuristics. Not suitable for regions with high attribute variation.

## Background
The gstack was introduced as part of the QuadStack representation, an efficient method for the direct rendering of layered datasets. It builds on the Stack-Based Representation (SBR) by adding spatial grouping. The concept was proposed in the paper "QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets" by Graciano et al. (2021) [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
