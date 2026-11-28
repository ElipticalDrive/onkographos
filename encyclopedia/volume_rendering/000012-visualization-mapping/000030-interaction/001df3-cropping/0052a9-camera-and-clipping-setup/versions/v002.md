# Camera and Clipping Setup

Also called: View Reference and Clipping Planes

## Overview
Camera and Clipping Setup defines the initial camera parameters and clipping boundaries for focus+context volume rendering [S1, unofficial]. The view reference point is computed as the center of the focus clusters, while the projection window covers both focus and contextual clusters to keep the whole context visible while centering the image on the focus [S1, unofficial]. It integrates distance-based voxel culling for surface proximity filtering [S1, unofficial].

## How it works
**Inputs:**
- A set of identified clusters, categorized as "focus" and "context" [S1, unofficial].
- User-defined clipping plane parameters (initially derived from focus bounds) [S1, unofficial].
- A distance map voxel model containing the distance of each voxel to the nearest surface boundary [S1, unofficial].
- User-defined threshold for surface proximity culling [S1, unofficial].

**Outputs:**
- Camera view reference point [S1, unofficial].
- Camera projection window (bounding box) [S1, unofficial].
- Clipping plane positions [S1, unofficial].
- Culling mask for voxels based on distance threshold [S1, unofficial].

**Data Structures:**
- **Clusters:** List of 3D centroids $(x, y, z)$ for focus and context groups [S1, unofficial].
- **Distance Map:** 3D voxel grid where each voxel stores a scalar distance value $d$ to the nearest boundary [S1, unofficial].
- **Clipping Planes:** User-defined planes defined by normal vector $\mathbf{n}$ and distance $D$ from origin, or plane equation $Ax + By + Cz + D = 0$ [S1, unofficial].

**Steps:**
1. **Compute View Reference Point:** Calculate the centroid of the focus clusters [S1, unofficial]. Let $F$ be the set of focus cluster centers. The view reference point $\mathbf{V}_{ref}$ is:
   $$ \mathbf{V}_{ref} = \frac{1}{|F|} \sum_{c \in F} c $$
2. **Compute Projection Window:** Determine the axis-aligned bounding box (AABB) that encloses both the focus clusters and the context clusters [S1, unofficial]. Let $C$ be the set of context cluster centers. The projection window is the AABB of $F \cup C$ [S1, unofficial].
3. **Initialize Clipping Planes:** Set the initial position of user-defined clipping planes to coincide with the faces of the focus bounding box [S1, unofficial].
4. **Load Distance Map:** Load the third voxel model containing the precomputed distance map of all voxels to the nearest surface boundary [S1, unofficial].
5. **Apply Distance-Based Culling:** During rendering, for each voxel $v$ with distance value $d_v$ and user-defined threshold $T$: if $d_v > T$, cull the voxel (do not render) [S1, unofficial]. This applies to both focus and context voxels [S1, unofficial].
6. **Apply Opacity Modulation:** During 3D texture rendering, apply per-plane opacity modulation to all proxy polygons in front of the focus bounding box to make slices traversing only contextual information more transparent [S1, unofficial].

**Parameters:**
- **Distance Threshold ($T$):** User-defined value for surface proximity culling [S1, unofficial].
- **Clipping Plane Initial Position:** Bounding box of focus clusters [S1, unofficial].

**Edge Cases and Pitfalls:**
- **View Point Optimization:** Not currently applied; camera orientation is specified interactively by users [S1, unofficial].
- **Context Opacity:** The parser sets the opacity of the Color Transfer Function (CTF) to zero for context voxels to prevent them from being rendered during texture slicing, while context bounding boxes are shown [S1, unofficial]. Opacity modulation is then applied to proxy polygons in front of the focus bounding box [S1, unofficial].

## Subtopics

### [Shader Clip Plane Setup](18337-shader-clip-plane-setup.md)
A variant where clipping is implemented via shader uniforms. The method passes a 4D homogeneous plane equation to a shader to enable custom clipping of rendered geometry. The plane equation $Ax + By + Cz + D = 0$ is uploaded as a 4-component float uniform `clip_eqn[0..3]`. The shader program is bound, the uniform location is retrieved, and the plane components are uploaded as floats. This allows for dynamic, per-fragment clipping decisions based on the plane equation.

## Operating characteristics
Not given in the sources.

## Background
This method originates from research on structural focus+context rendering of multiclassified volume data [S1, unofficial]. It integrates cluster-based view framing with distance-based culling to enhance the visualization of complex, multi-class volumetric datasets [S1, unofficial].

## Sources
[S1] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
