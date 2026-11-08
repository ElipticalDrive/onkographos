# X-cube loop connection criteria

Also called: Criterion a, Criterion b, Criterion c, Criterion d

## Overview
X-cube loop connection criteria determine how to resolve topological ambiguities within X-cubes after X-faces have been processed, specifically by deciding whether to connect remaining free loops [S1]. Unlike [X-face slashing criteria](1777-x-face-slashing-criteria.md), which merge loops across cube faces, these criteria operate on loops contained entirely within an X-cube [S1]. The choice of criterion directly influences the topological invariants of the resulting mesh, specifically the number of shells ($S$) and handles ($H$), allowing the algorithm to prioritize either mesh simplicity (fewer shells) or feature preservation (more shells) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The set of free loops identified within X-cubes after X-face processing [S1]. Each loop is associated with an equivalence class identifier [S1].
*   **Output:** A decision for each pair of connectable loops in an X-cube: connect or do not connect [S1].

**Data Structures:**
*   **Equivalence Class Map:** A mapping from loop identifier to an integer or unique ID representing its topological equivalence class [S1].
*   **Loop List:** A list of loops currently residing in the X-cube [S1].

**Mathematical Context:**
The criteria are derived from the relationship between the number of triangles ($T$), vertices ($V$), handles ($H$), and shells ($S$):
$$ T = 2V + 4(H - S) $$
Connecting two loops within an X-cube results in a net increase of $T$ by 4 [S1]. Consequently, the term $(H - S)$ increases by 1 [S1]. The criteria differ in how they distribute this increase between $H$ and $S$ [S1].

**Steps:**
1. Identify all free loops within the current X-cube [S1].
2. Determine the equivalence class for each loop [S1].
3. Apply the selected criterion (a, b, c, or d) to decide whether to connect pairs of loops [S1].

**Criteria Definitions:**

*   **Criterion a (Never connect):**
    *   **Logic:** Loops are never connected [S1].
    *   **Effect:** $(H - S)$ is decreased [S1]. This decision tends to generate many small blobs (disconnected shells) [S1]. In case of noisy models, irrelevant small features can be easily identified and removed [S1].

*   **Criterion b (Always connect):**
    *   **Logic:** Loops are always connected [S1].
    *   **Effect:** $(H - S)$ is increased and either $S$ is decreased or $H$ is increased [S1]. At the end we will have a lower number of equivalence classes and a small $S$ [S1].

*   **Criterion c (Connect if same class):**
    *   **Logic:** Two loops are connected when they belong to the same equivalence class [S1].
    *   **Effect:** $S$ remains constant [S1]. As we have an increase in $H - S$, the final consequence is an increase of $H$ by one [S1].

*   **Criterion d (Connect if different class):**
    *   **Logic:** Two loops are connected when they do not belong to the same equivalence class [S1].
    *   **Effect:** $S$ is reduced by one [S1]. As we have an increase in $H - S$ of also one, $H$ remains constant [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_x_cube_criteria(loops, equivalence_classes, criterion):
    """
    Applies the specified X-cube loop connection criterion.
    
    Args:
        loops: List of loop objects in the X-cube.
        equivalence_classes: Dict mapping loop_id -> class_id.
        criterion: String 'a', 'b', 'c', or 'd'.
        
    Returns:
        List of connections (pairs of loop_ids) to perform.
    """
    connections = []
    
    # Generate all unique pairs of loops
    for i in range(len(loops)):
        for j in range(i + 1, len(loops)):
            loop_a = loops[i]
            loop_b = loops[j]
            class_a = equivalence_classes[loop_a.id]
            class_b = equivalence_classes[loop_b.id]
            
            should_connect = False
            
            if criterion == 'a':
                # Criterion a: Never connect loops
                should_connect = False
                
            elif criterion == 'b':
                # Criterion b: Always connect loops
                should_connect = True
                
            elif criterion == 'c':
                # Criterion c: Connect if same equivalence class
                should_connect = (class_a == class_b)
                
            elif criterion == 'd':
                # Criterion d: Connect if different equivalence class
                should_connect = (class_a != class_b)
                
            if should_connect:
                connections.append((loop_a.id, loop_b.id))
                
    return connections
```

**Edge Cases and Pitfalls:**
*   **Equivalence Class Dependency:** Criteria c and d explicitly rely on equivalence class membership to decide connections [S1]. Incorrect class assignment leads to topological errors [G].
*   **Processing Order:** These criteria must be applied after X-face slashing is complete, as they operate on the remaining free loops within X-cubes [S1]. If X-faces are not processed first, the loop structure within X-cubes may be ill-defined [G].
*   **Topological Impact:** Criterion a decreases $(H - S)$, while criteria b, c, and d increase $(H - S)$ by one per connection [S1].

## Subtopics

### [Combined optimization strategies](2147-combined-optimization-strategies.md)
Combined optimization strategies are four specific combinations of X-face slashing and X-cube loop connection criteria used to optimize the topological and combinatorial properties of the resulting isosurface mesh. They resolve the ambiguity in how to handle X-faces and X-cubes by prioritizing specific mesh metrics: minimizing triangles ($T$), maximizing triangles, maximizing shells ($S$), or quasi-minimizing shells. These strategies are applied after the initial X-face identification and are distinct from sibling methods that do not explicitly target these topological invariants.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Provides explicit control over the topological complexity (shells and handles) of the output mesh [S1].
*   **Limitations:** Does not address geometric accuracy, only topology [G].
*   **When to use:** Use Criterion a for noisy data or when post-processing small features is desired [S1]. Use Criterion b, c, or d when a smoother, more connected surface with fewer shells is preferred [S1].

## Background
The criteria were introduced in the context of optimizing the topological and combinatorial complexity of isosurfaces [S1]. They are part of a broader framework for handling ambiguous cases in marching cubes-like algorithms, specifically focusing on the resolution of X-cubes after X-faces have been processed [S1].

## Sources
[S1] Optimizing the topological and combinatorial complexity of isosurfaces, https://doi.org/10.1016/j.cad.2004.09.013, fetched 2026-10-30
