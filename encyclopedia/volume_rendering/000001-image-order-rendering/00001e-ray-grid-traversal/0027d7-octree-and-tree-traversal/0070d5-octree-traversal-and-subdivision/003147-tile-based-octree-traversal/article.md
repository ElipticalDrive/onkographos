# Tile-based Octree Traversal
Also called: Tile-based rendering with octree
## Overview
Tile-based Octree Traversal accelerates volume rendering by grouping rays into image tiles and pre-computing per-tile node lists of intersecting octree nodes [S1, unofficial]. It addresses scalability for web-based rendering of large biomedical datasets by combining tile-based rendering with hierarchical data structures [S1, unofficial]. It builds on [Octree Traversal and Subdivision](22160-octree-traversal-and-subdivision.md) by batching hierarchical descent across rays sharing a tile frustum [G].
## How it works
**Inputs:** Octree structure, volume data, camera parameters, image resolution, tile dimensions [G].
**Outputs:** Rendered image buffer [G].

**Data Structures:**
1.  **Octree Node:** Contains a bounding box (center $\mathbf{c}$ and half-extent $s$), an empty/uniform flag, pointers to children, and leaf data [G].
2.  **Tile Node List:** A list of pointers to octree nodes intersecting the tile's frustum [S1, unofficial].

**Steps:**
1.  **Tile Node List Generation:** Traverse the octree to collect nodes intersecting each tile's frustum. Sources do not specify the exact generation algorithm [G].
2.  **Ray Generation:** For each pixel in a tile, generate a ray from the camera origin through the pixel center [G].
3.  **Tile-Aware Traversal:** For each ray, traverse the octree using the tile node list as a constraint or starting point [G].
    *   **Child Selection:** For a node with center $\mathbf{c}$ and half-extent $s$, the ray position $\mathbf{p}$ determines the child index $k \in \{0, \dots, 7\}$:
        $$
        k = 4 \cdot \mathbb{I}(p_x \ge c_x) + 2 \cdot \mathbb{I}(p_y \ge c_y) + 1 \cdot \mathbb{I}(p_z \ge c_z)
        $$
        where $\mathbb{I}(\cdot)$ is the indicator function [S1, unofficial][G].
    *   **Empty-Space Skipping:** If the selected child is empty, advance the ray parameter $t$ to the exit of the current node and backtrack to the parent [G].
    *   **Leaf Handling:** Upon reaching a leaf, switch to fixed-step DDA traversal within the leaf's voxel grid to sample volume data [G].
    *   **Backtracking:** Use an explicit stack to return to the parent node when a ray exits a subtree or encounters an empty branch [G].
4.  **Compositing:** Accumulate color and opacity along the ray using front-to-back compositing [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_tile(tile, octree_root, volume_data, max_t):
    # Tile node list generation is not specified in the sources [G].
    tile_node_list = []

    for pixel in tile.pixels:
        ray_o = tile.camera.origin
        ray_d = normalize(tile.camera.ray_for_pixel(pixel))
        color = vec3(0.0)
        alpha = 0.0
        t = 0.0
        stack = [(octree_root, t)]

        while stack:
            node, t_entry = stack.pop()
            if t_entry >= max_t:
                continue

            p = ray_o + ray_d * t_entry
            c = node.center
            s = node.half_extent

            # Child selection
            k = 4 * int(p.x >= c.x) + 2 * int(p.y >= c.y) + 1 * int(p.z >= c.z)

            if node.is_empty:
                # Empty-space skipping: advance t to node exit
                inv_d = 1.0 / ray_d
                t_exit = max(min((c.x - s.x - ray_o.x) * inv_d.x, (c.x + s.x - ray_o.x) * inv_d.x),
                             min((c.y - s.y - ray_o.y) * inv_d.y, (c.y + s.y - ray_o.y) * inv_d.y),
                             min((c.z - s.z - ray_o.z) * inv_d.z, (c.z + s.z - ray_o.z) * inv_d.z))
                if node.parent is not None:
                    stack.append((node.parent, t_exit))
                continue

            if node.is_leaf:
                # Leaf handling: fixed-step DDA
                step = s / node.leaf_resolution
                t_step = max(step / abs(ray_d.x), step / abs(ray_d.y), step / abs(ray_d.z))
                while t < max_t:
                    val = sample_volume(ray_o + ray_d * t, node)
                    color, alpha = composite(color, alpha, val, t_step)
                    t += t_step
                continue

            # Descent: push children
            for i in range(7, -1, -1):
                child = node.children[i]
                if child is not None:
                    inv_d = 1.0 / ray_d
                    b_min = vec3(c.x - s.x, c.y - s.y, c.z - s.z)
                    b_max = vec3(c.x + s.x, c.y + s.y, c.z + s.z)
                    if i & 4: b_min.x = c.x + s.x
                    if i & 2: b_min.y = c.y + s.y
                    if i & 1: b_min.z = c.z + s.z
                    if not (i & 4): b_max.x = c.x - s.x
                    if not (i & 2): b_max.y = c.y - s.y
                    if not (i & 1): b_max.z = c.z - s.z
                    t_child = max((b_min - ray_o) * inv_d)
                    stack.append((child, max(t_entry, t_child)))
        tile.image_buffer[pixel.coords] = color

def sample_volume(pos, node): return 1.0 # Standard volume sampling [G]
def composite(c, a, val, dt): return c + val * (1.0 - a) * dt, a + (1.0 - a) * dt # Front-to-back compositing [G]
```

**Parameters:**
*   **Leaf Node Size:** A smaller leaf-node size leads to less overlap of node lists between image tiles, but increases storage and traversal overhead [S1, unofficial].
*   **Tile Size:** Common sizes are $8 \times 8$ or $16 \times 16$ pixels [G]. Larger tiles increase coherence but may lead to larger node lists and cache misses [G].

**Edge Cases:**
*   **Precision:** Floating-point precision issues in ray-node intersection tests can lead to missed nodes or double-counting. Use epsilon values for comparisons [G].
*   **Degenerate Tiles:** Tiles with no intersecting nodes should be skipped entirely [G].
## Subtopics

### [TileTree Rendering](17468-tiletree-rendering.md)
TileTree Rendering is a method for storing and accessing high-resolution texture data on octree-structured volumes, enabling seamless interpolation and adaptive resolution. It differs from standard octree texturing by storing separate tile maps for each face of the octree leaves and using a weighted blending scheme based on surface normal direction to eliminate visible seams at tile boundaries. Choose when scene complexity varies significantly across the image [G].

## Operating characteristics
**Time Complexity:** $O(N \cdot L)$, where $N$ is the number of rays and $L$ is the average number of nodes visited per ray [G]. Tile-based traversal reduces $L$ by sharing traversal work, but adds overhead for node list generation [G].
**Memory Complexity:** $O(N_{nodes} + N_{tiles} \cdot L_{avg})$, where $N_{nodes}$ is the number of octree nodes and $L_{avg}$ is the average tile node list size [G].
**Performance:** Generally faster than per-ray traversal for large, sparse volumes due to reduced redundant node visits [G]. Performance degrades if tile node lists are large or if the scene is highly uniform [G].
**Strengths:** Efficient for sparse volumes, reduces overdraw, leverages spatial coherence [G].
**Limitations:** Increased memory usage for node lists, complexity in dynamic scenes where the tree changes frequently [G].
**When to Use:** Large, static, sparse volumes where spatial coherence is high [G].
**When Not to Use:** Highly dynamic scenes, very dense volumes, or when memory is constrained [G].
## Background
Tile-based rendering with octrees was popularized by Wang et al. [WBD+ 21] for scalable web-based volume rendering of large biomedical datasets [S1, unofficial]. It addresses the scalability issues of traditional raycasting by combining tile-based rendering techniques with hierarchical data structures [S1, unofficial].
## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
