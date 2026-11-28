# Brush-based Interaction
Also called: Brushing

## Overview
Brush-based interaction allows users to select, modify, or inspect subsets of volumetric or splat-based data by defining a spatial region (the "brush") and applying operations to elements within it. It solves the problem of intuitively targeting specific data structures without requiring precise geometric modeling or manual indexing. Unlike [Ray-based Picking](19665-ray-based-picking.md) which targets single objects or points, this family supports continuous, multi-point selection and modification. It differs from [Brushing and Linking with Hilbert Codes](10912-brushing-and-linking-with-hilbert-codes.md) by focusing on spatial manipulation of the volume/splats themselves rather than encoding selection states for linked views, and from [Object Manipulation](22032-object-manipulation.md) by operating on raw data samples rather than abstracted scene objects.

## Shared principles
The core mechanism involves a user-defined region $B$ (the brush) and a set of data elements $P$ (voxels, samples, or splats). An element $p \in P$ is considered "selected" if it satisfies a spatial predicate relative to the brush center $c$ and radius $r$.

1.  **Brush Definition**: The brush is typically defined by a center $c$ (in screen or world coordinates) and a radius $r$.
2.  **Selection Predicate**: For a point $p$, the selection status $S(p)$ is determined by:
    $$ S(p) = \begin{cases} \text{true} & \text{if } \|p - c\| \le r \\ \text{false} & \text{otherwise} \end{cases} $$
    In some variants, this is extended to smooth degrees of interest (DOI) where $S(p) \in [0, 1]$.
3.  **Operation Application**: Once selected, elements undergo an operation $Op$ (e.g., lock, delete, highlight). The state of the dataset is updated based on $Op(S(p))$.
4.  **Feedback**: The visual representation is updated to reflect the new state, either immediately (direct) or after a computation step (indirect).

## Subtopics

### [Lock Tool](13353-lock-tool.md)
The Lock Tool prevents specific data points from being modified by other brushes or tools. It maintains a set $L$ of locked indices. Users add to $L$ via `Shift`+Click and remove via `Control`+Click. Rendering visibility is masked such that only points in $L$ are drawn. This is useful for protecting regions during exploration. See [Lock Tool](13353-lock-tool.md) for implementation details.

### [Interactive Feature Specification via Brushing](19538-interactive-feature-specification-via-brushing.md)
This technique defines complex data features by selecting subsets across multiple linked visualizations (e.g., scatterplots). It uses composite non-binary brushes to assign a continuous Degree of Interest (DOI) to every sample point, enabling the identification of overlapping high-dimensional structures. Unlike binary selection, it outputs a Feature Definition Language (FDL) structure. See [Interactive Feature Specification via Brushing](19538-interactive-feature-specification-via-brushing.md) for implementation details.

### Direct Selection and Deletion
An approach where users instantly highlight all splats that intersect with the brush during selection, contrasting with indirect approaches that compute intersections after brushing [S1].
- Selection/deletion brushes may be either circular (screen space) or spherical (world space) [S1].
- Users may choose selections based on splat centers or their border [S1].
- In case of circular brushes and splat-border intersection tests, an approximation checks whether the center of the brush circle is inside an ellipse that is enlarged by the radius of the brush [S1].
- To remove deleted splats, all splats are sorted by their deletion flag, putting deleted splats towards the end of the buffer [S1].
- Physical allocations at the buffer's end are unmapped and released in steps of the GPU's granularity [S1].
- Using radix sort is not required; developers may implement this functionality with GPU-friendly stream compaction [S1].

## Background
The concept of "brushing" originated in visual analytics for selecting data points in 2D plots. In volume rendering and splat-based visualization, it was adapted to handle large datasets efficiently. The specific variants described here, such as the Lock Tool and Interactive Feature Specification, are detailed in recent literature on Gaussian Splat editing and high-dimensional data exploration [S1]. The direct selection method for splats addresses performance limitations in WebGL and VR environments where immediate feedback is critical [S1].

## Sources
[S1] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
