# VTK Image Interpolation

Also called: vtkImageInterpolator, vtkImageSincInterpolator

## Overview
VTK Image Interpolation is a family of classes in the Visualization Toolkit (VTK) library used to resample volume data into an isotropic grid by retrieving scalar values at arbitrary 3D positions [S1]. It distinguishes itself from sibling methods by providing specific implementations for standard polynomial interpolation (`vtkImageInterpolator`) and windowed sinc approximation (`vtkImageSincInterpolator`), allowing users to balance computational cost against spectral fidelity during volume resampling [S1].

## Shared principles
The core mechanism involves evaluating a continuous function $f(x, y, z)$ from discrete voxel data $V_{i,j,k}$ at arbitrary query coordinates $(x_q, y_q, z_q)$. Both classes in this family operate on the principle of kernel-based interpolation, where the value at the query point is a weighted sum of neighboring voxel values. The weights are determined by the interpolation kernel and the relative distance between the query point and the voxel centers.

For `vtkImageSincInterpolator`, the shared principle extends to spectral windowing. The ideal sinc kernel $\text{sinc}(r) = \frac{\sin(\pi r)}{\pi r}$ is truncated and multiplied by a window function $W(r)$ to limit the kernel size and reduce ringing artifacts [S1]. The resulting kernel is:
$$ K(r) = \text{sinc}(r) \cdot W(r) $$
where $r$ is the normalized distance from the query point. The window half-width (WHW) determines the support of the kernel, i.e., the number of neighboring voxels considered [S1].

## Subtopics

### vtkImageInterpolator
This is the default interpolator in VTK, supporting linear, cubic, and nearest-neighbor interpolation modes [S1]. It is chosen when computational efficiency is prioritized or when the data does not require high-frequency preservation. Linear interpolation provides a balance between speed and smoothness, while cubic offers higher accuracy at increased cost. Nearest-neighbor is used for categorical data or maximum speed [S1].

### vtkImageSincInterpolator
This class approximates sinc interpolation by multiplying the sinc function with a window function to limit kernel size [S1]. It is chosen when high-fidelity resampling is required, such as in medical imaging analysis where spectral content preservation is critical [S1]. It supports specific window functions: Lanczos, Kaiser (adjustment parameter off), Cosine, Hann, Hamming, Blackman, and Nuttall [S1]. In Fourier space, the resolution of the spectral window is reduced to the order of the half-width of the sinc function [S1]. Consequently, the quality of sinc interpolation depends on both the window function type and the window half-width (WHW), which ranges from 1 to 16 [S1]. Additionally, it supports blur factors (BF) in x, y, and z directions to blur data during interpolation, which automatically increases the kernel size and computation time [S1].

## Background
The VTK Image Interpolation classes are part of the Visualization Toolkit (VTK), a C++ software library for 3D computer graphics, image processing, and visualization [S1]. The specific classes `vtkImageInterpolator` and `vtkImageSincInterpolator` were utilized in research for optimizing breast tomosynthesis visualization through 3D volume rendering, where volume data was resampled into an isotropic grid [S1]. The implementation relies on internal VTK mechanisms for data interpolation [S1]. The study of these interpolators involved testing various window functions and parameters, including the window half-width (WHW) and blur factors, to determine optimal resampling strategies for medical imaging data [S1]. The default sampling distance used in these studies was 1.0 mm [S1].

## Sources
[S1] Optimization of Breast Tomosynthesis Visualization through 3D Volume Rendering, https://doi.org/10.3390/jimaging6070064, fetched 2026-10-26
