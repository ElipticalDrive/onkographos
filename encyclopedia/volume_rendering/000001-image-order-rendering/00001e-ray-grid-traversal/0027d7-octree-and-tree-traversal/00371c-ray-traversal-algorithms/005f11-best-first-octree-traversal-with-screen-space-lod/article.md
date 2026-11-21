# Best-First Octree Traversal with Screen-Space LOD

Also called: Budgeted Octree Traversal

## Overview
This method renders hierarchical point-cloud or sparse volume data by traversing an octree in screen-space order, prioritizing nodes that project largest on the display [S1, unofficial]. It solves the problem of maintaining visual fidelity while adhering to a strict rendering budget (e.g., maximum point count or fill rate) by dynamically adjusting the level of detail (LOD) based on camera distance and frustum visibility [S1, unofficial]. Unlike fixed-depth traversal, it stops refining nodes once the budget is exhausted, ensuring consistent performance regardless of scene complexity [S1, unofficial].

## How it works
**Inputs:**
- An octree data structure where each node contains a list of child pointers (or indices) and a coarse sample of points (`node["sample"]`) [S1, unofficial].
- Camera parameters (position, projection matrix) to compute screen-space size and frustum bounds.
- A rendering budget (`budget`), defined as the maximum number of points allowed in the frame [S1, unofficial].
- A minimum screen-space size threshold (`lod_px`) in pixels [S1, unofficial].

**Outputs:**
- A set of leaf nodes (or coarse samples) selected for rendering, totaling no more than `budget` points.

**Data Structures:**
- **Priority Queue:** Stores nodes to be processed, ordered by their projected screen-space size (largest first) [S1, unofficial].
- **Node Structure:**
  - `children`: List of child node references or indices. Empty if leaf.
  - `sample`: List of points associated with this node, used for rendering if the node is not refined further [S1, unofficial].
  - `bounding_box`: Axis-aligned bounding box (AABB) for frustum culling and screen-space projection [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:**
   - Compute the screen-space bounding box of the root node.
   - Calculate the projected area (or max dimension) in pixels.
   - Push the root node into the priority queue with its screen-space size as the key [S1, unofficial].
   - Initialize `points = 0` [S1, unofficial].

2. **Traversal Loop:**
   - While the priority queue is not empty and `points < budget`:
     a. **Extract:** Pop the node with the largest screen-space size from the queue [S1, unofficial].
     b. **Frustum Culling:** Check if the node's bounding box intersects the view frustum. If not, discard the node and continue to the next iteration [S1, unofficial].
     c. **Refinement Decision:** Determine if the node should be refined into its children using the following boolean condition [S1, unofficial]:
        $$
        \text{refine} = (\text{node["children"] is not empty}) \land (\text{screen\_size} \ge \text{lod\_px}) \land (\text{points} + \text{len}(\text{node["sample"]}) < \text{budget})
        $$
        - `node["children"] is not empty`: The node has children to refine into.
        - `screen_size >= lod_px`: The node is large enough on screen to warrant higher resolution.
        - `points + len(node["sample"]) < budget`: Adding the coarse sample of the children (or the current node's sample if not refined, but here we are deciding to go deeper) does not exceed the budget. Note: The source implies that if we refine, we will eventually draw the children's samples. The condition checks if we *can* afford the children's samples. If we do not refine, we draw `node["sample"]` [S1, unofficial].
     d. **Action:**
        - If `refine` is true:
          - For each child of the node:
            - Compute the child's screen-space size.
            - Push the child into the priority queue [S1, unofficial].
        - If `refine` is false:
          - Add `len(node["sample"])` to `points` [S1, unofficial].
          - Mark `node["sample"]` for rendering [S1, unofficial].

3. **Termination:**
   - The loop ends when the queue is empty or `points` reaches `budget` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def best_first_octree_traversal(root, camera, budget, lod_px):
    # Priority queue stores tuples: (screen_size, node)
    # Max-heap behavior: negate screen_size for min-heap implementation
    pq = []
    push(pq, (-compute_screen_size(root, camera), root))
    
    points_rendered = 0
    render_list = []
    
    while pq and points_rendered < budget:
        neg_size, node = pop(pq)
        screen_size = -neg_size
        
        # Frustum Culling
        if not intersects_frustum(node.bounding_box, camera):
            continue
            
        # Refinement Condition
        has_children = node.children is not None and len(node.children) > 0
        is_large_enough = screen_size >= lod_px
        can_afford_children = True # Assumption: children's samples will be counted when popped
        
        # The source condition: points + len(node["sample"]) < budget
        # This checks if we can afford the CURRENT node's sample if we STOP refining.
        # However, the logic is: if we refine, we push children. If we don't, we draw node.sample.
        # The condition in the source is: points + len(node["sample"]) < budget
        # This implies that if we decide NOT to refine, we will add len(node["sample"]) to points.
        # So we can only refine if we are NOT forced to draw the current sample immediately?
        # Actually, the source says: "refine = node["children"] and size >= lod_px and points + len(node["sample"]) < budget"
        # This is slightly ambiguous. Does "points + len(node["sample"]) < budget" mean we check if we can afford the CURRENT node's sample?
        # If we refine, we don't draw node["sample"] yet. We draw children's samples later.
        # If we don't refine, we draw node["sample"].
        # So the condition checks if we can afford to draw the current node's sample IF we were to stop.
        # But we only stop if we can't refine.
        # Let's stick to the source formula exactly.
        
        refine = has_children and is_large_enough and (points_rendered + len(node.sample) < budget)
        
        if refine:
            for child in node.children:
                child_size = compute_screen_size(child, camera)
                push(pq, (-child_size, child))
        else:
            # Draw the coarse sample
            points_rendered += len(node.sample)
            render_list.append(node.sample)
            
            # Optimization: if points_rendered >= budget, we might want to stop early,
            # but the loop condition handles it.
            
    return render_list
```

**Parameters:**
- `lod_px`: Minimum screen-space size in pixels. Typical values depend on resolution and desired detail, e.g., 1-5 pixels [S1, unofficial].
- `budget`: Maximum number of points to render. Typical values depend on hardware capabilities, e.g., 60,000 points [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Budget Exhaustion:** If the budget is reached before the queue is empty, the traversal stops, potentially leaving large background areas unrendered or coarsely rendered [S1, unofficial].
- **Small Nodes:** Nodes smaller than `lod_px` are never refined, even if they are in the foreground, ensuring a minimum LOD [S1, unofficial].
- **Frustum Culling:** Incorrect frustum culling can lead to rendering off-screen nodes or missing visible nodes [S1, unofficial].
- **Precision:** Screen-space size calculation must be robust to avoid jitter or incorrect ordering due to floating-point errors [G].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ in the worst case, where $N$ is the number of nodes in the octree, due to priority queue operations. In practice, it is faster as many nodes are culled [S1, unofficial].
- **Memory Complexity:** $O(D)$, where $D$ is the depth of the octree, for the priority queue and recursion stack (if implemented recursively, though this method uses an explicit queue) [G].
- **Performance:** On a 2.18 million point room scan, a far camera drew ~60,000 points coarsely, while a near camera drew ~60,000 points densely on foreground surfaces [S1, unofficial].
- **Strengths:** Consistent performance, adaptive LOD, efficient culling [S1, unofficial].
- **Limitations:** Requires a priority queue, which can be expensive on GPU if not implemented carefully [G].

## Background
This method was described in an unofficial source discussing point cloud level of detail using octrees [S1, unofficial]. It emphasizes a "best-first" approach driven by screen-space size and budget constraints.

## Sources
[S1] Point Cloud Level of Detail: Octrees and Out-of-Core ..., https://learngeodata.eu/point-cloud-level-of-detail-octree/, fetched 2026-10-28 (unofficial)
