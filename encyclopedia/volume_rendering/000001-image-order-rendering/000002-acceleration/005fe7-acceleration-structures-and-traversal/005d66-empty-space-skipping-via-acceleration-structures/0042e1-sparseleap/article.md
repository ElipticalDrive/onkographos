# SparseLeap

Also called: Empty Space Skipping via Rasterized Ray Segment Lists, SparseLeap-an effective empty space skipping technique

## Overview
SparseLeap is a hybrid object-order (rasterization) and image-order (ray-casting) method for efficient empty space skipping in large, sparse volumes with fine structures [S1, unofficial]. It decouples empty space skipping data structures from the underlying volume representation, avoiding the space fragmentation and hierarchy traversal overhead typical of standard octree-based approaches [S1, unofficial]. By rasterizing view-independent occupancy geometry into per-pixel linked lists of ray segments, it moves the major cost of empty space skipping from the ray-casting shader to GPU hardware rasterization [S1, unofficial].

## How it works
SparseLeap operates in four main stages: building an occupancy histogram tree, generating occupancy geometry, rasterizing ray segment lists, and performing ray-casting with linear list traversal [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Volume data (grid, octree, etc.) and segmentation labels [S1, unofficial].
- Current transfer function and set of active (enabled) objects [S1, unofficial].
- Camera view parameters (position, orientation) [S1, unofficial].

**Outputs:**
- Rendered image with empty space skipped [S1, unofficial].
- Updated occupancy histogram tree (if grown or pruned) [S1, unofficial].

### Data Structures

**Occupancy Histogram Tree Node (CPU):**
- Child pointers (standard tree structure) [S1, unofficial].
- Histogram: Three integer counts representing the number of leaf nodes in the sub-tree for each occupancy class: `count_empty`, `count_non_empty`, `count_unknown` [S1, unofficial].
- Leaf nodes may store metadata (min/max scalar value, contained objects) for culling [S1, unofficial].

**Occupancy Geometry (GPU):**
- A set of nested bounding boxes [S1, unofficial].
- Each box has an associated occupancy class (`empty`, `non-empty`, `unknown`) [S1, unofficial].
- Boxes are emitted only when the occupancy class changes during a top-down tree traversal [S1, unofficial].

**Ray Segment List Buffer (GPU):**
- Two arrays per pixel resolution [S1, unofficial]:
  1. **Linked List Elements:** Array of size `screen_width * screen_height * max_depth_complexity`. Each element is 12 bytes [S1, unofficial].
     - Layout: `start` (float, ray parameter), `end` (float, ray parameter), `class` (uint8, occupancy class) [G: inferred from 12-byte size and segment definition].
  2. **Linked List Heads:** Array of size `screen_width * screen_height`. Each head is 16 bytes [S1, unofficial].
     - Layout: `head_index` (int32, index into elements array), `count` (uint32, number of segments) [G: inferred from 16-byte size and linked list header definition].

### Numbered Steps

1. **Build/Update Occupancy Histogram Tree:**
   - Traverse the tree recursively in depth-first order starting at the root [S1, unofficial].
   - For leaf nodes, determine occupancy class by comparing metadata against the current transfer function and active object set [S1, unofficial]:
     - If all contained objects are disabled or min/max scalar value results in zero opacity: `empty` [S1, unofficial].
     - Otherwise: `non-empty` [S1, unofficial].
     - If metadata is missing or culling is delayed: `unknown` [S1, unofficial].
   - For non-leaf nodes, compute histogram as the sum of children's histograms [S1, unofficial].

2. **Generate Occupancy Geometry:**
   - Traverse the occupancy histogram tree top-down [S1, unofficial].
   - Emit a bounding box for a node only if its occupancy class differs from its parent's class [S1, unofficial].
   - Box sizes decrease from root to leaves [S1, unofficial].
   - Re-use previously generated geometry if occupancy classes have not changed (e.g., only light direction changed) [S1, unofficial].

3. **Rasterize Ray Segment Lists:**
   - Rasterize the occupancy geometry onto the screen [S1, unofficial].
   - For each pixel, generate a linked list of ray segments corresponding to the intersections with the bounding boxes [S1, unofficial].
   - Merge consecutive segments of the same occupancy class into a single segment [S1, unofficial].
   - The rasterization order ensures segments are sorted from front to back along the ray [S1, unofficial].
   - Re-use existing ray segment lists if the view has not changed (e.g., only lighting changed) [S1, unofficial].

4. **Ray-Casting with Linear List Traversal:**
   - For each pixel, retrieve its ray segment list [S1, unofficial].
   - Traverse the list from front to back [S1, unofficial].
   - Skip segments with class `empty` [S1, unofficial].
   - For segments with class `non-empty` or `unknown`:
     - Sample the volume data within the segment [S1, unofficial].
     - If class is `unknown`, generate an occupancy miss [S1, unofficial].
   - Perform standard front-to-back compositing for sampled segments [G: standard volume rendering loop].

### Formulas

**Occupancy Histogram Propagation:**
For a non-leaf node $N$ with children $C_1, C_2, \dots, C_k$:
$$ H_N = \sum_{i=1}^{k} H_{C_i} $$
where $H_N$ is the histogram vector $(count\_empty, count\_non\_empty, count\_unknown)$ for node $N$ [S1, unofficial].

**Ray-Casting Loop:**
$$ C_{acc} = 0, \alpha_{acc} = 0 $$
$$ \text{for each segment } S \text{ in ray segment list:} $$
$$ \quad \text{if } S.class == \text{empty: continue} $$
$$ \quad \text{if } S.class == \text{unknown:} $$
$$ \quad \quad \text{trigger occupancy miss for } S $$
$$ \quad \quad \text{continue} $$
$$ \quad \text{sample volume along } S $$
$$ \quad C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
$$ \quad \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$

### Parameters
- **Occupancy Classes:** `empty`, `non-empty`, `unknown` [S1, unofficial].
- **List Element Size:** 12 bytes [S1, unofficial].
- **List Head Size:** 16 bytes [S1, unofficial].
- **Screen Resolution:** 1,200×1,200 (used in implementation examples) [S1, unofficial].

### Edge Cases and Pitfalls
- **Unknown Class Handling:** If a ray encounters a segment of class `unknown`, the ray-caster generates an occupancy miss [S1, unofficial]. This triggers asynchronous culling or tree growth [S1, unofficial].
- **Sparsity Requirement:** The technique is only efficient if the volume is sufficiently sparse, as storing and computing metadata incurs overhead that must be amortized [S1, unofficial].
- **Tree Growth/Pruning:** If an occupancy miss corresponds to a non-existent node, the tree is grown to include it [S1, unofficial]. Unused nodes are pruned using a least-recently used strategy [S1, unofficial].
- **NeRF Dependency:** When used with Neural Radiance Fields, the effectiveness of empty space skipping depends on the quality of the NeRF output, as accurate volume density information improves skipping [S2, unofficial][S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Occupancy Histogram Tree Node
class OccupancyNode:
    def __init__(self):
        self.children = []
        self.histogram = [0, 0, 0]  # [empty, non_empty, unknown]
        self.metadata = None  # min/max scalar, objects
        self.occupancy_class = None  # leaf class

# Step 1: Build/Update Occupancy Histogram Tree
def build_occupancy_tree(root, transfer_function, active_objects):
    def dfs(node):
        if node is leaf:
            node.occupancy_class = determine_leaf_class(node, transfer_function, active_objects)
        else:
            for child in node.children:
                dfs(child)
            node.histogram = [sum(c[i] for c in node.children) for i in range(3)]
    dfs(root)

def determine_leaf_class(leaf, transfer_function, active_objects):
    if leaf.metadata is None:
        return 'unknown'
    if all_objects_disabled(leaf.metadata, active_objects):
        return 'empty'
    if zero_opacity(leaf.metadata, transfer_function):
        return 'empty'
    return 'non_empty'

# Step 2: Generate Occupancy Geometry
def generate_occupancy_geometry(root):
    geometry = []
    def traverse(node, parent_class):
        if node.occupancy_class != parent_class:
            geometry.append(BoundingBox(node.spatial_extent, node.occupancy_class))
        for child in node.children:
            traverse(child, node.occupancy_class)
    traverse(root, 'empty')
    return geometry

# Step 3: Rasterize Ray Segment Lists
def rasterize_ray_segment_lists(geometry, camera, screen_width, screen_height):
    elements = []  # List of (start, end, class)
    heads = [None] * (screen_width * screen_height)
    
    for box in geometry:
        for y in range(screen_height):
            for x in range(screen_width):
                ray = get_ray(x, y, camera)
                if ray_intersects_box(ray, box):
                    segment = (box.ray_start, box.ray_end, box.class)
                    idx = y * screen_width + x
                    if heads[idx] is None:
                        heads[idx] = []
                    heads[idx].append(segment)
    
    for idx in range(screen_width * screen_height):
        if heads[idx] is not None:
            heads[idx] = merge_segments(heads[idx])
    
    return elements, heads

def merge_segments(segments):
    if not segments:
        return []
    merged = [segments[0]]
    for seg in segments[1:]:
        if seg[2] == merged[-1][2]:
            merged[-1] = (merged[-1][0], seg[1], seg[2])
        else:
            merged.append(seg)
    return merged

# Step 4: Ray-Casting
def ray_cast(screen_width, screen_height, heads):
    image = zeros((screen_height, screen_width, 3))
    for y in range(screen_height):
        for x in range(screen_width):
            ray = get_ray(x, y)
            segments = heads[y * screen_width + x]
            if segments is None:
                continue
            
            C_acc = 0
            alpha_acc = 0
            for seg in segments:
                if seg[2] == 'empty':
                    continue
                if seg[2] == 'unknown':
                    continue  # Trigger occupancy miss handled asynchronously
                
                C_sample, alpha_sample = sample_volume(ray, seg[0], seg[1])
                
                alpha_term = alpha_sample * (1 - alpha_acc)
                C_acc += C_sample * alpha_term
                alpha_acc += alpha_term
            
            image[y, x] = C_acc
    return image
```

## Subtopics

### [Hierarchical Bounding Box Tree for Empty-Space Skipping](8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md)
Uses a hierarchical tree of bounding boxes for empty space skipping. Choose when a strict hierarchy is preferred over rasterization.

### [SparseLeap Ray-Casting with Occupancy Misses](9818-sparseleap-ray-casting-with-occupancy-misses.md)
Extends SparseLeap with dynamic tree growth and pruning based on occupancy misses. Choose when volume sparsity changes dynamically.

### [Space Skipping and Empty Volume Culling](10063-space-skipping-and-empty-volume-culling.md)
Traditional space skipping using bounding volumes. Choose for simpler implementations without rasterization.

### [Empty-space skipping via Acceleration Structure](13062-empty-space-skipping-via-acceleration-structure.md)
General acceleration structure-based skipping. Choose for generic volume representations.

### [Progressive Ray Casting for Volumetric Models on Mobile Devices](16346-progressive-ray-casting-for-volumetric-models-on-mobile-devi.md)
Focuses on mobile performance with progressive rendering. Choose for mobile devices.

## Operating characteristics
- **Time Complexity:** Rasterization is $O(G)$ where $G$ is the number of occupancy geometry primitives [S1, unofficial]. Ray-casting is $O(S)$ where $S$ is the number of non-empty segments per ray [S1, unofficial].
- **Memory Complexity:** GPU memory for ray segment lists is $O(W \cdot H \cdot D)$ where $W, H$ are screen dimensions and $D$ is average depth complexity [S1, unofficial]. Occupancy geometry is significantly smaller than equivalent octrees (kilobytes vs megabytes) [S1, unofficial].
- **Measured Performance:** On a Geforce Titan X with 1,200×1,200 resolution, SparseLeap achieved higher frame rates than octree-based methods for sparse volumes [S1, unofficial]. For the Lego dataset (100x100 resolution), SparseLeap skipped 31.96% of empty space and achieved a PSNR of 22.23 with N=32 samples per ray, compared to 20.60 for hierarchical sampling [S2, unofficial][S3, unofficial].
- **Strengths:** Avoids space fragmentation, decouples empty space skipping from volume sampling, leverages GPU rasterization, supports dynamic transfer functions and object sets [S1, unofficial].
- **Limitations:** The technique is only efficient if the volume is sufficiently sparse, as storing and computing metadata incurs overhead that must be amortized [S1, unofficial]. When used with Neural Radiance Fields, the effectiveness of empty space skipping depends on the quality of the NeRF output, as accurate volume density information improves skipping [S2, unofficial][S3, unofficial].
- **When to Use:** Large, sparse volumes with fine structures, dynamic transfer functions, or active object sets [S1, unofficial].
- **When Not to Use:** Dense volumes where sparsity is low, or when hardware rasterization is not available [S1, unofficial].

## Background
SparseLeap was introduced by Hadwiger et al. in 2018 [S1, unofficial]. It was designed to address the limitations of standard octree empty space skipping, which suffers from space fragmentation and high traversal costs for large, finely detailed volume data [S1, unofficial]. The method has been applied to various datasets including SEM Mouse Cortex, Aneurysm, Stag Beetle, and Xmas Tree [S1, unofficial]. It is also used in conjunction with Neural Radiance Fields (NeRF) to improve rendering efficiency [S2, unofficial][S3, unofficial].

## Sources
[S1] SparseLeap: Efficient Empty Space Skipping, https://vcg.seas.harvard.edu/publications/20180101-sparseleap-efficient-empty-space-skipping-for-large-scale-volume-rendering/paper, fetched 2026-10-24 (unofficial)
[S2] Kelebrimbor97/Improving-Neural-Radiance-Field-Inference-by-Skipping-Empty-Space-in-Volume-Rendering, https://github.com/Kelebrimbor97/Improving-Neural-Radiance-Field-Inference-by-Skipping-Empty-Space-in-Volume-Rendering, fetched 2026-10-26 (unofficial)
[S3] Pradip-Kathiriya/Improving-Neural-Radiance-Field-Inference-by-Skipping-Empty-Space-in-Volume-Rendering-, https://github.com/Pradip-Kathiriya/Improving-Neural-Radiance-Field-Inference-by-Skipping-Empty-Space-in-Volume-Rendering-, fetched 2026-10-26 (unofficial)
[S4] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
