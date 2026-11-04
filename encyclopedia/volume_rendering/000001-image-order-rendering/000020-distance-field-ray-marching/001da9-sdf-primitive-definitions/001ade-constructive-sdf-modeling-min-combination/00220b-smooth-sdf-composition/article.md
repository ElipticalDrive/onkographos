# Smooth SDF Composition

Also called: Signed Distance Function (SDF) Scene Composition, Smooth Min SDF, Smooth Union of SDFs

## Overview
Smooth SDF Composition blends multiple primitive Signed Distance Functions (SDFs) into a single continuous field using a smooth minimum operator, avoiding the sharp creases produced by standard constructive SDF modeling [S1, unofficial]. Unlike the nearest-surface control of [Constructive SDF Modeling (Min Combination)](871-constructive-sdf-modeling-min-combination.md), this method creates a transitional volume between objects, enabling organic merging effects in ray marching [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Two or more scalar SDF values ($d_1, d_2, \dots$) evaluated at a point $p$, and a blending factor $k \ge 0$.
- Output: A single blended distance value $d_{smooth}$.

**Data Structures:**
No specific data structures are required beyond the scalar values returned by individual primitive SDF evaluators (e.g., `sdfTorus`, `sdfBox` as defined in [S1, unofficial]).

**Steps:**
1. Evaluate the individual SDF for each primitive at the query point $p$. For example, calculate $d_{torus}$ and $d_{box}$ [S1, unofficial].
2. Select a blending factor $k$. A typical value is $0.5$ for moderate blending [S1, unofficial].
3. Compute the smooth minimum of the distances using the formula:
   $$ d_{smooth} = -\frac{1}{k} \ln(e^{-k d_1} + e^{-k d_2}) $$
   This formula approximates the minimum while maintaining $C^1$ continuity [G].
4. Return $d_{smooth}$ as the distance to the nearest object in the blended scene [S1, unofficial].

**Parameters:**
- $k$: Blending factor. Controls the width of the transition zone. Larger $k$ approaches the hard minimum; smaller $k$ creates a smoother, wider blend. Typical value: $0.5$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $k=0$, the formula involves division by zero. In practice, if $k$ is very small, the result approaches the arithmetic mean or requires a fallback to the standard minimum [G].
- Precision issues may arise if $d_1$ and $d_2$ differ significantly in magnitude, causing one exponential term to dominate and underflow/overflow [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def smooth_min(d1: float, d2: float, k: float) -> float:
    if k == 0.0:
        return min(d1, d2)
    # Clamp to prevent exp() overflow for large positive distances [G]
    k_d1 = -k * max(d1, -100.0)
    k_d2 = -k * max(d2, -100.0)
    exp1 = math.exp(k_d1)
    exp2 = math.exp(k_d2)
    return -1.0 / k * math.log(exp1 + exp2)

def evaluate_scene_sdf(primitives, k):
    if not primitives:
        return float('inf')
    current_dist = primitives[0]()
    for i in range(1, len(primitives)):
        next_dist = primitives[i]()
        current_dist = smooth_min(current_dist, next_dist, k)
    return current_dist
```

## Subtopics

### [Scene SDF Evaluation](4452-scene-sdf-evaluation.md)
Computes the minimum signed distance from a point to any primitive in the scene, effectively performing a union operation. It differs from [Smooth SDF Composition](2685-smooth-sdf-composition.md) by aggregating multiple distinct primitives into a single scene-level distance field rather than composing a single smooth blend of shapes.

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Strengths: Enables organic, fluid merging of shapes without sharp edges [S1, unofficial].
- Limitations: The smooth transition zone can obscure the true boundary of objects, potentially causing ray marching artifacts if the step size is not adjusted for the gradient changes in the blend region [G].

## Background
Not given in the sources.

## Sources
[S1] Sample - Ray Marching and Signed Distance Functions, https://github.com/ComputeNode/cyfra/pull/13, fetched 2026-10-27 (unofficial)
