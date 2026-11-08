# Infinite Tiling with Modulo

Also called: Infinite Crystal Grid SDF, Modulo SDF, Modulo Tiling, Tiling, Tiling via Modulo

## Overview
Infinite Tiling with Modulo is a technique for rendering infinite, repeating patterns by folding space using the modulo operator, allowing a single primitive Signed Distance Function (SDF) to be tiled infinitely [S1, S2, unofficial]. It enables ray-marched fractal renderers (e.g., Mandelbulbs, Menger sponges) that would be impossible to represent as explicit meshes [S2, unofficial].

## How it works
**Inputs:**
- Query point $\mathbf{p} \in \mathbb{R}^3$ in world space [G].
- Tile size (period) $T \in \mathbb{R}$ (scalar or vector) [S1, unofficial].
- Offset vector $\mathbf{o} \in \mathbb{R}^3$ (optional, for centering) [S1, unofficial].
- Primitive SDF function $d_{\text{prim}}(\mathbf{q})$ [G].

**Outputs:**
- Signed distance $d \in \mathbb{R}$ to the nearest instance of the primitive [G].

**Data Structures:**
- None required beyond standard floating-point variables [G].

**Steps:**
1. **Fold Space:** Transform the query point $\mathbf{p}$ into the fundamental tile domain. The standard approach centers the tile at the origin [S1, unofficial].
   $$ \mathbf{q} = \text{mod}(\mathbf{p} + \mathbf{o}, T) - \frac{T}{2} $$
   Where $\text{mod}(x, T)$ returns the remainder of $x/T$ such that the result is in $[0, T)$ [G]. The offset $\mathbf{o}$ is typically $T/2$ to center the tile at $\mathbf{0}$ [S1, unofficial].
   
2. **Evaluate Primitive:** Compute the distance to the primitive shape using the folded coordinate $\mathbf{q}$:
   $$ d = d_{\text{prim}}(\mathbf{q}) $$

**Parameters:**
- Tile Size $T$: The period of the modulo operation. In [S1, unofficial], $T=3.0$.
- Offset $\mathbf{o}$: Added before modulo to center the tile. In [S1, unofficial], $\mathbf{o} = (1.5, 1.5, 1.5)$ for $T=3.0$.

**Edge Cases and Pitfalls:**
- **Boundary Precision:** Points exactly on the tile boundary may suffer from floating-point precision issues, causing visual artifacts or incorrect distances [G].
- **Positive Quadrant Requirement:** Some implementations of infinite tiling require the position to be in the positive quadrant for the modulo operation to work correctly without negative wrapping artifacts [S3, unofficial]. The centering offset $\mathbf{o}$ mitigates this by shifting the domain [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def infinite_tiling_modulo(p, tile_size, offset, sdf_prim):
    """
    Computes the SDF for an infinitely tiled primitive using modulo folding.
    Operations are component-wise on vectors.
    """
    # Step 1: Fold space into the fundamental domain centered at origin
    # Python's % operator matches GLSL mod(a, p) for positive p: a - p * floor(a/p)
    q = (p + offset) % tile_size - (tile_size / 2.0)
    
    # Step 2: Evaluate the primitive SDF at the folded coordinate
    d = sdf_prim(q)
    
    return d
```

## Subtopics

### Finite Tiling
A variant of Infinite Tiling which produces a finite number of repetitions of a shape [S3, unofficial]. It takes an additional vector $\vec{l}$ as input, where $\vec{l}$ is the corner of the box used to limit the number of repetitions of the shape [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray step, identical to evaluating a single primitive [G]. The modulo operation is a constant-time arithmetic operation [G].
- **Memory Complexity:** $O(1)$. No additional data structures are required [G].
- **Performance:** Enables rendering of infinite detail at essentially zero extra cost [S2, unofficial]. This allows for real-time frame rates even as scene complexity grows, provided other optimizations (bounding-volume early-outs, reduced step counts) are used [S2, unofficial].
- **Strengths:** Extremely efficient for creating complex, repeating structures and fractals. No mesh storage or LOD management required [G].
- **Limitations:** Only supports periodic tiling. Cannot easily create non-repeating or irregular patterns without combining with other techniques [G]. Normal estimation at tile boundaries can be noisy due to derivative discontinuities [G].

## Background
The technique is foundational to ray-marched fractal rendering, enabling the creation of Mandelbulbs and Menger sponges [S2, unofficial]. It leverages the mathematical property of periodicity to reuse a single SDF definition across an infinite domain. The method is widely used in demoscene productions and real-time ray marching demos due to its efficiency and visual impact [S1, S2, unofficial].

## Sources
[S1] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S2] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S3] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
