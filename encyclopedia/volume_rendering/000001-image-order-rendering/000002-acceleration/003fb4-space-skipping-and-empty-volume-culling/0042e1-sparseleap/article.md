# SparseLeap

Also called: Empty Space Skipping via Rasterized Ray Segment Lists, SparseLeap-an effective empty space skipping technique

## Overview
SparseLeap is a hybrid object-order (rasterization) and image-order (ray-casting) method for efficient empty space skipping in large, sparse volumes with fine structures [S1, unofficial]. It solves the problem of space fragmentation and hierarchy traversal overhead inherent in standard octree-based skipping by decoupling empty space detection from volume sampling [S1, unofficial]. Unlike siblings that traverse a tree per ray, SparseLeap rasterizes occupancy geometry into per-pixel linked lists of ray segments, allowing the ray-caster to leap over empty space via simple linear list traversal [S1, unofficial].

## How it works
SparseLeap operates in four main phases: building an occupancy histogram tree, generating occupancy geometry, rasterizing ray segment lists, and performing ray-casting with leaping [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Volume data (scalar field and/or segmentation labels), transfer function, active object set, camera parameters.
*   **Outputs:** Rendered image, updated occupancy histogram tree (if grown/pruned).

### Data Structures

**1. Occupancy Histogram Tree (CPU)**
A spatial subdivision tree where each node represents a volume region.
*   **Leaf Nodes:** Store a single occupancy class: `empty`, `non-empty`, or `unknown`.
*   **Non-Leaf Nodes:** Store an occupancy histogram, which is an array of three integer counts: `[count_empty, count_non_empty, count_unknown]`. This histogram is the sum of the histograms of its children [S1, unofficial].
*   **Meta-data:** Leaf nodes may store min/max scalar values and a list of contained objects for culling [S1, unofficial].

**2. Occupancy Geometry (GPU)**
A set of nested bounding boxes extracted from the tree.
*   **Structure:** Bounding boxes with an associated occupancy class.
*   **Generation Rule:** A box is emitted only when the occupancy class changes while traversing down the tree from the root. Smaller boxes override larger containing boxes over their extent [S1, unofficial].

**3. Ray Segment Lists (GPU)**
Per-pixel linked lists storing 1D intervals along the ray.
*   **Buffer Layout:** Two arrays per pixel resolution.
    *   **List Elements:** 12 bytes each. Contains: `start_t` (float), `end_t` (float), `next_index` (uint32). The `next_index` points to the next element in the list or a sentinel value (e.g., -1) if it is the last element [S1, unofficial].
    *   **List Heads:** 16 bytes each. Contains: `head_index` (uint32), `padding` (uint32) [S1, unofficial].
*   **Content:** Each segment corresponds to one of the three occupancy classes. Consecutive segments of the same class are merged into a single segment during rasterization [S1, unofficial].

### Numbered Steps

**Step 1: Build/Update Occupancy Histogram Tree**
1.  Initialize leaf nodes with class `unknown` [S1, unofficial].
2.  Perform culling asynchronously or on demand:
    *   If all contained objects are disabled, set class to `empty`.
    *   If min/max scalar values result in zero opacity for the current transfer function, set class to `empty`.
    *   Otherwise, set class to `non-empty`.
    *   If meta-data is missing, compute it upon an "occupancy miss" [S1, unofficial].
3.  Propagate histograms: Traverse the tree recursively in depth-first order. For each non-leaf node, after returning from children, compute its histogram as the sum of its children's histograms [S1, unofficial].
4.  Grow/Prune:
    *   If an occupancy miss occurs for a non-existent node, create it with class `unknown` [S1, unofficial].
    *   Retire invisible nodes (occluded or outside frustum) using a least-recently used strategy [S1, unofficial].

**Step 2: Generate Occupancy Geometry**
1.  Traverse the occupancy histogram tree top-down.
2.  Maintain the current occupancy class from the parent.
3.  If the current node's class differs from the parent's class, emit a bounding box for the current node's spatial extent with the current node's class [S1, unofficial].
4.  Recurse into children.
5.  Re-use this geometry across frames unless the active object set or transfer function changes [S1, unofficial].

**Step 3: Rasterize Ray Segment Lists**
1.  For each new view (or if geometry changed), rasterize the occupancy geometry onto the screen [S1, unofficial].
2.  For each pixel, collect intersections of the ray with the bounding boxes.
3.  Sort intersections by distance from the camera.
4.  Merge consecutive segments of the same occupancy class into a single segment [S1, unofficial].
5.  Store the resulting segments in the per-pixel linked list structure [S1, unofficial].
    *   *Note:* If the view only changes lighting (no camera movement), re-use existing lists [S1, unofficial].

**Step 4: Ray-Casting with Leap**
1.  For each pixel, retrieve the ray segment list head.
2.  Traverse the linked list from front to back.
3.  For each segment:
    *   If class is `empty`, skip sampling and continue to the next segment.
    *   If class is `non-empty` or `unknown`, perform volume sampling along the segment interval [S1, unofficial].
    *   If class is `unknown` and sampling reveals it is actually empty, report an occupancy miss to trigger tree growth/culling [S1, unofficial].
4.  Composite samples using standard front-to-back compositing [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class OccupancyNode:
    class_id: int  # 0: empty, 1: non-empty, 2: unknown
    histogram: [int, int, int] # counts for empty, non-empty, unknown
    children: [OccupancyNode, ...] # 8 children for octree-like structure
    bbox: BoundingBox
    meta_data: dict # min/max scalar, object IDs

class RaySegment:
    start_t: float
    end_t: float
    class_id: int
    next_index: int # index in the global segment buffer

class RaySegmentListHead:
    head_index: int

# Step 1: Histogram Propagation
def propagate_histograms(node: OccupancyNode):
    if node.is_leaf():
        return
    
    # Initialize histogram to zeros
    node.histogram = [0, 0, 0]
    
    for child in node.children:
        propagate_histograms(child)
        for i in range(3):
            node.histogram[i] += child.histogram[i]

# Step 2: Geometry Generation
def generate_geometry(node: OccupancyNode, parent_class: int, geometry_list: list):
    current_class = node.class_id
    
    # Emit geometry if class changes
    if current_class != parent_class:
        geometry_list.append({
            'bbox': node.bbox,
            'class_id': current_class
        })
        
    # Recurse
    if not node.is_leaf():
        for child in node.children:
            generate_geometry(child, current_class, geometry_list)

# Step 3: Rasterization (Simplified Conceptual)
def rasterize_geometry(geometry_list: list, screen_width: int, screen_height: int):
    # Allocate buffers
    # segments_buffer: array of RaySegment
    # heads_buffer: array of RaySegmentListHead
    
    for y in range(screen_height):
        for x in range(screen_width):
            ray = get_ray(x, y)
            intersections = []
            
            for geo in geometry_list:
                t_start, t_end = ray.intersect_bbox(geo['bbox'])
                if t_start is not None:
                    intersections.append({
                        't_start': t_start,
                        't_end': t_end,
                        'class_id': geo['class_id']
                    })
            
            # Sort by t_start
            intersections.sort(key=lambda k: k['t_start'])
            
            # Merge consecutive same-class segments
            merged = []
            if intersections:
                current = intersections[0]
                for i in range(1, len(intersections)):
                    next_seg = intersections[i]
                    if current['class_id'] == next_seg['class_id']:
                        # Merge
                        current['t_end'] = next_seg['t_end']
                    else:
                        merged.append(current)
                        current = next_seg
                merged.append(current)
            
            # Build linked list
            head_idx = allocate_head()
            current_idx = head_idx
            for seg in merged:
                seg_idx = allocate_segment()
                segments_buffer[seg_idx] = RaySegment(
                    start_t=seg['t_start'],
                    end_t=seg['t_end'],
                    class_id=seg['class_id'],
                    next_index=-1
                )
                if current_idx != head_idx:
                    segments_buffer[current_idx].next_index = seg_idx
                current_idx = seg_idx
            
            heads_buffer[x + y * screen_width] = RaySegmentListHead(head_index=head_idx)

# Step 4: Ray-Casting
def ray_cast_pixel(x: int, y: int, heads_buffer: list, segments_buffer: list):
    head = heads_buffer[x + y * screen_width]
    idx = head.head_index
    
    acc_color = [0.0, 0.0, 0.0, 0.0]
    acc_alpha = 0.0
    
    while idx != -1:
        seg = segments_buffer[idx]
        
        if seg.class_id == 0: # Empty
            idx = seg.next_index
            continue
        
        # Sample non-empty or unknown
        samples = sample_volume(seg.start_t, seg.end_t)
        for sample in samples:
            color, alpha = sample.color, sample.alpha
            acc_color = acc_color + color * alpha * (1.0 - acc_alpha)
            acc_alpha = acc_alpha + alpha * (1.0 - acc_alpha)
            
            if acc_alpha > 0.99: # Early termination threshold
                break
        
        if acc_alpha > 0.99:
            break
            
        idx = seg.next_index
        
    return acc_color
```

### Parameters and Edge Cases
*   **Occupancy Classes:** `empty` (0), `non-empty` (1), `unknown` (2) [S1, unofficial].
*   **List Element Size:** 12 bytes [S1, unofficial].
*   **List Head Size:** 16 bytes [S1, unofficial].
*   **Screen Resolution:** Examples use 1,200×1,200 [S1, unofficial].
*   **Edge Case - Unknown Class:** If a ray encounters an `unknown` segment, the ray-caster samples it. If the sample reveals the region is empty, an "occupancy miss" is generated. This triggers asynchronous culling or tree growth [S1, unofficial].
*   **Edge Case - Sparsity:** The technique incurs overhead (meta-data storage, rasterization). It is only efficient if the volume is sufficiently sparse [S1, unofficial].
*   **Edge Case - Frame Coherence:** If the camera does not move, ray segment lists can be reused, skipping rasterization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rasterization is $O(G \cdot P)$ where $G$ is geometry complexity and $P$ is pixel count, but hardware rasterization makes this very fast. Ray-casting is $O(S)$ where $S$ is the number of non-empty segments per ray.
*   **Memory Complexity:**
    *   **CPU:** Similar to an octree, plus 3 integers per node for histograms [S1, unofficial].
    *   **GPU Geometry:** Significantly less than an octree (kilobytes vs megabytes) because many nodes do not emit geometry [S1, unofficial].
    *   **GPU Ray Segment Lists:** Depends on depth complexity. For 1,200×1,200 resolution and average depth complexity of 4, ~154 MB is sufficient for up to depth complexity 8 [S1, unofficial].
*   **Performance:** SparseLeap skips 31.96% of empty space in the Lego dataset [S2, unofficial]. It requires ~32 fewer samples per ray than hierarchical sampling for equal quality (PSNR 22.23 vs 20.60 at N=32) [S2, unofficial].
*   **Strengths:** Avoids space fragmentation; linear list traversal is faster than tree traversal; agnostic to underlying volume data structure; handles dynamic transfer functions efficiently [S1, unofficial].
*   **Limitations:** Overhead of rasterization and list management; requires sufficient sparsity to amortize costs; initial frames may have `unknown` segments requiring misses [S1, unofficial].

## Background
SparseLeap was proposed by Hadwiger et al. in 2018 [S1, unofficial]. It addresses the limitations of standard octree empty space skipping, which suffers from space fragmentation and high traversal costs for fine structures [S1, unofficial]. The method leverages GPU hardware rasterization to move the cost of empty space detection out of the ray-casting shader [S1, unofficial]. It has been tested on datasets such as SEM Mouse Cortex, Aneurysm, Stag Beetle, and Xmas Tree [S1, unofficial].

## Sources
[S1] SparseLeap: Efficient Empty Space Skipping, https://vcg.seas.harvard.edu/publications/20180101-sparseleap-efficient-empty-space-skipping-for-large-scale-volume-rendering/paper, fetched 2026-10-24 (unofficial)
[S2] Pradip-Kathiriya/Improving-Neural-Radiance-Field-Inference-by-Skipping-Empty-Space-in-Volume-Rendering-, https://github.com/Pradip-Kathiriya/Improving-Neural-Radiance-Field-Inference-by-Skipping-Empty-Space-in-Volume-Rendering-, fetched 2026-10-26 (unofficial)
[S3] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
