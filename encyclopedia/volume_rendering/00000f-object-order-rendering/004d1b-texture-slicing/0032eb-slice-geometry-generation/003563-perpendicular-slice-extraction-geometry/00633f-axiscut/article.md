# AxisCut

Also called: Axis-aligned cut filter

## Overview
AxisCut is a filter that cuts a hypertree grid using axis-aligned planes, producing 2D Adaptive Mesh Refinement (AMR) outputs [S1, unofficial]. It is designed for visualizing cross-sections of tree-based AMR simulations with rectilinear geometry [S1, unofficial]. It is distinguished from its sibling [PlaneCutter](21407-plane-cutter.md) by its speed and output format: AxisCut produces 2D AMR representations suitable for visualization, whereas PlaneCutter produces conforming polygonal mesh data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3-dimensional hypertree grid (typically ternary) with rectilinear geometry [S1, unofficial].
*   **Output:** One or more 2D AMR representations [S1, unofficial].

**Data Structures:**
The input is a hierarchical grid structure where cells are defined by their position in a tree. The output retains the AMR hierarchy but projected onto 2D planes [S1, unofficial]. Specific bit-packing or field layouts for the internal hypertree nodes are not detailed in the sources [G].

**Steps:**
1.  Define two axis-aligned cut planes within the coordinate system of the input grid [S1, unofficial].
2.  Apply these planes to the input hypertree grid to extract the intersecting regions [S1, unofficial].
3.  Generate 2D AMR outputs corresponding to the cut sections [S1, unofficial].
4.  (Optional) Apply a shrink filter to the output for visualization legibility [S1, unofficial].

**Parameters:**
*   **Cut Planes:** Defined by their axis alignment (x, y, or z) and position. The sources mention "two axis-aligned cut planes" but do not specify if they are parallel or orthogonal, or how their positions are parameterized [S1, unofficial].
*   **Shrink Factor:** A parameter for the visualization legibility step, not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The output is not a conforming mesh, which limits its use for post-processing requiring perfect connectivity [S1, unofficial].
*   Precision issues related to the intersection of planes with the hierarchical grid boundaries are not discussed [S1, unofficial].

## Subtopics

### PlaneCutter
PlaneCutter is a filter that cuts a hypertree grid using planes, producing polygonal data sets [S1, unofficial]. It operates in primal and dual modes [S1, unofficial]. It is chosen when conforming mesh output suitable for subsequent post-processing requiring perfect connectivity is needed [S1, unofficial]. It is considerably slower than AxisCut [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
AxisCut is faster than PlaneCutter [S1, unofficial]. Exact complexity metrics are not given in the sources [S1, unofficial].

**Measured Performance:**
No specific performance benchmarks are provided in the sources [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Fast execution; produces 2D AMR outputs suitable for visualization [S1, unofficial].
*   **Limitations:** Does not produce conforming mesh output, making it unsuitable for post-processing requiring perfect connectivity [S1, unofficial].

**When to Use:**
Use AxisCut for visualizing cross-sections of hypertree grids where speed is prioritized over mesh conformity [S1, unofficial]. Use PlaneCutter when conforming mesh output is required for further processing [S1, unofficial].

## Background
AxisCut is a native filter for visualizing tree-based AMR simulations with rectilinear geometry [S1, unofficial]. It is part of a suite of filters including Geometry, AxisReflection, PlaneCutter, and AxisClip [S1, unofficial]. The method is described in the context of efficient visualization and analysis of large-scale simulations [S1, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
