# Distributed Volume Raycasting with Bricks

Also called: Cluster-based volume raycasting, GPU raycasting on cluster systems

## Overview
Distributed Volume Raycasting with Bricks is a parallel rendering technique that subdivides a volumetric dataset into convex subvolumes (bricks), assigning each to a separate node in a cluster for independent GPU raycasting [S1]. It differs from single-GPU methods by distributing the computational load across multiple machines and using software-based compositing to handle the final image assembly, thereby avoiding hardware blending limitations and `glReadPixels` performance bottlenecks [S1].

## How it works
**Inputs:**
- Volumetric dataset $V$ with total slice counts $S_x, S_y, S_z$ in each dimension.
- User-defined number of bricks $B_x, B_y, B_z$ in each dimension.
- User-defined sampling distance $d$.
- Camera parameters (view matrix, projection matrix).

**Outputs:**
- Final composited image sent to the viewer.

**Data Structures:**
- **Brick Layout:** The volume is partitioned into a grid of $B_x \times B_y \times B_z$ bricks. Each brick $\vec{b} = (b_x, b_y, b_z)^T$ is a convex subvolume.
- **Brick Dimensions:** The number of slices $S_c(\vec{b})$ for brick $\vec{b}$ in dimension $c \in \{x, y, z\}$ is:
  $$
  S_c(\vec{b}) = \begin{cases} \left\lceil \frac{S_c}{B_c} \right\rceil & \text{if } b_c < B_c - 1 \\ S_c - (B_c - 1) \cdot \left\lceil \frac{S_c}{B_c} \right\rceil & \text{otherwise} \end{cases}
  $$
- **Overlap Handling:** To support tri-linear interpolation at brick borders, each brick includes one overlapping slice from its neighbor. The actual texture size $\tilde{s}_c(\vec{b})$ in dimension $c$ is:
  $$
  \tilde{s}_c(\vec{b}) = \begin{cases} S_c(\vec{b}) + 1 & \text{if } b_c < B_c - 1 \\ S_c(\vec{b}) & \text{otherwise} \end{cases}
  $$
  The texture is shifted by half a texel width to account for this overlap [S1].
- **Normalized Volume Size:** The extent of a brick $\vec{b}$ in dimension $c$ is $e_c(\vec{b}) = S_c(\vec{b}) \cdot D_c$, where $D_c$ is the voxel depth. The normalized volume size is:
  $$
  E_c(\vec{b}) = \frac{e_c(\vec{b})}{\max\{e_x^b, e_y^b, e_z^b\}}
  $$
  [S1]

**Algorithm Steps:**
1. **Subdivision:** Subdivide the volume dataset into bricks using a kd-tree to ensure convex partitions [S1].
2. **Assignment:** Assign bricks to render nodes in the cluster.
3. **Request:** The viewer sends a render request (camera parameters, volume data references) to a designated master node [S1].
4. **Broadcast:** The master node broadcasts the request to all render nodes via MPI [S1].
5. **Raycasting:** Each render node raycasts its assigned bricks independently on its GPU.
   - **Vertex Shader:** Computes ray direction vectors and passes them as additional texture coordinates to the fragment shader. This avoids recomputing directions in the fragment shader and ensures correct interpolation across brick boundaries [S1].
   - **Fragment Shader:** Performs raycasting within texture space $[0,1]$ for each brick.
   - **Sampling Distance Adjustment:** The user-defined sampling distance $d$ must be scaled depending on the number of bricks to prevent an increase in the total number of sampling points, as raycasting happens in normalized texture space per brick [S1].
6. **Compositing:** Nodes send their rendered images to a compositing node (or the viewer). Compositing is performed in software using the Direct-Send communication scheme [S1].
   - **Empty-Space Skipping:** An sl-sparse system based on the projection of bounding boxes is used to prevent blending empty pixels and mitigate `glReadPixels` performance issues [S1].
7. **Transfer:** The final image is sent to the viewer.
   - **Compression:** The image may be compressed using LZO real-time compression before transmission over the "last mile" (cluster to viewer). Computation time is extremely low, and compressed size is typically ~10% of the original [S1].
   - **Transfer Methods:**
     1. Collect all images on one node and send one large package (uses fast interconnect, efficient for slow last-mile).
     2. Send partial images directly from each node to the viewer (less idle time, but $n$ smaller packages) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the distributed workflow
# Note: MPI and GPU-specific calls are abstracted as per source description

def distributed_volume_raycasting(volume_data, camera_params, B_x, B_y, B_z, d):
    # 1. Subdivision and Assignment
    bricks = subdivide_volume(volume_data, B_x, B_y, B_z)
    render_nodes = assign_bricks_to_nodes(bricks)
    
    # 2. Request and Broadcast
    master_node = render_nodes[0]
    master_node.receive_request(camera_params)
    master_node.broadcast_request(render_nodes)
    
    # 3. Parallel Raycasting
    results = []
    for node in render_nodes:
        # Each node processes its assigned bricks
        node_images = []
        for brick in node.bricks:
            # Adjust sampling distance based on brick count to maintain constant total samples
            # [G] Scaling factor logic is implied but not explicitly formulaic in sources
            adjusted_d = scale_sampling_distance(d, len(bricks))
            
            # GPU Raycasting
            # Vertex shader computes ray direction and passes as texture coords
            # Fragment shader raycasts in [0,1] texture space
            image = gpu_raycast(brick, camera_params, adjusted_d)
            node_images.append(image)
        
        # 4. Compositing (Software-based Direct-Send)
        # [G] Assuming node_images are composited into a single image for this node's contribution
        composited_image = software_composite(node_images)
        results.append(composited_image)
    
    # 5. Final Compositing and Transfer
    # Method 1: Collect on one node
    final_image = software_composite(results)
    
    # Compress if needed
    if use_lzo:
        final_image = lzo_compress(final_image)
        
    # Send to viewer
    send_to_viewer(final_image)
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and sampling points, distributed across $N$ nodes. The sampling distance must be scaled with the number of bricks to keep the total sampling cost constant [S1].
- **Memory Complexity:** Each node stores only its assigned bricks. Overlap slices increase local memory usage slightly [S1].
- **Performance:** Compositing in software avoids the performance penalty of `glReadPixels` and hardware blending of empty pixels [S1]. LZO compression adds negligible computation time while reducing bandwidth usage by ~90% [S1].
- **Strengths:** Scalability across cluster nodes; efficient use of fast interconnects for internal compositing; reduced bandwidth requirements for the final image transfer [S1].
- **Limitations:** Requires MPI infrastructure and software compositing logic; communication overhead between nodes and viewer; potential load imbalance if bricks vary significantly in complexity (though convex partitioning helps) [S1].
- **When to use:** Large volume datasets that exceed single-GPU memory or compute capacity; environments with cluster infrastructure and MPI support [S1].
- **When not to use:** Small datasets; single-GPU systems; scenarios requiring real-time interactive feedback where network latency is prohibitive [S1].

## Background
The method was introduced in the paper "Optimized volume raycasting for graphics-hardware-based cluster systems" (2006) [S1]. It addresses the limitations of single-GPU raycasting by leveraging cluster computing. The approach builds on earlier CPU-based empty-space-skipping techniques [Lev90] but adapts them for GPU hardware and distributed software compositing [S1].

## Sources
[S1] Optimized volume raycasting for graphics-hardware-based cluster systems, https://doi.org/10.2312/EGPGV/EGPGV06/059-066, fetched 2026-10-27
