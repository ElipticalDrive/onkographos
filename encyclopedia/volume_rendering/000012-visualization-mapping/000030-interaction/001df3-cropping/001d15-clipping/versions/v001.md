# Clipping
Also called: Front Clipping for Perspective Projection, Near-plane clipping, Viewpoint-relative slab clipping

## Overview
Clipping is a technique to enable correct perspective projection in sliced-based volume rendering by defining a near clipping boundary relative to the viewpoint [S1, unofficial]. It solves the problem of incorrect depth impression in inside views, where parallel projection fails to provide accurate spatial cues [S1, unofficial]. Unlike general cropping which cuts along volume faces, this method clips against the camera's near plane and active clipping planes to ensure only relevant slabs are processed [S2, unofficial].

## How it works
**Inputs and outputs:**
- **Inputs:**
  - `ren`: A renderer pointer providing camera state (viewpoint, orientation, near plane distance) and active clipping plane definitions [S2, unofficial].
  - `worldBounds[6]`: A double array containing the unclipped world-space bounding box coordinates of the volume [S2, unofficial].
  - `vol`: A volume pointer providing the volume's transformation matrix and clipping state flags [S2, unofficial].
- **Outputs:**
  - A set of slabs (or primitives) that lie within the valid viewing frustum and beyond the near clipping plane.

**Data structures:**
- **Bounding Box:** Represented as an array of 6 doubles: `[minX, maxX, minY, maxY, minZ, maxZ]` in world coordinates [S2, unofficial].
- **Clipping Planes:** Defined by the renderer's state, typically as plane equations $ax + by + cz + d = 0$ [S2, unofficial].

**Steps:**
1. Retrieve the camera's near plane distance $d_{near}$ and the world-space bounding box `worldBounds` [S2, unofficial].
2. Transform the volume's bounding box into the camera's view space using the volume's transformation matrix and the camera's view matrix [S2, unofficial].
3. Clip the transformed bounding box against the camera's near plane. The near plane is defined by the distance from the viewpoint along the view direction [S1, unofficial].
4. Clip the resulting box against any additional active clipping planes defined in `ren` [S2, unofficial].
5. Determine the first slab to be processed. The user specifies or the system calculates the distance from the viewpoint to this first slab, ensuring it is $\ge d_{near}$ [S1, unofficial].
6. Render only slabs beyond this defined distance to ensure correct depth impression for inside views [S1, unofficial].

**Formulas:**
- Let $P_{view}$ be a point in view space. The near plane condition is:
  $$ P_{view} \cdot \mathbf{v}_{dir} \ge d_{near} $$
  where $\mathbf{v}_{dir}$ is the normalized view direction vector [S1, unofficial].
- For a slab at distance $d_{slab}$ from the viewpoint, it is rendered if:
  $$ d_{slab} \ge d_{near} $$
  [S1, unofficial].

**Parameters:**
- `ren`: VTK renderer pointer [S2, unofficial].
- `worldBounds`: Array of 6 doubles [S2, unofficial].
- `vol`: VTK volume pointer [S2, unofficial].

**Edge cases and pitfalls:**
- **Inside Views:** Without front clipping, perspective projection artifacts occur because slabs behind the viewpoint might be rendered or ordered incorrectly [S1, unofficial].
- **Precision:** Ensure that the near plane distance is not set to zero or negative values, which can cause division by zero or incorrect clipping logic [G].
- **Degenerate Input:** If the volume is entirely behind the near plane, no slabs are rendered [G].

## Subtopics

### Hard Cull
**Definition:** A clipping operator that keeps or drops each Gaussian primitive entirely based on whether its center lies within the half-space defined by the clip plane. It adds no per-pixel attenuation factor [S3, unofficial].
**Steps:** For each primitive, compute the dot product of the clip normal $\mathbf{n}$ and the primitive mean $\boldsymbol{\mu}$. If $\mathbf{n}^\top \boldsymbol{\mu} \le \tau$, keep the primitive; otherwise, drop it [S3, unofficial].
**Pitfalls:** Primitives straddling the clip plane either overshoot (if kept) or vanish (if dropped), producing a center-quantized boundary artifact as the plane sweeps [S3, unofficial]. The decision is non-differentiable [S3, unofficial].
**Comparison:** Unlike Moment-Matched (MM) and analytic operators, Hard Cull has no gradient through its binary decision and updates only kept primitives [S3, unofficial].

## Operating characteristics
**Time and memory complexity:**
- Time complexity is linear with respect to the number of slabs/primitives, as each is tested against the clipping planes [S2, unofficial].
- Memory complexity is minimal, requiring only the bounding box and plane definitions [S2, unofficial].

**Measured performance:**
- Not given in the sources.

**Scaling:**
- Scales well with the number of clipping planes, as each plane adds a constant-time test per slab [S2, unofficial].

**Strengths and limitations:**
- **Strengths:** Enables correct perspective projection for inside views, which is mandatory for accurate depth impression [S1, unofficial].
- **Limitations:** Does not handle partial occlusion or attenuation at the clipping boundary, leading to artifacts if primitives straddle the plane [S3, unofficial].

**When to use it and when not:**
- Use when correct perspective projection is required, especially for inside views [S1, unofficial].
- Do not use when smooth attenuation at the clipping boundary is needed, in which case Moment-Matched or analytic clipping is preferred [S3, unofficial].

## Background
**Origin:**
- The concept of front clipping for perspective projection was introduced to address limitations in sliced-based volume rendering systems like VolumePro, which lacked perspective projection support [S1, unofficial].
- The term "Front Clipping" was used in the Shear-Warp Factorization Volume Rendering System to solve the problem of incorrect depth impression in inside views [S1, unofficial].

**History and development:**
- Early volume rendering systems like VolumePro supported only parallel projection and back-to-front compositing [S1, unofficial].
- The SWF VRS improved this by introducing perspective projection and front clipping [S1, unofficial].
- Modern implementations, such as those in VTK, integrate clipping with camera and volume state to handle complex scenes [S2, unofficial].
- In the context of Gaussian Splatting, Hard Cull was isolated as a binary operator to analyze the effects of clipping without learned components [S3, unofficial].

**Where it is used:**
- Medical volume rendering, where inside views are common [S1, unofficial].
- General volume rendering systems that support perspective projection [S1, unofficial].
- Gaussian Splatting for volume rendering, as a baseline clipping operator [S3, unofficial].

## Sources
[S1] Shear-warp factorization volume rendering visualization system, https://archive.org/details/0162018299360, fetched 2026-10-26 (unofficial)
[S2] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S3] XClipGS: Exact Half-Space Clipping for Medical Volume Gaussian Splatting, https://doi.org/10.48550/arxiv.2608.07760, fetched 2026-10-26 (unofficial)
