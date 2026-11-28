# Simple Exploration Path Construction
Also called: Inspection Walkthrough Path Construction
## Overview
An algorithm for generating an inspection walkthrough path for complex volume models, ensuring all representative views (keypoints) are visited [S1]. It constructs a minimal length path on a bounding sphere of the object or region of interest [S1]. It sets itself apart from siblings like [B-Spline Camera Path Interpolation](11453-b-spline-camera-path-interpolation.md) by using an exhaustive search for the shortest path through a small set of discrete candidates rather than continuous curve fitting, and from [Improved Exploration Path Construction](#improved-exploration-path-construction) by not deviating from the shortest path to maximize local information [S1].
## How it works
**Inputs and Outputs:**
- Input: A set of $N$ representative viewpoints (keypoints) selected from the volume, typically $N \in \{3, 4\}$ [S1].
- Input: The viewpoint with the highest entropy, denoted $B_0$ [S1].
- Input: A bounding sphere enclosing the object or region of interest [S1].
- Output: An ordered sequence of camera positions on the bounding sphere that visits all keypoints [S1].

**Data Structures:**
- `keypoints`: A list of $N$ 3D positions on the bounding sphere.
- `best_view`: The index or position of $B_0$ within `keypoints`.
- `path`: A list of 3D positions representing the camera trajectory.

**Steps:**
1. **Identify Key Points:** Use the important viewpoints selected in the previous section as key points [S1]. Let the set of key points be $K = \{k_1, k_2, ..., k_N\}$ [S1].
2. **Identify Best View:** Determine the viewpoint with the highest entropy, $B_0$, from $K$ [S1].
3. **Enforce Constraint:** Ensure $B_0$ is visited in the second position of the path [S1]. This allows the camera to visit the surroundings of $B_0$, which are intuitively more informative [S1].
4. **Compute Minimal Path:** Compute the minimal length path that passes through all representative views in $K$, subject to the constraint that $B_0$ is second [S1].
5. **Exhaustive Search:** Since there are only three to four candidates ($N \le 4$), an exhaustive search of all permutations of the remaining $N-1$ viewpoints is performed to find the shortest path [S1].
6. **Construct Path:** The resulting sequence of positions forms the simple exploration path [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simple_exploration_path(keypoints, best_view):
    """
    Generates a simple exploration path visiting all keypoints.
    The best_view (highest entropy) must be visited second.
    
    Args:
        keypoints: List of 3D positions on the bounding sphere.
        best_view: The 3D position of the viewpoint with highest entropy.
        
    Returns:
        List of 3D positions representing the camera path.
    """
    # Step 1 & 2: Identify key points and best view
    # keypoints is already provided as input
    # best_view is already provided as input
    
    # Step 3: Enforce constraint - best_view is second
    # Remove best_view from the list of remaining viewpoints to visit
    remaining_views = [k for k in keypoints if k != best_view]
    n = len(remaining_views)
    
    # Step 4 & 5: Compute minimal path via exhaustive search
    # We need to order the remaining_views to minimize total path length
    # The path structure is: [Start, best_view, ...remaining_views...]
    # However, the problem states "minimal length path that passes through all... 
    # ensuring best view in second place". This implies the path starts at one 
    # of the remaining views, goes to best_view, then visits the rest.
    # Or it implies a fixed start? The text says "minimal length path... passes 
    # through all". Usually, this is a Traveling Salesperson Problem variant.
    # With N<=4, we can brute force permutations of the remaining views.
    
    import itertools
    import math
    
    def distance(p1, p2):
        return math.sqrt(sum((a-b)**2 for a, b in zip(p1, p2)))
    
    def path_length(path):
        total = 0.0
        for i in range(len(path) - 1):
            total += distance(path[i], path[i+1])
        return total

    best_length = float('inf')
    best_permutation = None
    
    # Generate all permutations of the remaining views
    for perm in itertools.permutations(remaining_views):
        # Construct candidate path: Start with first of perm, then best_view, then rest
        # Note: The text doesn't specify the start point explicitly, but "minimal length 
        # path that passes through all" usually implies finding the optimal ordering.
        # If the start is not fixed, the first element of the permutation is the start.
        candidate_path = list(perm) + [best_view] # This order is wrong based on constraint
        
        # Constraint: best_view is SECOND.
        # So the path is: [First of perm, best_view, Rest of perm]
        # Wait, if perm is (A, B, C), and best is X.
        # Path: A -> X -> B -> C ?
        # Or is the start fixed? "The viewpoint with the highest entropy is visited in second place".
        # This implies the path has a start. Let's assume the path starts at one of the 
        # remaining views.
        
        # Let's re-read carefully: "minimal length path that passes through all... 
        # ensures we are providing a high amount of information... best view in second place".
        # If N=3 (A, B, C) and B is best.
        # Permutations of remaining {A, C}: (A, C) or (C, A).
        # Path 1: A -> B -> C. Length: d(A,B) + d(B,C).
        # Path 2: C -> B -> A. Length: d(C,B) + d(B,A).
        # We choose the shorter one.
        
        # Generalizing for N views:
        # Path = [perm[0], best_view, perm[1], ..., perm[N-2]]
        
        current_path = [perm[0], best_view] + list(perm[1:])
        current_len = path_length(current_path)
        
        if current_len < best_length:
            best_length = current_len
            best_permutation = current_path
            
    # Step 6: Return the best path
    return best_permutation
```

**Parameters:**
- Number of candidate viewpoints: Typically three to four [S1].

**Edge Cases and Pitfalls:**
- If $N < 2$, the constraint "best view in second place" is undefined. The sources assume $N \ge 3$ ("three to four candidates") [S1].
- Precision issues in distance calculations on the sphere are not discussed, but standard Euclidean distance on 3D coordinates is implied [S1].
- The algorithm assumes the bounding sphere is already defined and the keypoints lie on it [S1].

## Subtopics

### Improved Exploration Path Construction
An improvement to the simple path where the camera deviates from the short path to maximize information gathered, evaluated by entropy [S1]. For each step, it analyzes the entropy of three candidate positions placed toward the destination point, separated by $\pm 4$ to $6$ degrees [S1]. It chooses the candidate with the highest entropy [S1]. The allowed total deviation is limited and reduced as the camera gets closer to the next key point to ensure all keypoints are visited [S1]. The speed of the camera is reduced when visiting points nearby the best view ($B_0$) because they show a higher amount of valuable information [S1]. This approach gathers a higher amount of details than the simple method [S1]. Time depends on the number of steps produced, as it requires the evaluation of entropy at each step [S1].

## Operating characteristics
- **Time Complexity:** The simple path calculation time is negligible due to the exhaustive search over a small number of candidates ($N \le 4$) [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** The simple path is described as having "fast response" and "ease of use" [S1].
- **Strengths:** Provides a global vision of complex models by visiting all representative views; fast computation; no parameter definition required [S1].
- **Limitations:** Does not actively maximize information gain along the path (unlike the improved variant); relies on pre-selected representative views [S1].
- **When to use:** When a quick, comprehensive inspection of a volume model is needed without the computational cost of entropy evaluation at every step [S1].
- **When not to use:** When maximizing local detail and information gain is critical, in which case the Improved Exploration Path Construction should be used [S1].

## Background
Originated in the context of medical volume visualization, specifically for radiologists inspecting complex anatomical structures like kidneys and thorax [S1]. The method was proposed to address the insufficiency of a static set of views for providing a global vision of complex models [S1]. It is designed to be easily incorporated into daily work by radiologists with null cost [S1]. The technique is applicable to other rendering metaphors beyond medical imaging [S1].

## Sources
[S1] Representative Views and Paths for Volume Models, https://doi.org/10.1007/978-3-540-85412-8_10, fetched 2026-10-26
