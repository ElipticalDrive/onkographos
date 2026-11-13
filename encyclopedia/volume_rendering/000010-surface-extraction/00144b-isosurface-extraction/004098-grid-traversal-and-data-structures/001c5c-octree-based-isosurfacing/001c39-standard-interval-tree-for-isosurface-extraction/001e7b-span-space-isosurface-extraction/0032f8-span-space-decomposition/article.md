# Span Space Decomposition
Also called: span space techniques
## Overview
Span space decomposition is a family of isosurface extraction methods that partition volume cells by mapping their extreme scalar values into a 2D coordinate system [S1]. It solves the problem of efficiently identifying cells intersecting an isosurface by reducing the search to a 2D range query [S1]. It differs from sibling methods by explicitly leveraging the geometric properties of min/max value pairs in a 2D plane rather than relying solely on hierarchical spatial subdivision of the domain [G].
## Shared principles
The core mechanism involves mapping each volume cell to a point in a 2D space based on its scalar field extrema [S1].
### Mapping
Each cell $c$ is mapped to a point $P_c = (x, y)$ in span space, where $x = \min_{v \in c} f(v)$ and $y = \max_{v \in c} f(v)$ [S1]. By definition, $y \ge x$, so all points lie on or above the line $y = x$ [S1].
### Intersection Condition
For a given isovalue $v$, a cell intersects the isosurface if and only if $x < v$ and $y > v$ [S1]. This defines a rectangular region in the 2D span space bounded by $x = v$ and $y = v$ [S1].
## Subtopics

### NOISE Algorithm
The primary implementation of span space decomposition introduced by Livnat et al.. It overlays a kd-tree on the span space points to accelerate seed set construction. It is chosen when preprocessing time must be minimized for large datasets like MRI or CT scans, though it may still generate large seed sets for noisy data. See [NOISE Algorithm](1751-noise-algorithm.md).

## Background
The span space concept was introduced by Livnat et al. in the context of the NOISE algorithm [S1]. The technique partitions cells based on their extreme values to accelerate isosurface extraction [S1].
## Sources
[S1] A Case Study of Isosurface Extraction Algorithm Performance, https://doi.org/10.1007/978-3-7091-6783-0_26, fetched 2026-10-26
