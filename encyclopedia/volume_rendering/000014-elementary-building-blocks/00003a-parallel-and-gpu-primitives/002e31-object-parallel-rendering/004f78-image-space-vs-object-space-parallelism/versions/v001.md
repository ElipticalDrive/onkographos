# Image-Space vs Object-Space Parallelism

Also called: Forward vs Reverse Mapping Parallelism, Object-Space vs Image-Space Parallelism

## Overview
This family classifies parallel volume rendering strategies by the granularity of the primary computational unit: data elements (macros/voxels) or screen elements (pixels/rays). It distinguishes between **Object-Space Parallelism**, which processes data regardless of its projection, and **Image-Space Parallelism**, which processes pixels by mapping them back into the data. This distinction determines the data distribution strategy and the necessity of a redistribution phase.

## Shared principles
The core distinction lies in the direction of the mapping between the data space and the image space, which dictates the task granularity and data locality requirements.

### Object-Space Parallelism (Forward Mapping)
In object-space parallelism, tasks are performed per data element (e.g., per macro or voxel) at each processing node. This approach uses **forward mapping**, where the data is projected toward the screen [S1, unofficial]. The computational load is determined by the volume of data, not the resolution of the output image. Data is typically distributed statically or dynamically among nodes based on spatial partitions (macros) [S1, unofficial].

### Image-Space Parallelism (Reverse Mapping)
In image-space parallelism, tasks are performed per screen element (pixel or ray). This approach uses **reverse mapping**, where a pixel is mapped back into the data volume to determine its color and opacity [S1, unofficial]. The computational load is determined by the number of pixels and the number of samples per ray. This method often requires global knowledge of the data distribution or significant communication to fetch necessary voxels for each ray.

### The Redistribution Boundary
A critical structural element in hybrid or transition schemes is the **redistribution task**. This task generally performs an **image-space sort** of the data [S1, unofficial]. It usually forms the boundary between object-space and image-space tasks. Before image-space processing can begin efficiently, data must often be redistributed so that the voxels required by a specific set of pixels (or a specific node's pixel workload) are locally available. This step resolves the mismatch between the initial data distribution (often object-space oriented) and the access pattern of image-space rays [S1, unofficial].

## Subtopics

### [Object Parallel Rendering](5728-object-parallel-rendering.md)
The parent scheme that exploits parallelism in object space by dividing the volume into subvolumes.

### [Image-Space Parallel Rendering](5729-image-space-parallel-rendering.md)
Strategies where parallelism is derived from the image plane, often requiring ray casting per pixel.

### [Object-Space Parallel Rendering](5730-object-space-parallel-rendering.md)
Strategies where parallelism is derived from the data volume, often using forward projection or splatting.

## Background
The terminology and conceptual framework for distinguishing object-space and image-space parallelism in the context of multicomputer volume rendering were established in the early 1990s. The specific definitions of forward mapping (data to screen) and reverse mapping (pixel to data) as the basis for task organization are attributed to Westover (1990) and elaborated upon in the taxonomy of algorithms for multicomputers [S1, unofficial]. The concept highlights that while data is naturally organized in object space, the final output is an image, necessitating a transformation or redistribution step to bridge the two parallelism domains.

## Sources
[S1] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
