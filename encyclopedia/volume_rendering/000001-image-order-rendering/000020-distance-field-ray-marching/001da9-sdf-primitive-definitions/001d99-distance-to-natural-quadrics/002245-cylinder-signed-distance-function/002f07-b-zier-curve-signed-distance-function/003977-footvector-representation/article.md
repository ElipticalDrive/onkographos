# Footvector Representation

Also called: Foot mapping

## Overview
The Footvector Representation is a geometric generalization of Signed Distance Functions (SDFs) for curves and surfaces, defined by the vector pointing from a query point to its closest point (footpoint) on the shape [S1, abstract only, unofficial]. Unlike standard SDFs which only provide scalar distance, this representation retains directional information, enabling exact offset surface computation for Constructive Solid Geometry (CSG) operations via fast iterative methods [S1, abstract only, unofficial]. It is chosen when precise geometric invariants and exact offsetting are required, though it poses implementation challenges for GPU shaders due to higher-order function mappings [S1, abstract only, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $x \in \mathbb{R}^n$ and a parametric representation of a curve or surface.
*   **Output:** The footvector $v(x)$, which is the vector from $x$ to the footpoint $p$ on the shape, such that $v(x) = p - x$. The magnitude $\|v(x)\|$ serves as the distance estimate.

**Data Structures:**
Not given in the sources. The representation relies on function mappings rather than explicit grid structures.

**Method Steps:**
1.  **Characterization:** Define the footvector mapping $v(x)$ using differential geometric invariants of the represented shape. The spatial partial derivatives of the footvector mapping depend on the principal curvatures at the footpoint [S1, abstract only, unofficial].
2.  **Precision Quantification:** Determine the precision factor $q_g(x)$ for a distance estimate $g$. If there exists a global precision $c > 0$ such that $0 < c \le q_g(x) \le 1$, then $g$ is a valid signed distance function estimate [S1, abstract only, unofficial].
3.  **CSG Computation:** Compute the exact footvector mapping for offset surfaces of CSG trees using fast iterative methods. For intersection operations (e.g., $q_{\max(g_1, g_2)}(x)$), the computation can be expensive [S1, abstract only, unofficial].
4.  **Set Operations:** Implement set operations as higher-order functions that map a tuple of footpoint mapping functions to a single function [S1, abstract only, unofficial].
5.  **GPU Implementation:** Transform CSG trees into GLSL shader code using a code generation framework to handle the complexity of higher-order functions on the GPU [S1, abstract only, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Conceptual implementation of the footvector mapping and CSG combination
# Note: Exact iterative solver details are not provided in the source [S1, abstract only, unofficial]

def compute_footvector(query_point, shape_params):
    """
    Computes the vector from query_point to the closest point (footpoint) on the shape.
    Returns the footvector v.
    """
    # Step 1: Find footpoint p using iterative method
    # The source mentions "fast iterative methods" but does not specify the algorithm [S1, abstract only, unofficial]
    footpoint = find_closest_point_iterative(query_point, shape_params)
    
    # Step 2: Return the vector
    return footpoint - query_point

def combine_footvectors(func_tuple, query_point):
    """
    Implements set operations (e.g., union, intersection) as higher-order functions.
    Maps a tuple of footvector functions to a single result.
    """
    # The source states these are higher-order functions but does not provide the specific composition logic [S1, abstract only, unofficial]
    # For intersection, precision q_max(g1, g2) is computed, which is expensive [S1, abstract only, unofficial]
    results = [f(query_point) for f in func_tuple]
    return combine_results(results)

def generate_glsl(cs_tree):
    """
    Transforms CSG trees to GLSL shader code.
    """
    # Code generation framework details are not provided [S1, abstract only, unofficial]
    pass
```

**Parameters:**
*   **Global precision $c$:** A constant such that $0 < c \le q_g(x) \le 1$. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Intersection Cost:** Computing the footvector for intersection operations (e.g., $\max(g_1, g_2)$) can be expensive [S1, abstract only, unofficial].
*   **GPU Implementation:** The use of higher-order functions for set operations poses challenges for GPU implementations [S1, abstract only, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, intersection operations are noted as potentially expensive [S1, abstract only, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides a geometric generalization of SDFs; enables exact offset surface computation for CSG trees; characterized by differential geometric invariants [S1, abstract only, unofficial].
*   **Limitations:** Computationally expensive for intersections; difficult to implement directly on GPUs without code generation [S1, abstract only, unofficial].
*   **When to use:** When exact footvectors and offset surfaces for CSG operations are required, and CPU-side or generated shader code is acceptable.
*   **When not to use:** When real-time GPU performance is critical and the overhead of higher-order function mapping is prohibitive.

## Background
The Footvector Representation was proposed by Gábor Valasek, Csaba Bálint, and András Leitereg in a paper titled "Footvector Representation of Curves and Surfaces" published in August 2021 (with a preprint in January 2021) [S1, abstract only, unofficial]. It is used in the context of volume rendering and geometric modeling, specifically as a generalization of signed distance functions to support complex CSG operations on curves and surfaces [S1, abstract only, unofficial].

## Sources
[S1] Footvector Representation of Curves and Surfaces, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
