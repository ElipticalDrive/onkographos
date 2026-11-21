# SOAR-based Adaptive Refinement

Also called: Lindstrom and Pascucci's SOAR algorithm adaptation

## Overview
SOAR-based Adaptive Refinement is a crack-free mesh generation scheme for adaptive subdivision of tetrahedral meshes. It solves the problem of maintaining geometric consistency during incremental refinement by storing object-space error terms and bounding spheres for "diamonds" (pairs of cousin patches). It sets itself apart from static mesh representations by enforcing a strict nesting hierarchy where parent error/sphere values dominate children, and cousins agree on shared values, ensuring that refinement decisions are globally consistent without requiring full mesh reconstruction [S1].

## How it works
**Inputs:** A multiresolution volume dataset organized as a diamond hierarchy; a screenspace error tolerance threshold.
**Outputs:** A conforming tetrahedral mesh $\Sigma$ with adaptive refinement levels.

**Data Structures:**
The core unit is the **diamond**, defined as two cousin patches sharing a hypotenuse. Each diamond stores:
1.  **Object-space error term ($E$):** A scalar value representing the maximum deviation from the ideal surface.
2.  **Bounding sphere ($B$):** A sphere enclosing the diamond's geometry, used for culling and error estimation.

**Precomputation (Encoding):**
1.  For each patch in the hierarchy, compute the initial error $E$ and bounding sphere $B$.
2.  **Inflation for Nesting:** Iterate through the hierarchy from leaves to root. For every parent diamond, update its $E$ and $B$ to be the maximum of its own values and those of its children. This ensures $E_{parent} \ge E_{child}$ and $B_{parent} \supseteq B_{child}$ [S1].
3.  **Cousin Agreement:** For any pair of cousin diamonds, enforce a single shared $E$ and $B$. This value is typically the maximum of the two cousins' values to ensure conservative refinement. This nesting is enforced for cousins as well [S1].

**Refinement Steps:**
1.  Traverse the diamond hierarchy.
2.  For each diamond, compare its stored object-space error $E$ against the projected screenspace error tolerance.
3.  If $E$ exceeds the tolerance, mark the diamond for refinement.
4.  Due to the precomputed inflation and cousin agreement, refining a diamond automatically implies the necessary refinement of its ancestors and cousins to maintain a crack-free mesh [S1].

**Parameters:**
*   **Screenspace error tolerance:** Determines the threshold for refinement. Not given in the sources.
*   **Patch size parameter ($n$):** Each patch has $V = (n + 1)(n + 2)/2$ vertices and $T = n^2$ triangles. The terrain dimensions are limited to $(2mn + 1)^2$ for some non-negative integer $m$. Not given in the sources for typical $n$ values.

**Edge Cases and Pitfalls:**
*   **Precision:** Object-space error terms must be computed with sufficient precision to avoid false negatives in refinement decisions.
*   **Boundary Conditions:** The hierarchy structure imposes limits on terrain dimensions based on $n$ and $m$.

## Subtopics

### [Selective Refinement with Active Front](18965-selective-refinement-with-active-front.md)
Maintains a dynamic "cut" of the refinement DAG for incremental updates.

## Operating characteristics
*   **Time Complexity:** Precomputation of error/sphere inflation is linear in the number of diamonds. Refinement traversal is proportional to the number of diamonds exceeding the threshold.
*   **Memory Complexity:** Requires storing $E$ and $B$ for each diamond.
*   **Strengths:** Guarantees crack-free meshes; efficient incremental updates; precomputed data allows fast runtime decisions.
*   **Limitations:** Precomputation overhead; limited terrain dimensions based on patch size.
*   **When to use:** When crack-free adaptive refinement is required for multiresolution volume data.
*   **When not to use:** When dynamic mesh topology changes are frequent and precomputation is prohibitive.

## Background
The method is based on Lindstrom and Pascucci's SOAR algorithm [S1]. It is used in the context of on-the-fly decompression and rendering of multiresolution terrain.

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
