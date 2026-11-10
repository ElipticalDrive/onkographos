# Adaptive Sampling-based Isosurface Extraction

Also called: Octree Dual Contouring (OctDC), Octree Marching Cubes (OctMC)

## Overview
Adaptive Sampling-based Isosurface Extraction is a category of methods that construct a mesh by adaptively sampling the implicit field, as opposed to analytic extraction [S1]. These methods rely on spatial data structures, typically octrees, to refine resolution near the surface. They are distinguished from their siblings by their reliance on discrete sampling and topological reconstruction algorithms (like Marching Cubes or Dual Contouring) rather than direct analytic evaluation of the implicit function [S1].

## Shared principles
The core mechanism involves traversing an octree to identify regions where the implicit field $f(\mathbf{x}) = 0$ intersects the volume. Unlike analytic methods that evaluate the field continuously, these methods sample the field at specific points (vertices of the octree cells or dual grid nodes) to determine topology [S1]. The adaptive nature implies that the octree depth is determined by local error criteria or surface proximity, leading to variable resolution across the domain [S1].

## Subtopics

### [Octree Marching Cubes (OctMC)](1247-octree-based-isosurfacing.md)
Uses the Marching Cubes algorithm on an octree. It is simpler to implement but prone to cracks if the octree is not strictly adapted to the surface topology.

### [Octree Dual Contouring (OctDC)](1247-octree-based-isosurfacing.md)
Uses Dual Contouring on an octree. It produces smoother surfaces and handles topology changes better than OctMC but requires solving a least-squares system for vertex positions.

### [McGrids](1247-octree-based-isosurfacing.md)
A grid-based adaptive method that uses a different spatial discretization strategy than standard octrees, often trading off some adaptivity for simpler data structures.

## Background
This family of methods is widely used in computer graphics for extracting surfaces from volumetric data or neural implicit representations. The comparison data comes from the evaluation of **TetraSDF**, which contrasts these sampling-based approaches with its proposed analytic extraction method [S1].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
