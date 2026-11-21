# Image-Space vs Object-Space Parallelism

Also called: Forward vs Reverse Mapping Parallelism, Object-Space vs Image-Space Parallelism

## Overview
This family distinguishes between two task organization schemes for volume rendering on multicomputers [S1, unofficial]. It separates **Object-Space Parallelism**, which processes data regardless of its projection, from **Image-Space Parallelism**, which processes pixels by mapping them back into the data [S1, unofficial].

## Shared principles
The distinction is defined by the direction of the mapping between the data space and the image space [S1, unofficial]. Data distribution may be static or dynamic, and of any granularity [S1, unofficial].

### Object-Space Parallelism (Forward Mapping)
In object-space parallelism, tasks are performed per macro at each node, regardless of that data’s projection onto the viewing plane [S1, unofficial]. This is a forward mapping of the data toward the screen projection [S1, unofficial].

### Image-Space Parallelism (Reverse Mapping)
In image-space parallelism, tasks are performed per screen element (pixel or ray) [S1, unofficial]. This is a reverse mapping of a pixel into the data at a node [S1, unofficial].

### The Redistribution Boundary
The redistribution task generally performs an image-space sort of the data and usually forms the boundary between object-space and image-space tasks [S1, unofficial]. The data is already partially sorted due to its organization into macros and distribution among nodes [S1, unofficial].

## Subtopics

### [Object Parallel Rendering](5728-object-parallel-rendering.md)
The parent scheme that exploits parallelism in object space by dividing the volume into subvolumes.

### [Image-Space Parallel Rendering](5729-image-space-parallel-rendering.md)
Strategies where parallelism is derived from the image plane, often requiring ray casting per pixel.

### [Object-Space Parallel Rendering](5730-object-space-parallel-rendering.md)
Strategies where parallelism is derived from the data volume, often using forward projection or splatting.

## Background
The specific definitions of forward mapping (data to screen) and reverse mapping (pixel to data) as the basis for task organization are attributed to Westover (1990) and elaborated upon in the taxonomy of algorithms for multicomputers [S1, unofficial].

## Sources
[S1] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
