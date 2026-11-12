# Distributed Volume Raycasting with Bricks

Also called: Cluster-based volume raycasting, GPU raycasting on cluster systems

## Overview
Distributed Volume Raycasting with Bricks is a parallel rendering technique that subdivides a volumetric dataset into convex subvolumes (bricks), assigning each to a separate node in a cluster for independent GPU raycasting [S1]. It differs from single-GPU methods by distributing the computational load across multiple machines and using software-based compositing to handle the final image assembly, thereby avoiding hardware blending limitations and `glReadPixels` performance bottlenecks [S1].

## How it works
**Inputs:**
- Volumetric dataset $V$ with total slice counts $S_x, S_y, S_z$ in each dimension [S1].
- User-defined number of bricks $B_x, B_y, B_z$ in each dimension [S1].
- User-defined sampling distance $d$ [S1].
- Camera parameters (view matrix, projection matrix) [G].

**Outputs:**
- Final composited image sent to the viewer [S1].

**Data Structures:**
- **Brick Layout:** The volume is partitioned into a grid of $B_x \times B_y \times B_z$ bricks [S1]. Each brick $\vec{b} = (b_x, b_y, b_z)^T$ is a convex subvolume [S1].
- **Brick Dimensions:** The number of slices $S_c(\vec{b})$ for brick $\vec{b}$ in dimension $c \in \{x, y, z\}$ is:
  $$
  S_c(\vec{b}) = \begin{cases} \left\lceil \frac{S_c}{B_c} \right\rceil & \text{if } b_c < B_c - 1 \\ S_c - (B_c - 1) \cdot \left\lceil \frac{S_c}{B_c} \right\rceil & \text{otherwise} \end{cases}
  $$
  [S1]
- **Overlap Handling:** To support tri-linear interpolation at brick borders, each brick includes one overlapping slice from its neighbor [S1]. The actual texture size $\tilde{s}_c(\vec{b})$ in dimension $c$ is:
  $$
  \tilde{s}_c(\vec{b}) = \begin{cases} S_c(\vec{b}) + 1 & \text{if } b_c < B_c - 1 \\ S_c(\vec{b}) & \text{otherwise} \end{cases}
  $$
  The texture is shifted by half a texel width to account for this overlap [S1].
- **Normalized Volume Size:** The extent of a brick $\vec{b}$ in dimension $c$ is $e_c(\vec{b}) = S_c(\vec{b}) \cdot D_c$, where $D_c$ is the extent of a full-size brick [S1]. The normalized volume size is:
  $$
  E_c(\vec{b}) = \frac{e_c(\vec{b})}{\max\{e_x^b, e_y^b, e_z^b\}}
  $$
  [S1]

**Algorithm Steps:**
1. **Subdivision:** Subdivide the volume dataset into bricks using a kd-tree to ensure convex partitions [S1].
2. **Assignment:** Assign bricks to render nodes in the cluster [S1].
3. **Request:** The viewer sends a render request containing necessary setup information to a designated master node [S1].
4. **Broadcast:** The master node broadcasts the request to all render nodes via MPI [S1].
5. **Raycasting:** Each render node raycasts its assigned bricks independently on its GPU [S1].
   - **Vertex Shader:** Computes ray direction vectors and passes them as additional texture coordinates to the fragment shader. This avoids recomputing directions in the fragment shader and ensures correct interpolation across brick boundaries [S1].
   - **Fragment Shader:** Performs raycasting within texture space $[0,1]$ for each brick [S1].
   - **Sampling Distance Adjustment:** The user-defined sampling distance $d$ must be scaled depending on the number of bricks to prevent an increase in the total number of sampling points, as raycasting happens in normalized texture space per brick [S1].
6. **Compositing:** Nodes send their rendered images to a compositing node (or the viewer). Compositing is performed in software using the Direct-Send communication scheme [S1].
   - **Empty-Space Skipping:** An sl-sparse system based on the projection of bounding boxes is used to prevent blending empty pixels and mitigate `glReadPixels` performance issues [S1].
7. **Transfer:** The final image is sent to the viewer [S1].
   - **Compression:** The image may be compressed using LZO real-time compression before transmission over the "last mile" (cluster to viewer). Computation time is extremely low, and compressed size is typically ~10% of the original [S1].
   - **Transfer Methods:**
     1. Collect all images on one node and send one large package (uses fast interconnect, efficient for slow last-mile) [S1].
     2. Send partial images directly from each node to the viewer (less idle time, but $n$ smaller packages) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def distributed_volume_raycasting(volume_data, camera_params, B_x, B_y, B_z, d):
    # 1. Subdivision: Subdivide volume into bricks using a kd-tree for convex partitions [S1]
    # [G] Source does not specify kd-tree implementation; standard spatial partitioning assumed.
    bricks = [(bx, by, bz) for bx in range(B_x) for by in range(B_y) for bz in range(B_z)]

    # 2. Assignment: Assign bricks to render nodes [S1]
    num_nodes = len(bricks)
    render_nodes = [[] for _ in range(num_nodes)]
    for i, brick in enumerate(bricks):
        render_nodes[i % num_nodes].append(brick)

    # 3. Request & 4. Broadcast: Viewer sends request to master, master broadcasts via MPI [S1]
    master_node = render_nodes[0]
    request = {"camera": camera_params, "bricks": bricks, "d": d}
    # [G] MPI broadcast abstracted as per source description [S1]

    # 5. Parallel Raycasting: Each node raycasts assigned bricks independently [S1]
    node_results = []
    for node_idx, node_bricks in enumerate(render_nodes):
        node_images = []
        for brick in node_bricks:
            bx, by, bz = brick
            B = (B_x, B_y, B_z)
            S = (volume_data.S_x, volume_data.S_y, volume_data.S_z)

            # Compute brick slice counts [S1]
            def calc_slices(c, b_c, B_c, S_c):
                return math.ceil(S_c / B_c) if b_c < B_c - 1 else S_c - (B_c - 1) * math.ceil(S_c / B_c)
            S_b = tuple(calc_slices(dim, b_c, B_c, S_c) for dim, (b_c, B_c, S_c) in enumerate(zip(brick, B, S)))

            # Overlap handling: replicate adjacent slice, shift texture by half texel [S1]
            tilde_s = tuple(s + 1 if b < B - 1 else s for s, b, B in zip(S_b, brick, B))
            texture_shift = 0.5 # half texel width [S1]

            # Sampling distance scaling: scale d by number of bricks to prevent increased sampling points [S1]
            total_bricks = B_x * B_y * B_z
            adjusted_d = d / math.sqrt(total_bricks) # [G] Standard scaling heuristic implied by source

            # Vertex shader computes ray direction, passes as texture coords [S1]
            # [G] Source states vertex shader computes ray direction and passes it as texture coordinates.
            ray_dir = compute_ray_direction(camera_params, brick)

            # Fragment shader raycasts in texture space [0,1] [S1]
            # [G] Source states raycasting happens in texture space [0,1]. Standard raymarching loop.
            image = raycast_brick(volume_data, brick, S_b, tilde_s, texture_shift, adjusted_d, ray_dir)
            node_images.append(image)

        # 6. Compositing: Software compositing using Direct-Send [S1]
        # Empty-space skipping via sl-sparse bounding box projection [S1]
        composited = software_composite_direct_send(node_images)
        node_results.append(composited)

    # 7. Transfer: Send final image to viewer [S1]
    # Method 1: Collect on one node, send large package [S1]
    final_image = software_composite_direct_send(node_results)

    # 8. Compression: LZO compression on "last mile" (cluster to viewer) [S1]
    # Computation time extremely low, size ~10% of original [S1]
    compressed_image = lzo_compress(final_image)

    return compressed_image

def compute_ray_direction(camera_params, brick):
    # [G] Standard vertex shader operation: transforms view ray through camera matrix
    return (1.0, 0.0, 0.0)

def raycast_brick(vol, brick, S_b, tilde_s, shift, d, ray_dir):
    # [G] Standard fragment shader operation: marches ray in [0,1] texture space
    # Accumulates color/opacity using tri-linear interpolation with overlap shift
    return [0.0] * 100

def software_composite_direct_send(images):
    # [G] Standard software compositing: blends images using Direct-Send scheme
    return [0.0] * 100

def lzo_compress(data):
    # [G] Standard LZO compression algorithm call
    return data
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and sampling points, distributed across $N$ nodes. The sampling distance must be scaled with the number of bricks to keep the total sampling cost constant [S1].
- **Memory Complexity:** Each node stores only its assigned bricks. Overlap slices increase local memory usage slightly [S1].
- **Performance:** Compositing in software avoids the performance penalty of `glReadPixels` and hardware blending of empty pixels [S1]. LZO compression adds negligible computation time while reducing bandwidth usage by ~90% [S1].
- **Strengths:** Scalability across cluster nodes; efficient use of fast interconnects for internal compositing; reduced bandwidth requirements for the final image transfer [S1].
- **Limitations:** Requires MPI infrastructure and software compositing logic; communication overhead between nodes and viewer; potential load imbalance if bricks vary significantly in complexity (though convex partitioning helps) [S1].
- **When to use:** Large volume datasets that exceed single-GPU memory or compute capacity; environments with cluster infrastructure and MPI support [S1].

## Background
The method was introduced in the paper "Optimized volume raycasting for graphics-hardware-based cluster systems" (2006) [S1]. It addresses the limitations of single-GPU raycasting by leveraging cluster computing. The approach builds on earlier CPU-based empty-space-skipping techniques [Lev90] but adapts them for GPU hardware and distributed software compositing [S1].

## Sources
[S1] Optimized volume raycasting for graphics-hardware-based cluster systems, https://doi.org/10.2312/EGPGV/EGPGV06/059-066, fetched 2026-10-27
