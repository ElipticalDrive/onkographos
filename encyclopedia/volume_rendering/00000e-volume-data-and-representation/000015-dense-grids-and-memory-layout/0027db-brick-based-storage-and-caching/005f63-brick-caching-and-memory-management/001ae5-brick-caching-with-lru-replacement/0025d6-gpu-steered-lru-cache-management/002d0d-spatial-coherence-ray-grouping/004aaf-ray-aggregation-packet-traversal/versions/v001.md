# Ray Aggregation (Packet Traversal)
Also called: Packet Traversal, Ray Packet Traversal

## Overview
Ray aggregation is a traversal strategy that exploits spatial coherence by grouping rays into packets to be processed as a single unit, thereby reducing traversal overhead and enabling efficient SIMD execution. It is the most widely used coherence strategy in interactive ray tracing systems, distinguishing itself from beam tracing by maintaining explicit ray representations throughout the traversal process [S1].

## How it works
**Inputs and Outputs**
- Input: A group of rays (a packet) sharing similar spatial origins and directions, and a scene representation (typically a k-dtree or grid) [S1].
- Output: Intersection points or traversal status for each ray in the packet.

**Data Structures**
The packet is a container holding the state of multiple rays. While specific field layouts depend on the implementation, the packet must store:
- Origin coordinates $(O_x, O_y, O_z)$ for each ray.
- Direction vectors $(D_x, D_y, D_z)$ for each ray.
- Current traversal state (e.g., current node index in a k-dtree) for each ray.
- Validity flags indicating if a ray has already intersected an object or exited the scene [S1].

**Steps**
1. **Grouping**: Rays are grouped based on spatial coherence, typically by image plane regions (e.g., 2x2 pixel blocks) [S1].
2. **Bounding Volume Calculation**: Compute a conservative bounding volume (such as an AABB or frustum) that encloses all rays in the packet. This allows the traversal algorithm to skip nodes that do not intersect the entire packet's volume [S1].
3. **Traversal**: Traverse the scene hierarchy (e.g., k-dtree) using the packet's bounding volume. If a node is outside the packet's bounds, all rays in the packet skip that node. If inside, the node is visited, and individual ray tests are performed [S1].
4. **SIMD Execution**: Process the rays within the packet using register-level SIMD instructions. For example, four rays can be processed in parallel using 128-bit registers [S1].
5. **Intersection Testing**: Perform intersection tests against primitives. If a ray intersects, record the hit and mark the ray as complete or continue traversal if multiple hits are required [S1].
6. **Divergence Handling**: If rays in the packet diverge significantly (e.g., some hit objects while others do not), the packet may be split or rays marked as inactive to maintain efficiency [S1].

**Formulas**
- Packet Bounding Box: For a packet with rays $i=0..N-1$, the bounding box $B$ is defined by:
  $$ B_{min} = \min_i(O_i + t_{min} D_i) $$
  $$ B_{max} = \max_i(O_i + t_{max} D_i) $$
  where $t_{min}$ and $t_{max}$ are the near and far clipping planes [S1].

**Parameters**
- Packet Size: Commonly 4 rays to match 128-bit SIMD register width [S1].
- Data Structure: k-dtree is a common choice for traversal [S1].

**Edge Cases and Pitfalls**
- Ray Divergence: High divergence within a packet reduces SIMD efficiency. Splitting packets or using smaller packet sizes may be necessary [S1].
- Precision: Conservative bounding volumes may lead to false positives in traversal, requiring additional intersection tests [S1].

## Subtopics

### [Four-Tier Hierarchical Ray Packet Marching](18629-four-tier-hierarchical-ray-packet-marching.md)
A variant that integrates an AABB module with a four-tier traversal hierarchy to handle ray divergence via a "lag-first" scheduler and Gaussian-filtered occupancy grids, reducing computational load in neural volume rendering.

## Operating characteristics
- **Time Complexity**: Reduces the number of intersection tests and traversal steps by exploiting spatial coherence [S1].
- **Memory Complexity**: Requires additional memory to store packet states and bounding volumes [S1].
- **Performance**: Highly effective for primary eye rays, hard shadow rays, soft shadow rays, and secondary rays in interactive systems [S1].
- **Strengths**: Enables SIMD optimization, reduces traversal overhead, and is widely supported in current interactive systems [S1].
- **Limitations**: Performance degrades with high ray divergence; requires careful packet management [S1].

## Background
Ray aggregation was introduced as a key technique for exploiting spatial coherence in ray tracing. It is particularly effective in grid-based and k-dtree-based rendering systems. The method has been widely adopted in interactive ray tracing due to its compatibility with SIMD hardware and its ability to handle various ray types efficiently [S1].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
