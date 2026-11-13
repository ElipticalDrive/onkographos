# Barycentric Shape Selection Widget

Also called: Barycentric Shape Selection, Shape-Selection Widget

## Overview
The Barycentric Shape Selection Widget is a user interface component that maps 4D transfer function space (tubiness, surfaceness, blobbiness, and intensity) onto a 2D equilateral triangle using barycentric coordinates, allowing users to intuitively select shape classes for volume rendering [S1]. It distinguishes itself from scalar-only transfer functions by enabling direct manipulation of geometric shape memberships (tubiness, surfaceness, blobbiness) derived from curve skeletons, serving as the first stage of classification before intensity-based optical property assignment [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of computed shape segments from a pre-segmented volume. Each segment has associated normalized shape classifier values: tubiness ($T$), surfaceness ($S$), and blobbiness ($B$).
*   **Output:** A set of selected markers within the widget, which constrain the visualization to specific shape classes.

**Data Structures:**
The widget uses an equilateral triangle geometry. The three corners represent the three shape classes:
1.  Corner 1: Tubiness
2.  Corner 2: Surfaceness
3.  Corner 3: Blobbiness

Each marker is defined by its position $(x, y)$ within the triangle, derived from the barycentric coordinates of the segment's shape descriptors.

**Steps:**
1.  **Normalization:** Ensure the shape classifiers (tubiness, surfaceness, blobbiness) are normalized to be comparable [S1]. Let the normalized values be $t, s, b$ such that they represent the likelihood of the segment belonging to each class.
2.  **Marker Placement:** For each computed shape segment, place a marker inside the triangle. The position is determined by barycentric coordinates defined by the three normalized shape classifiers [S1].
    *   The marker is placed closer to the corners representing the shape classes the identified segment is best classified as [S1].
    *   Mathematically, if the triangle vertices are $V_T, V_S, V_B$, the marker position $P$ is:
        $$ P = t \cdot V_T + s \cdot V_S + b \cdot V_B $$
        where $t + s + b = 1$ (assuming normalization ensures this sum or is scaled accordingly) [S1].
3.  **User Selection (First Stage):** The user selects an arbitrary number of markers to constrain the shape classes to be visualized [S1]. This selection filters the volume data to only include voxels associated with the selected shape segments.
4.  **Intensity Mapping (Second Stage):** Use a conventional 1D transfer function to assign optical properties (color, opacity) to the selected shape classes based on intensity values [S1].

**Parameters:**
*   **Tubiness Threshold:** Default merging parameter of 0.8 [S1].
*   **Maximum Inner-Surface-to-Outer-Surface Ratio:** Default merging parameter of 0.1 [S1].
*   **Maximum Skeleton:** Parameter adjusted by user, but default value not specified [S1].

**Edge Cases and Pitfalls:**
*   If the data set has no planar structures, the surfaceness class can be omitted, collapsing the triangle to a line [S1].
*   Background voxels are ignored; their descriptor set is all zero [Parent].

## Subtopics

### Shape Attribute
An operator prepared on segmented volumes that examines three aspects to discriminate an object: tubiness (cylindrical aspect), surfaceness (planar aspect), and blobbiness (bubble aspect) [S2, unofficial]. It serves as the underlying classification mechanism that provides the input values for the Barycentric Shape Selection Widget.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Tested with synthetic and real-world data sets, mostly from the medical domain, integrated into the Voreen volume rendering engine [S1].
*   **Strengths:** Intuitive selection of shape classes; normalization allows comparison of shape classifiers [S1].
*   **Limitations:** Requires pre-segmentation; background voxels are ignored [Parent].
*   **When to use:** When visualizing data with distinct geometric shapes (tubes, surfaces, blobs) and the user wants to select specific shape classes.
*   **When not to use:** When the data lacks planar structures (surfaceness class may be omitted) or when intensity-based classification is sufficient [S1].

## Background
The method was proposed by Praßni et al. (2010) and integrated into the Voreen volume rendering engine [S1][S2, unofficial]. It is similar to the barycentric coordinate approach proposed by Kindlmann and Weinstein for DTI data [S1]. The technique is part of the broader field of shape-based transfer functions for volume visualization [S1].

## Sources
[S1] Shape-based Transfer Functions for Volume Visualization, https://www.diva-portal.org/smash/get/diva2:623421/FULLTEXT01.pdf, fetched 2026-10-29
[S2] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
