# Object Combination Operators
Also called: SDF boolean operators, smooth minimum alternatives

## Overview
Object Combination Operators are constructive modeling primitives that merge two Signed Distance Functions (SDFs), $a$ and $b$, into a single distance field $d_{out}$ to create rounded edges, chamfers, or smooth transitions, replacing the sharp geometric intersections of standard boolean logic [S1, S2, unofficial]. Unlike the standard `min` operator used in [Constructive SDF Modeling (Min Combination)](871-constructive-sdf-modeling-min-combination.md), these operators introduce a radius parameter $k$ (or $r$) to control the feature size, aiming to preserve Lipschitz continuity ($|\nabla d| \le 1$) better than naive smooth minimums when surface gradients are orthogonal [S1, S2, unofficial].

## Shared principles
All operators in this family accept two scalar distance values $a$ and $b$ and a scalar radius parameter $k > 0$ (sometimes denoted $r$) [S1, S2, unofficial]. The parameter $k$ defines the width of the transition region or feature size; it must be significantly smaller than the scale of the objects to prevent geometric distortion [S1, S2, unofficial].

The core mathematical challenge is maintaining the Lipschitz condition. The resulting SDF preserves Lipschitz continuity only if the gradients of the two input SDFs are orthogonal (at right angles) [S1, S2, unofficial]. If the angle between gradients deviates by more than $30^\circ$, or if surfaces are parallel and close, continuity is lost, leading to ray marching artifacts such as overshooting or jittering [S1, S2, unofficial].

While standard boolean operators use `min` (union) and `max` (intersection), combination operators use blended formulas. Common structural patterns include:
1.  **Polynomial Blends:** Using cubic or quadratic interpolation terms like $k \cdot h \cdot (1-h)$ to subtract or add a rounded profile [S1, S2, unofficial].
2.  **Exponential Blends:** Using LogSumExp approximations for smoothness [S4, unofficial].
3.  **Geometric/Root Blends:** Using Euclidean distance calculations (e.g., $\sqrt{\dots}$) to construct rounded corners [S3, unofficial].
4.  **Chamfer/Round Specifics:** Some operators explicitly construct chamfered or rounded profiles using vector lengths and clamping [S2, unofficial].

## Subtopics

### [SDF Smooth Union](1817-sdf-smooth-union.md)
Blends two SDFs into a single smooth distance field using polynomial (cubic) or exponential (LogSumExp) formulations, replacing sharp intersections with a rounded transition controlled by $k$. Choose this for general smooth merging where curvature control is needed.

### [Smooth Intersection](1839-smooth-intersection.md)
Blends the overlap region of two shapes to produce rounded edges, serving as a smooth approximation of boolean intersection. It uses polynomial or geometric root formulations to create a continuous transition. Choose this when you need to merge interiors smoothly rather than exteriors.

### [Multi-Material Distance Fields](1943-multi-material-distance-fields.md)
Extends standard SDFs to return both a geometric distance and a material identifier (e.g., color index) for every point. This allows CSG operations to preserve material assignments during boolean combinations. Choose this when per-object shading or texturing is required within a single ray-marched scene.

### [Linear SDF Blend](3004-linear-sdf-blend.md)
Interpolates between two SDFs using a linear blend factor $a \in [0, 1]$ to create smooth transitions or morphing. Unlike rounded boolean unions, it produces a linear interpolation of distance values. Choose this for morphing animations or soft blending where geometric accuracy of the boundary is less critical than visual smoothness.

## Background
The concept of combining SDFs to create features like rounded edges and chamfers originated in the context of ray marching and procedural modeling, popularized by blogs and community repositories such as *kushchevd/RayMarching* and *yeataro/TD-Raymarching-System* [S1, S2, unofficial]. These sources document operators like `fOpUnionChamfer`, `fOpUnionRound`, and `fOpUnionSoft`, which aim to produce correct distance bounds or approximate distances, distinguishing them from "smooth minimum" operators that may violate Lipschitz continuity [S1, S2, unofficial]. The specific constraint regarding the $30^\circ$ gradient deviation and parallel surfaces highlights the mathematical rigor required to ensure stable ray marching steps [S1, S2, unofficial].

## Sources
[S1] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
