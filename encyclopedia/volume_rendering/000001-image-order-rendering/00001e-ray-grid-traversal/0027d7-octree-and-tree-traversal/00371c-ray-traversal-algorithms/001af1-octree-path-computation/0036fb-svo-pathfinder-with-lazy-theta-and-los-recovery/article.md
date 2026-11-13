# SVO Pathfinder with Lazy Theta* and LOS Recovery

Also called: Lazy Theta* Pathfinding on SVO

## Overview
This method performs 3D pathfinding on a Sparse Voxel Octree (SVO) by combining A* search with Lazy Theta* logic, where line-of-sight (LOS) checks are performed optimistically during neighbor expansion to allow straight-line shortcuts, with a recovery step if LOS fails. It differs from standard A* on octrees by reducing path length and node visits through direct parent-to-current-node connections when possible, and differs from pure ray-casting pathfinding by caching collision results to avoid expensive per-step ray-marching.

## How it works
**Inputs and Outputs:**
- **Input:** Start and End world positions, Agent Radius, SVO structure (node array, depth, traversal data).
- **Output:** Sequence of nodes forming the path, or failure if no path exists.

**Data Structures:**
- **Node Array:** Flat array of SVO nodes. Each node has a cached collision test result field: `0` (untested), `-1` (collision), `+1` (no collision).
- **Cost Array:** Stores cumulative cost $g(n)$ for each node index.
- **Parent Array:** Stores parent node index for each node.
- **Open Array:** Priority queue of nodes to expand, ordered by $f(n) = g(n) + h(n)$.
- **Closed Array:** Boolean flag array indicating if a node has been expanded.
- **InlineAllocator:** Used for neighbor collection; behaves like a fixed allocator but switches to heap allocation if the set capacity is exceeded.

**Parameters:**
- **AgentRadius:** Radius of the agent. Clamped to prevent invalid access.
- **MaxDepth:** Maximum depth of the SVO. Typical values: 6 or 8.
- **AxisBit:** Bit mask for neighbor calculation. Points to 1 when Side is negative, because when the current node faces the positive direction, the neighboring node is adjacent in the negative direction.

**Steps:**
1. **Initialize and Validate:**
   - Clamp AgentRadius.
   - Get nodes at Start and End positions by accessing the node Array.
   - Verify traversability using cached collision test results. If Start or End nodes are not traversable (result $\neq +1$), return failure.
   - If Start and End are the same node, return immediately.

2. **Neighbor Collection:**
   - Iterate from 0 to 7 to collect neighbor indices based on target axis (0 or 1).
   - Traverse the tree using DFS to collect all leaf nodes.
   - Keep only passable nodes (collision test result $+1$).

3. **Pathfinding (Lazy Theta*):**
   - Initialize Cost, Parent, Open, and Closed arrays.
   - Add Start node to Open.
   - While Open is not empty:
     - Pop node $u$ with lowest $f(u)$ from Open.
     - If $u$ is End, reconstruct path and return.
     - Mark $u$ as closed.
     - For each neighbor $v$ of $u$:
       - If $v$ is closed or not passable, skip.
       - Calculate tentative $g(v)$ from $u$.
       - Perform LOS check between parent of $u$ (or $u$ if $u$ is Start) and $v$.
       - If LOS is clear:
         - Set parent of $v$ to parent of $u$ (or $u$ if $u$ is Start).
         - Update $g(v)$ using direct distance from parent of $u$ to $v$.
       - Else (LOS fails):
         - Set parent of $v$ to $u$.
         - Update $g(v)$ using distance from $u$ to $v$.
       - If $v$ is not in Open or new $g(v)$ is lower, update Open and Cost.

4. **LOS Method:**
   - Perform a simple collision test between two points.
   - Can be implemented with ray-marching using SVO, but simple collision test is used for lower code complexity with negligible performance loss.

**Pitfalls:**
- **Agent Radius vs Node Size:** Simple comparison of octree node size with AgentRadius is insufficient. Example: AgentRadius 700, octree alley sizes 200, 400, 200. It cannot pass through the 400-sized node despite the gap, requiring explicit collision testing.
- **Collision Test Cache:** Running a collision test every time is expensive. Only run it when AgentRadius is larger than the node size, and cache the result for each node.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_path(svo, start_pos, end_pos, agent_radius):
    # Initialize and validate
    agent_radius = clamp(agent_radius, min_radius, max_radius)
    start_node = get_node_at_position(svo, start_pos)
    end_node = get_node_at_position(svo, end_pos)
    
    if not is_traversable(start_node) or not is_traversable(end_node):
        return None
    
    if start_node == end_node:
        return [start_node]
    
    # Initialize arrays
    cost = array_zeros(svo.num_nodes)
    parent = array_zeros(svo.num_nodes)
    open_set = PriorityQueue()
    closed = array_zeros(svo.num_nodes, dtype=bool)
    
    open_set.push(start_node, heuristic(start_node, end_node))
    
    while not open_set.is_empty():
        u = open_set.pop()
        
        if u == end_node:
            return reconstruct_path(parent, u)
        
        closed[u] = True
        
        neighbors = collect_neighbors(svo, u, agent_radius)
        
        for v in neighbors:
            if closed[v] or not is_traversable(v):
                continue
            
            tentative_g = cost[u] + distance(u, v)
            
            # Lazy Theta* LOS check
            parent_u = parent[u] if parent[u] != -1 else u
            if line_of_sight(svo, parent_u, v):
                new_parent = parent_u
                new_g = cost[parent_u] + distance(parent_u, v)
            else:
                new_parent = u
                new_g = tentative_g
            
            if v not in open_set or new_g < cost[v]:
                cost[v] = new_g
                parent[v] = new_parent
                f = new_g + heuristic(v, end_node)
                open_set.push(v, f)
    
    return None

def collect_neighbors(svo, node, agent_radius):
    neighbors = []
    for i in range(8):
        # Calculate neighbor index based on target axis (0 or 1)
        # AxisBit points to 1 when Side is negative
        neighbor_idx = calculate_neighbor_index(node, i, svo.axis_bit)
        if is_valid_node(svo, neighbor_idx):
            # Traverse tree to collect leaf nodes
            leaves = traverse_tree(svo, neighbor_idx)
            for leaf in leaves:
                if is_traversable(leaf):
                    neighbors.append(leaf)
    return neighbors

def is_traversable(node):
    return node.collision_test_result == 1

def line_of_sight(svo, start_node, end_node):
    # Simple collision test
    return perform_collision_test(svo, start_node, end_node)
```

## Operating characteristics
- **Time Complexity:** Depends on SVO depth and branching factor. At MaxDepth 6, BuildSVO takes 700 microseconds, FindPath takes 300 microseconds. At MaxDepth 8, BuildSVO takes 14 milliseconds, FindPath takes 4 milliseconds.
- **Memory Complexity:** Linear with respect to the number of nodes in the SVO.
- **Performance:** Parallelization improves performance for simple repetitive tasks like BuildSVO. At MaxDepth 6, ParallelFor took 0.7ms, while a regular loop took 1.9ms.
- **Strengths:** Reduces path length and node visits through LOS checks. Caching collision results avoids expensive per-step ray-marching.
- **Limitations:** Code complexity increases with parallelization. Simple collision test may not be optimal for all scenarios.

## Background
This method was developed for Unreal Engine 5 (UE5) as a simple SVO pathfinder for 3D pathfinding. It was presented in a tutorial video by an unofficial source. The method leverages the SVO structure for efficient spatial queries and combines A* with Lazy Theta* for optimal pathfinding.

## Sources
[S1] UE5 Tutorial | Simple SVO Pathfinder for 3D Pathfinding, https://www.youtube.com/watch?v=QmDbX_gZSLA, fetched 2026-10-27 (unofficial)
