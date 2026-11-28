# Adaptive Octree Voxel Grid

## Overview
Adaptive Octree Voxel Grid is a dynamic volume representation that automatically refines and coarsens an octree structure based on simulation activity, such as entity density and water source presence [S1]. It distinguishes itself from static octrees by integrating a configurable `RefinementPolicy` that triggers structural changes during runtime, optimizing memory usage for sparse or dynamic scenes while maintaining a fallback to uniform grids for small worlds [S1].

## How it works
**Inputs and Outputs:**
The method takes a world configuration including dimensions, cell size, and a `RefinementPolicy` [S1]. It outputs a hierarchical grid structure that supports spatial queries (overlap, walk) and structural modifications (refine, coarsen) [S1].

**Data Structures:**
The core structure is `AdaptiveOctreeVoxelGrid`, which inherits from `OctreeVoxelGrid` [S1]. It maintains:
1. **Octree Nodes:** Standard octree nodes where leaves store voxel values [S1]. When a node is split, children inherit the parent's value [S1].
2. **Activity Tracking:** A dictionary `_activity_ticks: dict[int, int]` mapping node IDs to the last tick they were marked active [S1].
3. **Refinement Policy:** A `RefinementPolicy` dataclass defining thresholds [S1]:
   * `entity_density_threshold` (int, default 2): Refine if $\ge N$ entities are within bounds [S1].
   * `refine_water_sources` (bool, default True): Always refine regions overlapping water sources [S1].
   * `idle_coarsen_ticks` (int, default 100): Coarsen after $N$ ticks of inactivity [S1].
   * `max_depth` (int, default 4): Hard limit on refinement depth to bound memory [S1].

**Numbered Steps:**
1. **Initialization:** Create the grid with `dimensions`, `cell_size`, and `max_depth` derived from the policy. Initialize `_activity_ticks` as empty [S1].
2. **Entity Notification (`note_entity`):** Called every tick for moving entities [S1]. Determine the bounding box or leaf node containing the entity's position [S1]. Increment the activity count for relevant nodes in `_activity_ticks` [G]. Check refinement conditions: If `refine_water_sources` is True and the region overlaps a water source, mark for refinement [S1]. If the number of entities in the region $\ge$ `entity_density_threshold`, mark for refinement [S1]. If refinement is needed and current depth < `max_depth`, call `_refine_node` [S1].
3. **Refinement (`_refine_node`):** Split a leaf node into 8 children [S1]. Each child inherits the parent's voxel value [S1]. Update `_activity_ticks` for new children based on parent activity [G].
4. **Periodic Cleanup (`cleanup`):** Called every 50 ticks [S1]. Iterate through all leaf nodes [S1]. For each leaf, check if `current_tick - _activity_ticks[node_id]` $\ge$ `idle_coarsen_ticks` [S1]. If idle, check if the parent node has 8 identical-value children (including the current leaf) [S1]. If so, call `_try_coarsen_node` to merge them back into a single leaf [S1].
5. **Coarsening (`_try_coarsen_node`):** Verify all 8 siblings have identical values [S1]. Remove the 8 children and promote the parent to a leaf [S1]. Return `True` if coarsened, `False` otherwise [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
from typing import Optional, List, Tuple

@dataclass
class RefinementPolicy:
    entity_density_threshold: int = 2
    refine_water_sources: bool = True
    idle_coarsen_ticks: int = 100
    max_depth: int = 4

class AdaptiveOctreeVoxelGrid:
    def __init__(self, dimensions: Tuple[int, int, int], cell_size: float, policy: Optional[RefinementPolicy] = None):
        self.dimensions = dimensions
        self.cell_size = cell_size
        self.policy = policy or RefinementPolicy()
        self.max_depth = self.policy.max_depth
        self.root = self._create_root_node(dimensions, cell_size, depth=0)
        self._activity_ticks: dict[int, int] = {}
        self._next_node_id_counter: int = 0

    def _create_root_node(self, dims: Tuple[int, int, int], cs: float, depth: int) -> dict:
        return {"id": self._next_node_id_counter(), "depth": depth, "children": None, "value": None}

    def note_entity(self, position: Tuple[float, float, float], tick: int) -> None:
        """Called by engine each tick for mobile entities."""
        node = self._find_leaf_for_position(position)
        if node:
            self._activity_ticks[node["id"]] = tick
            self._check_refine(node, tick)

    def _check_refine(self, node: dict, tick: int) -> None:
        if node["depth"] >= self.max_depth:
            return
        entity_count = self._count_entities_in_node(node)
        is_water = self._is_overlapping_water(node)
        should_refine = False
        if self.policy.refine_water_sources and is_water:
            should_refine = True
        if entity_count >= self.policy.entity_density_threshold:
            should_refine = True
        if should_refine:
            self._refine_node(node)

    def _refine_node(self, node: dict) -> None:
        if node["children"] is not None:
            return
        parent_value = node["value"]
        node["children"] = []
        for _ in range(8):
            child = {
                "id": self._next_node_id_counter(),
                "depth": node["depth"] + 1,
                "children": None,
                "value": parent_value
            }
            node["children"].append(child)
            self._activity_ticks[child["id"]] = self._activity_ticks.get(node["id"], 0)

    def cleanup(self, tick: int) -> int:
        """Periodic coarsening pass. Returns number of nodes coarsened."""
        coarsened_count = 0
        leaves = self._get_all_leaves(self.root)
        for leaf in leaves:
            last_active = self._activity_ticks.get(leaf["id"], 0)
            if tick - last_active >= self.policy.idle_coarsen_ticks:
                parent = self._get_parent(leaf)
                if parent and self._try_coarsen_node(parent):
                    coarsened_count += 1
        return coarsened_count

    def _try_coarsen_node(self, parent: dict) -> bool:
        if parent["depth"] == 0:
            return False
        children = parent["children"]
        if len(children) != 8:
            return False
        first_value = children[0]["value"]
        all_leaves = all(c["children"] is None for c in children)
        all_same_value = all(c["value"] == first_value for c in children)
        if all_leaves and all_same_value:
            parent["children"] = None
            parent["value"] = first_value
            for c in children:
                self._activity_ticks.pop(c["id"], None)
            return True
        return False

    def _next_node_id_counter(self) -> int:
        nid = self._next_node_id_counter_val
        self._next_node_id_counter_val += 1
        return nid
    _next_node_id_counter_val: int = 0

    # [G] The sources do not provide the spatial query implementation.
    # Standard octree traversal logic would be inserted here.
    def _find_leaf_for_position(self, pos: Tuple[float, float, float]) -> Optional[dict]: return None
    def _count_entities_in_node(self, node: dict) -> int: return 0
    def _is_overlapping_water(self, node: dict) -> bool: return False
    def _get_all_leaves(self, node: dict) -> List[dict]: return []
    def _get_parent(self, node: dict) -> Optional[dict]: return None
```

**Edge Cases and Pitfalls:**
* **Thrashing:** Aggressive refinement policies can cause nodes to split and merge rapidly. Mitigated by `idle_coarsen_ticks` which prevents ping-pong behavior [S1].
* **Memory Overhead:** Octrees have higher overhead for small, dense worlds. The system defaults to `UniformVoxelGrid` and makes the octree opt-in via a world config flag [S1].
* **Interpolation Artifacts:** When splitting, children inherit the parent's value. This avoids interpolation artifacts for writes, but reads across boundaries may require care depending on the renderer [S1].

## Subtopics

### Adaptive and Large Volumetric Grid
A data structure using an array of deep octrees to support large, highly zoomable volumetric canvases, overcoming the limitations of uniform voxel editing engines [S3, unofficial]. It uses a root array of size $4^3$ (64) elements, where each root can be refined 24 times or higher, yielding an effective total octree depth of 26 [S3, unofficial]. This configuration supports a canvas size of $40 \text{ km}^3$ with a minimum voxel resolution of $0.3 \text{ mm}^3$ [S3, unofficial]. During painting, all octree roots that overlap with a brush stroke must be visited [S3, unofficial].

### Adaptive Meshing via Octree Subdivision
A technique to reconstruct a target object with adaptive mesh resolution by jointly optimizing mesh topology and an octree structure, starting from a low-resolution uniform voxel grid and subdividing cells based on objective loss [S4]. It begins optimization with a low-resolution uniform voxel grid, maintains a running average of the objective loss for each cell (averaged from all mesh vertices extracted from that cell), and subdivides cells if their objective loss exceeds a preset threshold of 0.04 [S4]. The process iterates until the adaptive mesh is obtained [S4]. It requires applying a specific constraint during optimization to ensure valid topology [S4]. This method is used in the context of the FlexiCubes method to achieve adaptive resolution without precomputed octree structures on ground truth geometry [S4].

## Operating characteristics
* **Time Complexity:** Refinement is $O(1)$ per node split. Cleanup is $O(N)$ where $N$ is the number of leaves, but runs periodically (every 50 ticks) [S1].
* **Memory Complexity:** Proportional to the number of active cells. Bounded by `max_depth` and the spatial extent of active entities [S1].
* **Performance:** Shows improvement over uniform grids in sparse worlds. Performance regression is possible in dense worlds, necessitating benchmarks and fallback to uniform grids [S1].
* **Strengths:** Automatic adaptation to activity; memory efficiency for sparse data.
* **Limitations:** Overhead for small worlds; potential for thrashing if policy is misconfigured.

## Background
The Adaptive Octree Voxel Grid was developed as part of the "multi-resolution voxel grid" project, specifically Phase 4, which focused on the octree implementation and refinement policy [S1]. It was introduced to address the need for dynamic memory management in simulation engines where entity distribution varies over time [S1].

## Sources
[S1] multi-resolution voxel grid: octree implementation + refinement policy (#64 phases 3-4), https://github.com/hellolifeforms/lila/issues/68, fetched 2026-10-26
[S2] Octree AMR in radmc3dPy, https://www.ita.uni-heidelberg.de/~dullemond/software/radmc-3d/manual_rmcpy/tutorial_octree_grid.html, fetched 2026-10-30
[S3] CanvoX: High-resolution VR Painting in Large Volumetric Canvas, https://archive.org/details/arxiv-1704.02724, fetched 2026-10-29 (unofficial)
[S4] Flexible Isosurface Extraction for Gradient-Based Mesh Optimization, https://arxiv.org/abs/2308.05371v1, fetched 2026-10-26
