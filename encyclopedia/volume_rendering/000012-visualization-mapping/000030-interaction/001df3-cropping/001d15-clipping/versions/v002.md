# Clipping
Also called: Front Clipping for Perspective Projection, Near-plane clipping, Viewpoint-relative slab clipping

## Overview
Clipping is a technique to enable correct perspective projection in sliced-based volume rendering by defining a near clipping boundary relative to the viewpoint [S1, unofficial]. It solves the problem of incorrect depth impression in inside views, where parallel projection fails to provide accurate spatial cues [S1, unofficial]. Unlike general cropping [Cropping](1648-cropping.md) which cuts along volume faces, this method clips against the camera's near/far planes and active clipping planes to ensure only relevant slabs are processed [S2, unofficial].

## How it works
**Inputs and outputs:**
- **Inputs:**
  - `ren`: A `vtkRenderer` pointer providing camera and clipping plane state [S2, unofficial].
  - `worldBounds[6]`: A double array containing the unclipped world-space bounding box coordinates [S2, unofficial].
  - `vol`: A `vtkVolume` pointer providing volume transformation and clipping state [S2, unofficial].
- **Outputs:**
  - A set of slabs/primitives that pass the clipping tests against the camera near/far planes and active clipping planes [S2, unofficial].

**Data structures:**
- **Bounding Box:** Represented as `worldBounds[6]`, a double array containing the unclipped world-space bounding box coordinates [S2, unofficial].
- **Clipping Planes:** Defined by the renderer's state (`ren`) [S2, unofficial].

**Steps:**
1. Retrieve the camera state and active clipping planes from `ren`, and the unclipped world-space bounding box from `worldBounds[6]` [S2, unofficial].
2. Clip the volume bounding box against all active clipping planes and the camera near/far planes [S2, unofficial].
3. The user specifies the distance from the viewpoint to the first slab to be processed [S1, unofficial].
4. Only slabs beyond this defined distance are rendered, ensuring correct depth impression for inside views [S1, unofficial].

**Formulas:**
- Let $d_{\text{slab}}$ be the distance from the viewpoint to a slab. The slab is processed if $d_{\text{slab}} \ge d_{\text{first}}$, where $d_{\text{first}}$ is the user-specified distance to the first slab [S1, unofficial].

**Parameters:**
- `ren`: `vtkRenderer` pointer [S2, unofficial].
- `worldBounds[6]`: Double array of unclipped world-space bounding box coordinates [S2, unofficial].
- `vol`: `vtkVolume` pointer [S2, unofficial].

**Edge cases and pitfalls:**
- **Inside Views:** Without front clipping, perspective projection artifacts occur because slabs behind the viewpoint might be rendered or ordered incorrectly [S1, unofficial].
- **Precision:** Not given in the sources.
- **Degenerate Input:** Not given in the sources.

## Subtopics

### Hard Cull
**Definition:** A clipping operator that keeps or drops each Gaussian primitive entirely based on whether its center lies within the half-space defined by the clip plane. It adds no per-pixel attenuation factor [S3, unofficial].
**Steps:** For each primitive, compute the dot product of the clip normal $\mathbf{n}$ and the primitive mean $\boldsymbol{\mu}$. If $\mathbf{n}^\top \boldsymbol{\mu} \le \tau$, keep the primitive; otherwise, drop it [S3, unofficial].
**Pitfalls:** Primitives straddling the clip plane either overshoot (if kept) or vanish (if dropped), producing a center-quantized boundary artifact as the plane sweeps [S3, unofficial]. The decision is non-differentiable [S3, unofficial].
**Comparison:** Unlike Moment-Matched (MM) and analytic operators, Hard Cull has no gradient through its binary decision and updates only kept primitives [S3, unofficial].

## Operating characteristics
**Time and memory complexity:**
- Not given in the sources.
**Measured performance:**
- Not given in the sources.
**Scaling:**
- Not given in the sources.
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
