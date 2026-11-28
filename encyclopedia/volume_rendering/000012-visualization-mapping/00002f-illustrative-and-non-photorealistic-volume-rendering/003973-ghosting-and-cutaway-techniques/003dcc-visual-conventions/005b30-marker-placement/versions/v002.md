# Marker Placement
Also called: Cone Markers

## Overview
Marker Placement is an illustrative volume rendering technique that overlays cone-shaped geometric primitives into the 3D data space to aid feature location and facilitate discussion during parameter adjustments [S1, S2, unofficial]. It is used to help locate features again after changes of visualization parameters [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset and user interaction via a pointer or wand [S1, S2, unofficial].
*   **Output:** Cone-shaped markers placed in the dataset with updated positions and orientations [S1, S2, unofficial].

**Data Structures:**
Each marker stores its position and orientation, updated by the pointer/wand input [G].

**Steps:**
1.  **Placement:** The user places a cone-shaped marker in the dataset using a pointer or wand [S1, S2, unofficial].
2.  **Manipulation:** The user moves and orients the marker arbitrarily using the pointer/wand [S1, S2, unofficial].

**Formulas:** Not given in the sources.

**Parameters:** Not given in the sources.

**Edge Cases:** Not given in the sources.

## Subtopics

### Waypoint Mechanism
Allows users to store and automatically revisit locations of structures of interest without manual 6 DoF navigation [S3]. The user places markers on isosurface structures using a 3D pointer, the application stores the marker locations, and on request automatically navigates to the marked structure [S3]. 6 DoF transformation (rotate and pan) is used as the basic navigation mode before waypoint activation [S3].

## Operating characteristics

## Background
Described in systems for user-friendly volume data set exploration [S1, S2, unofficial]. It was designed to aid discussion and help find features again after changes of visualization parameters [S1, S2, unofficial].

## Sources
[S1] IEEE Visualization 2004 Contest: User Friendly Volume Data Set Exploration in the Cave, http://vis.computer.org/vis2004contest/brown/, fetched 2026-10-28 (unofficial)
[S2] IEEE Visualization 2004 Contest: User Friendly Volume Data Set Exploration in the Cave, http://web.eng.ucsd.edu/~jschulze/projects/vis04-contest/, fetched 2026-10-28 (unofficial)
[S3] Immersive Isosurface Visualisation for Engineering Datasets, https://doi.org/10.1109/bdva.2017.8114624, fetched 2026-10-26
