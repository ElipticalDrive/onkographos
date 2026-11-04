# Kalra & Barr Implicit Surface Intersection
Also called: Kalra-Barr method, LG-implicit surface intersection

## Overview
Kalra & Barr Implicit Surface Intersection is a robust algorithm for finding the nearest intersection between a ray and an implicit surface $f(\mathbf{x}) = 0$ [S1, unofficial]. It solves the problem of missing fine surface details or grazing intersections inherent in standard discrete ray marching by guaranteeing numerical convergence to the nearest root without requiring fine-tuning or human choice of interactive parameters [S1, unofficial]. It distinguishes itself from its parent [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) by providing a specific mathematical framework for "LG-implicit surfaces" that ensures no features are missed, and from its sibling [Ray Tracing Implicit Surfaces](2814-ray-tracing-implicit-surfaces.md) by its reliance on global Lipschitz and gradient bounds rather than local sampling heuristics [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
- Implicit surface function $f: \mathbb{R}^3 \to \mathbb{R}$.
- Upper bound $L$: limits the net rate of change of $f$ (Lipschitz constant).
- Upper bound $G$: limits the rate of change of the gradient $\nabla f$ (bound on the Hessian norm).
- Search interval $[t_{min}, t_{max}]$ along the ray.

**Outputs:**
- Intersection distance $t^*$ if a root exists in $[t_{min}, t_{max}]$.
- Failure status if no root is found or bounds are violated.

**Data Structures:**
- Interval $[t_a, t_b]$ representing the current search segment along the ray.
- Function value $f(\mathbf{x})$ and gradient $\nabla f(\mathbf{x})$ at query points.

**Steps:**
1. **Initialization:** Set current interval $[t_a, t_b] = [t_{min}, t_{max}]$.
2. **Evaluation:** Evaluate $f(\mathbf{x}_a)$ and $f(\mathbf{x}_b)$ where $\mathbf{x}_a = \mathbf{o} + t_a \mathbf{d}$ and $\mathbf{x}_b = \mathbf{o} + t_b \mathbf{d}$.
3. **Sign Check:** If $f(\mathbf{x}_a)$ and $f(\mathbf{x}_b)$ have opposite signs, a root exists in $[t_a, t_b]$. Proceed to bisection. If not, the algorithm may terminate or expand the search, depending on implementation [S1, unofficial].
4. **Bisection:**
   - Compute midpoint $t_m = (t_a + t_b) / 2$.
   - Evaluate $f(\mathbf{x}_m)$.
   - If $|f(\mathbf{x}_m)| < \epsilon$ (tolerance), return $t_m$.
   - If $f(\mathbf{x}_a)$ and $f(\mathbf{x}_m)$ have opposite signs, set $t_b = t_m$. Else, set $t_a = t_m$.
5. **Convergence Check:** If $t_b - t_a < \delta$ (precision threshold), return $(t_a + t_b) / 2$. Else, repeat Step 4.

**Formulas:**
- Lipschitz condition: $|f(\mathbf{x}) - f(\mathbf{y})| \le L \|\mathbf{x} - \mathbf{y}\|$ [S1, unofficial].
- Gradient bound: $\|\nabla f(\mathbf{x}) - \nabla f(\mathbf{y})\| \le G \|\mathbf{x} - \mathbf{y}\|$ [S1, unofficial].

**Parameters:**
- $L$: Not given in the sources.
- $G$: Not given in the sources.
- $\epsilon$: Not given in the sources.
- $\delta$: Not given in the sources.

**Edge Cases and Pitfalls:**
- If $L$ or $G$ are underestimated, the guarantee of finding the nearest intersection is lost [S1, unofficial].
- The algorithm assumes the surface is an "LG-implicit surface" [S1, unofficial].

## Subtopics

### [Level-synchronized bisection ray casting](1313-level-synchronized-bisection-ray-casting.md)
Level-synchronized bisection ray casting is a robust algorithm for rendering the zero-level-set of an implicit function $f: \mathbb{R}^3 \to \mathbb{R}$ by processing all active rays in lockstep (batch) using interval arithmetic to guarantee no surface features are missed. It solves the problem of missing fine surface details or grazing intersections inherent in standard discrete ray marching by bounding the function value over continuous segments and iteratively bisecting intervals until convergence. It distinguishes itself from its parent [Kalra & Barr Implicit Surface Intersection](1278-kalra-barr-implicit-surface-intersection.md) by explicitly leveraging interval arithmetic for inclusion testing and batch synchronization for performance, and from its sibling [Bisection Ray Tracing with Interval Inclusion](1314-bisection-ray-tracing-with-interval-inclusion.md) by its specific implementation of level-synchronized processing and Hermite-based root estimation.

### [Bisection Ray Tracing with Interval Inclusion](1314-bisection-ray-tracing-with-interval-inclusion.md)
Bisection Ray Tracing with Interval Inclusion is a robust ray-surface intersection algorithm that guarantees finding the nearest intersection of an implicit surface with a ray without missing fine features. It solves the problem of precision loss and feature skipping inherent in fixed-step ray marching by using interval arithmetic to prune intervals that cannot contain a root, followed by bisection to narrow down the interval containing the sign change of the implicit function $f$. Unlike standard bisection which processes rays independently, this method processes rays in batches and uses front-to-back pruning to discard intervals that are behind already found hits, improving efficiency for scenes with multiple surfaces or occlusion. It builds on [Kalra & Barr Implicit Surface Intersection](1278-kalra-barr-implicit-surface-intersection.md) by providing a specific batched, interval-arithmetic-based implementation strategy.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Guaranteed to find the nearest intersection; guaranteed not to miss fine features; no fine tuning required [S1, unofficial].
- **Limitations:** Requires computation of $L$ and $G$ bounds; may be slower than heuristic methods if bounds are loose [S1, unofficial].
- **When to use:** When robustness and feature preservation are critical, and bounds $L$ and $G$ are known or can be estimated [S1, unofficial].
- **When not to use:** When performance is paramount and approximate results are acceptable [S1, unofficial].

## Background
The algorithm was presented by Devendra Kalra and Alan Barr in July 1989 [S1, unofficial]. It significantly extends the ability to intersect rays with implicit surfaces in a guaranteed fashion, addressing limitations of existing schemes that worked only for limited sets of implicit functions or were ad-hoc [S1, unofficial].

## Sources
[S1] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/390921688_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
