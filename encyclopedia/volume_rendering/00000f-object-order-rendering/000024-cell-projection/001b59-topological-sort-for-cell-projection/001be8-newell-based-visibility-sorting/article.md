# Newell-based Visibility Sorting

Also called: Visibility Sorting (Newell-based)

## Overview
Newell-based Visibility Sorting is an algorithm to sort cells into a back-to-front ordered list for correct hardware compositing [S1, S2][S3, unofficial]. It extends the Newell, Newell, and Sancha polygon sorting method to the case of polyhedra [S1, S2][S3, unofficial]. By definition, if cell A partially occludes cell B from a particular viewpoint V, cell B must come before cell A in the list [S1, S2][S3, unofficial]. It differs from its siblings by using a general iterative occlusion testing strategy rather than topological graph traversal (as in [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)) or per-pixel depth buffers (as in [Per-Pixel Sorting (General Variant)](1061-per-pixel-sorting-general-variant.md)).

## How it works
**Inputs and outputs:**
- Input: A list of $n$ cells, each defined by its vertices and geometry.
- Output: A sorted list of cells in back-to-front order.

**Data structures:**
- `cells`: List of cell objects.
- `initial_list`: List of cell indices, initially sorted by rear-most depth.
- `output_list`: List of cell indices, built incrementally.
- `moved_flag`: Boolean array of size $n$, initialized to false.

**Steps:**
1. **Initial Sort:** Sort cells from back to front by the depth coordinate ($z$) of their rear-most vertex [S1, S2][S3, unofficial]. Let this ordered list be `initial_list`.
2. **Iterative Testing:** While `initial_list` is not empty:
   a. Take the rearmost cell $A$ from `initial_list` [S1, S2][S3, unofficial].
   b. Test cell $A$ against all other cells $B$ in `initial_list` (excluding $A$) [S1, S2][S3, unofficial].
   c. If cell $A$ does not occlude any other cell $B$:
      i. Remove $A$ from `initial_list` [S1, S2][S3, unofficial].
      ii. Append $A$ to `output_list` [S1, S2][S3, unofficial].
   d. If cell $A$ occludes a cell $B$:
      i. Move $B$ to the rearmost position on `initial_list` [S1, S2][S3, unofficial].
      ii. Mark $B$ as having been moved (set `moved_flag[B] = true`) [S1, S2][S3, unofficial].
      iii. Continue testing with $B$ as the new current cell $A$ (i.e., do not remove $B$ yet, but re-evaluate it against the remaining cells in `initial_list`) [S1, S2][S3, unofficial].
      iv. If $B$ is moved a second time (i.e., `moved_flag[B]` was already true), it indicates a visibility cycle [S1, S2][S3, unofficial]. In this case, no visibility sort is possible unless one of the cells in the cycle is subdivided [S1, S2][S3, unofficial].

**Occlusion Test:**
Testing pairs of cells for mutual occlusion is the most expensive part [S1, S2][S3, unofficial]. A sequence of tests of increasing difficulty is used to eliminate the possibility of occlusion early [S1, S2][S3, unofficial]. The specific geometric tests are not detailed in the provided sources, but the hierarchy implies simple bounds checks first, followed by more complex intersection tests [S1, S2][S3, unofficial].

**Parameters:**
- Not given in the sources.

**Edge cases and pitfalls:**
- **Visibility Cycles:** Occur when cells mutually occlude each other [S2][S3, unofficial]. No visibility sort is possible unless one of the cells in the cycle is subdivided [S2][S3, unofficial].
- **Complexity:** The algorithm performs $O(n^2)$ pair-wise occlusion tests for a data volume of $n$ cells [S1, S2][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def newell_visibility_sort(cells):
    n = len(cells)
    # Step 1: Initial sort by rear-most vertex depth
    # Assuming cell has a method get_rear_depth()
    initial_list = sorted(range(n), key=lambda i: cells[i].get_rear_depth(), reverse=True)
    output_list = []
    moved_flag = [False] * n
    
    while initial_list:
        # Step 2a: Take rearmost cell A
        a_idx = initial_list[0]
        a = cells[a_idx]
        
        occluded_b_idx = None
        for b_idx in initial_list[1:]:
            b = cells[b_idx]
            # Step 2b: Test if A occludes B
            if a.occludes(b):
                occluded_b_idx = b_idx
                break
        
        if occluded_b_idx is None:
            # Step 2c: A does not occlude any cell
            initial_list.pop(0)
            output_list.append(a_idx)
        else:
            # Step 2d: A occludes B
            b_idx = occluded_b_idx
            if moved_flag[b_idx]:
                # Step 2d.iv: Visibility cycle detected
                raise ValueError("Visibility cycle detected. Subdivision required.")
            
            # Step 2d.ii: Mark B as moved
            moved_flag[b_idx] = True
            
            # Step 2d.i: Move B to rearmost position
            # Remove B from its current position
            initial_list.remove(b_idx)
            # Append B to the end (rearmost)
            initial_list.append(b_idx)
            
            # Step 2d.iii: Continue testing with B
            # The loop continues, and the next iteration will pick B as the new 'A'
            # because B is now at the front of initial_list? 
            # Note: The description says "move B to the rearmost position... and testing proceeds with it".
            # If B is moved to the *rearmost* position, it is at the end of the list.
            # However, the algorithm says "testing proceeds with it instead".
            # This implies we need to re-evaluate B immediately.
            # If B is at the end, the next iteration would pick the *next* cell in the list.
            # To proceed with B, we must ensure B is picked next.
            # The text says "move B to the rearmost position... and testing proceeds with it".
            # This is slightly ambiguous. Usually, "rearmost" means furthest back (end of list).
            # But if we proceed with B, B must be the next candidate.
            # Let's re-read: "B is moved to the rearmost position on the initial list and the testing proceeds with it instead."
            # If B is at the rearmost position, it is at the end. The next cell to test is the one at index 0.
            # This contradicts "proceeds with it".
            # Alternative interpretation: "rearmost" in the context of the *remaining* list might mean the one that is currently being tested?
            # No, "rearmost" usually refers to depth.
            # Let's look at the standard Newell sort logic.
            # Standard Newell sort: If A occludes B, B is moved to the front of the list (to be tested next)?
            # Or B is moved to the back?
            # The text says "rearmost position".
            # If B is moved to the rearmost position, it is at the end.
            # If testing proceeds with B, B must be at the front.
            # There is a contradiction in the text or my understanding of "rearmost".
            # Let's assume "rearmost" means the position corresponding to the current test candidate?
            # No, "rearmost position on the initial list" implies a specific slot.
            # Let's look at [S1]: "B is moved to the rearmost position on the initial list and the testing proceeds with it instead."
            # If B is at the rearmost position, it is the last element.
            # If we proceed with B, we must pick B.
            # Perhaps "rearmost" means the position *just removed*? No.
            # Perhaps the list is ordered front-to-back, and "rearmost" is the end?
            # If the list is back-to-front (rearmost first), then index 0 is rearmost.
            # If B is moved to the rearmost position, it goes to index 0.
            # Then the next iteration picks B. This makes sense.
            # So "rearmost position" means index 0.
            
            # Correction: The initial sort is "back to front". So index 0 is the rearmost (furthest back).
            # If B is moved to the rearmost position, it is moved to index 0.
            # Then the next iteration picks B.
            
            # Remove B from its current position
            initial_list.remove(b_idx)
            # Insert B at the beginning (rearmost position)
            initial_list.insert(0, b_idx)
            
            # Continue testing with B (it is now at index 0)
            # The loop will naturally pick it up in the next iteration.
            pass
            
    return output_list
```

## Subtopics

### [Exact Visibility Ordering for Cell Projection Volume Rendering (SXMPVO)](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md)
An exact method for visibility ordering. Choose when exactness is required and performance is secondary.

### [A-buffer visibility sort for non-convex grids](1057-a-buffer-visibility-sort-for-non-convex-grids.md)
Handles non-convex grids using A-buffers. Choose for non-convex geometries.

### [Power Distance Sort](1060-power-distance-sort.md)
Uses power distance for sorting. Choose for specific distance-based ordering needs.

### [Per-Pixel Sorting (General Variant)](1061-per-pixel-sorting-general-variant.md)
Sorts per-pixel. Choose for high accuracy at the cost of performance.

### [XMPVO](1363-xmpvo.md)
An extension of SXMPVO. Choose for convex grids with specific requirements.

### [MPVONC](1377-mpvonc.md)
A variant for non-convex cells. Choose for non-convex cell structures.

### [Convex Grid Topological Visibility Sorting](2370-convex-grid-topological-visibility-sorting.md)
An O(n) topological sort for convex grids. Choose for convex grids where performance is critical.

## Operating characteristics
- **Time Complexity:** $O(n^2)$ pair-wise occlusion tests for a data volume of $n$ cells [S1, S2][S3, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** The quadratic complexity makes it inefficient for large $n$.
- **Strengths:** General applicability to polyhedra; simple logic.
- **Limitations:** $O(n^2)$ complexity; visibility cycles require subdivision.
- **When to use:** Small to medium datasets where general polyhedral support is needed.
- **When not to use:** Large datasets or convex grids where faster $O(n)$ methods like [Convex Grid Topological Visibility Sorting](2370-convex-grid-topological-visibility-sorting.md) are available.

## Background
The Newell, Newell, and Sancha sort [11] for polygons was extended to the case of polyhedra in [12] [S1, S2][S3, unofficial]. It is used for correct back-to-front hardware compositing [S1, S2][S3, unofficial].

## Sources
[S1] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S2] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
