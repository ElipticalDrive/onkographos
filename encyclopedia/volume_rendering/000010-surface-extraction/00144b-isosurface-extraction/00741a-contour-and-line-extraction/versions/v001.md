# Contour and Line Extraction

Also called: Contouring, Line Extraction, Silhouette Extraction

## Overview
Contour and Line Extraction methods generate 1D geometric primitives (lines or curves) from volumetric data, rather than 2D surfaces. This family solves the problem of visualizing specific features such as crest lines, ridges, or view-dependent silhouettes, which are often obscured or irrelevant in dense surface meshes. Unlike [Isosurface extraction](30-isosurface-extraction.md) methods that produce watertight meshes covering the entire domain boundary, contour extraction produces sparse, output-sensitive geometry proportional to the length of the features, not the volume size.

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers [G].

The core mechanism involves defining two implicit scalar fields:
1.  The **isosurface field** $f(\mathbf{x})$, typically defined as $\phi(\mathbf{x}) - \tau = 0$, where $\phi$ is the input data and $\tau$ is a threshold .
2.  The **contour field** $c(\mathbf{x})$, which defines the specific feature to extract. For view-dependent silhouettes, this is defined by the condition $\mathbf{n} \cdot \mathbf{v} = 0$, where $\mathbf{n}$ is the surface normal and $\mathbf{v}$ is the view vector. Using $\mathbf{n} = -\nabla\phi$, the contour field becomes $c(\mathbf{x}) = -\nabla\phi(\mathbf{x}) \cdot \mathbf{v}$ .

The extraction target is the intersection curve where $f(\mathbf{x}) = 0$ and $c(\mathbf{x}) = 0$. Algorithms traverse the grid, identifying voxels where both fields change sign (zero-crossings) and computing the intersection points within those voxels using linear interpolation .

## Subtopics

### [Surface Tessellation from Iso-Contours](3696-surface-tessellation-from-iso-contours.md)
Reconstructs a 3D surface by triangulating "ribbons" between 2D contour slices. It differs by operating on pre-extracted 2D contours and establishing topological correspondence between slices, rather than tracing 3D volume intersections.

### [Marching Lines for Volumetric Contour Extraction](13112-marching-lines-for-volumetric-contour-extraction.md)
Extracts view-dependent silhouettes (contours) from a single scalar field by intersecting the isosurface with a contour surface defined by the view direction. It is output-sensitive and avoids full volume traversal.

### [Marching Lines](13118-marching-lines.md)
The volumetric counterpart to Marching Cubes, extracting the 1D intersection curves of two arbitrary implicit scalar fields. It produces closed loops or open segments representing crest lines or general surface intersections.

## Background
The concept of extracting contours from volumetric data has evolved alongside surface extraction. While Marching Cubes (1987) popularized surface meshing, the specific extraction of 1D features like ridges and silhouettes required distinct algorithms. The "Marching Lines" algorithm for two implicit functions is a direct generalization of Marching Squares to 3D, focusing on the intersection of zero-level sets. View-dependent contour extraction (silhouettes) leverages the gradient of the scalar field to define the contour surface dynamically based on the camera position . These methods are used in scientific visualization for highlighting features that are difficult to perceive on solid surfaces, such as the boundaries of regions in fluid dynamics or the edges of anatomical structures.

