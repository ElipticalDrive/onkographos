# Camera and Clipping Setup

Also called: View Reference and Clipping Planes

## Overview
Camera and Clipping Setup defines the initial camera parameters and clipping boundaries for focus+context volume rendering, ensuring the view is centered on relevant data clusters while allowing interactive exploration of surrounding context. It differs from standard cropping by dynamically computing the view reference point and projection window based on semantic cluster centers rather than static volume bounds, and by integrating distance-based voxel culling for surface proximity filtering.

## How it works
**Inputs:**
- A set of identified clusters, categorized as "focus" and "context".
- User-defined clipping plane parameters (initially derived from focus bounds).
- A distance map voxel model containing the distance of each voxel to the nearest surface boundary.
- User-defined threshold for surface proximity culling.

**Outputs:**
- Camera view reference point.
- Camera projection window (bounding box).
- Clipping plane positions.
- Culling mask for voxels based on distance threshold.

**Data Structures:**
- **Clusters:** List of 3D centroids $(x, y, z)$ for focus and context groups.
- **Distance Map:** 3D voxel grid where each voxel stores a scalar distance value $d$ to the nearest boundary.
- **Clipping Planes:** Six planes (or user-defined set) defined by normal vector $\mathbf{n}$ and distance $D$ from origin, or plane equation $Ax + By + Cz + D = 0$.

**Steps:**
1. **Compute View Reference Point:** Calculate the centroid of the focus clusters. Let $F$ be the set of focus cluster centers. The view reference point $\mathbf{V}_{ref}$ is:
   $$ \mathbf{V}_{ref} = \frac{1}{|F|} \sum_{c \in F} c $$
2. **Compute Projection Window:** Determine the axis-aligned bounding box (AABB) that encloses both the focus clusters and the context clusters. Let $C$ be the set of context cluster centers. The projection window is the AABB of $F \cup C$.
3. **Initialize Clipping Planes:** Set the initial position of user-defined clipping planes to coincide with the faces of the focus bounding box. This ensures the initial view is tightly cropped around the focus region.
4. **Load Distance Map:** Load the third voxel model containing the precomputed distance map of all voxels to the nearest surface boundary.
5. **Apply Distance-Based Culling:** During rendering, for each voxel $v$ with distance value $d_v$ and user-defined threshold $T$:
   - If $d_v > T$, cull the voxel (do not render).
   - This applies to both focus and context voxels.
6. **Apply Opacity Modulation:** During 3D texture rendering, for proxy polygons in front of the focus bounding box:
   - Apply per-plane opacity modulation to make slices traversing only contextual information more transparent. This reduces visual clutter from context data.

**Parameters:**
- **Distance Threshold ($T$):** User-defined value for surface proximity culling. Typical values depend on volume resolution and feature scale.
- **Clipping Plane Initial Position:** Bounding box of focus clusters.

**Edge Cases and Pitfalls:**
- **View Point Optimization:** Not currently applied. Camera orientation is specified interactively by users, which may lead to suboptimal viewing angles if not manually adjusted.
- **Context Opacity:** The context bounding boxes are shown, but the parser sets the opacity of the Color Transfer Function (CTF) to zero for context voxels during initial rendering to prevent them from being rendered during texture slicing. Opacity modulation is then applied to proxy polygons.

## Subtopics

### Shader Clip Plane Setup
A variant where clipping is implemented via shader uniforms. The method passes a 4D homogeneous plane equation to a shader to enable custom clipping of rendered geometry. The plane equation $Ax + By + Cz + D = 0$ is uploaded as a 4-component float uniform `clip_eqn[0..3]`. The shader program is bound, the uniform location is retrieved, and the plane components are uploaded as floats. This allows for dynamic, per-fragment clipping decisions based on the plane equation.

## Operating characteristics
- **Time Complexity:** Computing the view reference point and projection window is $O(N)$ where $N$ is the number of clusters. Distance-based culling is $O(1)$ per voxel during rendering.
- **Memory Complexity:** Requires storage for the distance map, which is $O(V)$ where $V$ is the number of voxels in the volume.
- **Strengths:** Provides semantic focus+context visualization. Distance-based culling allows for detailed surface exploration by hiding distant internal structures.
- **Limitations:** Relies on user interaction for camera orientation. Context data may still cause visual clutter if opacity modulation is not carefully tuned.
- **When to Use:** When semantic clustering of volume data is available and the user needs to explore specific regions while maintaining awareness of surrounding context.

## Background
This method originates from research on structural focus+context rendering of multiclassified volume data. It integrates cluster-based view framing with distance-based culling to enhance the visualization of complex, multi-class volumetric datasets. The approach is particularly useful in scientific visualization where identifying and exploring specific features within a larger context is critical.

## Sources
[S1] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
[S2] qqdiguo/CloudRender-PointSplatting, https://github.com/qqdiguo/CloudRender-PointSplatting, fetched 2026-10-28 (unofficial)
