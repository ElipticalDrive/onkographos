# Object Combination Operators
Also called: SDF boolean operators, smooth minimum alternatives

## Overview
Object Combination Operators are constructive modeling primitives that merge two Signed Distance Functions (SDFs), $a$ and $b$, into a single distance field $d_{\text{out}}$ to create rounded edges, chamfers, or smooth transitions, replacing the sharp geometric intersections of standard boolean logic [S1, unofficial][S2, unofficial]. Unlike the standard `min` operator used in [Constructive SDF Modeling (Min Combination)](871-constructive-sdf-modeling-min-combination.md), these operators introduce a radius parameter $r > 0$ to control the feature size, aiming to preserve Lipschitz continuity ($|\nabla d| \le 1$) better than naive smooth minimums when surface gradients are orthogonal [S1, unofficial][S2, unofficial].

## Shared principles
All operators in this family accept two scalar distance values $a$ and $b$ and a scalar radius parameter $r > 0$ [S1, unofficial][S2, unofficial]. The parameter $r$ specifies the radius of the feature and should be much smaller than the scale of the objects to prevent geometric distortion [S1, unofficial][S2, unofficial]. The core mathematical challenge is maintaining the Lipschitz condition ($|\nabla d| \le 1$) [S1, unofficial][S2, unofficial]. The resulting SDF preserves Lipschitz continuity only if the gradients of the two input SDFs are orthogonal (at right angles) [S1, unofficial][S2, unofficial]. If the angle between gradients deviates by more than $30^\circ$, or if surfaces are parallel and close, continuity is lost, leading to ray marching artifacts such as overshooting or jittering [S1, unofficial][S2, unofficial]. While standard boolean operators use `min` (union) and `max` (intersection), combination operators use blended formulas to approximate these operations [S1, unofficial][S2, unofficial]. Common structural patterns in the literature include polynomial blends, geometric constructions using vector lengths, and soft approximations [S1, unofficial][S2, unofficial].

## Subtopics

### [SDF Smooth Union](1817-sdf-smooth-union.md)
Blends two SDFs into a single smooth distance field using polynomial or exponential formulations, replacing sharp intersections with a rounded transition controlled by $k$. Choose this for general smooth merging where curvature control is needed.

### [Smooth Intersection](1839-smooth-intersection.md)
Blends the overlap region of two shapes to produce rounded edges, serving as a smooth approximation of boolean intersection. It uses polynomial or geometric root formulations to create a continuous transition. Choose this when you need to merge interiors smoothly rather than exteriors.

### [Multi-Material Distance Fields](1943-multi-material-distance-fields.md)
Extends standard SDFs to return both a geometric distance and a material identifier (e.g., color index) for every point. This allows CSG operations to preserve material assignments during boolean combinations. Choose this when per-object shading or texturing is required within a single ray-marched scene.

### [Linear SDF Blend](3004-linear-sdf-blend.md)
Interpolates between two SDFs using a linear blend factor $a \in [0, 1]$ to create smooth transitions or morphing. Unlike rounded boolean unions, it produces a linear interpolation of distance values. Choose this for morphing animations or soft blending where geometric accuracy of the boundary is less critical than visual smoothness.

## Background
The operators were documented in community repositories and shader codebases to provide constructive SDF modeling primitives that produce correct distance bounds or approximations, distinguishing them from naive "smooth minimum" operators that may violate Lipschitz continuity [S1, unofficial][S2, unofficial]. The implementations include specific operators for chamfers, rounds, columns, stairs, soft blends, pipes, engravings, grooves, and tongues [S1, unofficial][S2, unofficial].

## Sources
[S1] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
