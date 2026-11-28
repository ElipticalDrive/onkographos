# CUDA SDF Voxel Visualization

This project implements a CUDA/C++ visualization pipeline for sampling a signed distance function (SDF) on a 3D voxel grid and projecting the result into a 2D image.

The goal is to demonstrate basic GPU programming concepts such as CUDA kernels, 3D-to-1D memory mapping, block/grid configuration, and parallel processing of volumetric data.

## Overview

The program samples a torus-shaped signed distance function on a regular 3D grid. Each voxel is processed in parallel on the GPU. Voxels close to the surface are colored according to the local gradient direction, then the 3D volume is projected into a 2D image.

The project includes:

- a CUDA implementation in `src/visu.cu`
- a Jupyter notebook version for Google Colab
- example output images
- compilation and execution instructions


## Technical details

The 3D volume is stored as a contiguous 1D array using the following indexing scheme:

```cpp
idx(x, y, z) = x + size * (y + size * z)
