# Ray Functions

Also called: Alpha Compositing Ray Function, Alternative compositing methods, Max/Avg/Distance compositing

## Overview
Ray functions determine the final color and opacity contribution of a ray passing through a volume by aggregating sample values along the ray path [S1, unofficial][S2, book]. While standard compositing accumulates opacity and color, alternative ray functions such as Maximum Intensity Projection (MIP), average value, and distance-to-threshold provide different visualizations where the result is a scalar value or distance rather than an accumulated color [S2, book]. The alpha compositing ray function is distinct because its result is not a scalar value or distance that can be represented on the ray profile, but rather an accumulated color and alpha weight [S2, book].

## Shared principles
All ray functions operate on a sequence of samples taken along a ray intersecting the volume. Each sample $i$ at position $t_i$ provides a scalar value $v_i$ (e.g., density or intensity) and potentially a color $c_i$. The functions aggregate these values to produce a single output per ray.

1.  **Maximum Value (MIP):** Returns the maximum scalar value encountered along the ray. It is forgiving of noise and provides intuitive structure but loses depth information [S2, book].
2.  **Average Value:** Returns the mean of scalar values along the ray.
3.  **Distance to Value:** Returns the distance along the ray at which a scalar value first meets or exceeds a specified threshold (e.g., 30) [S2, book].
4.  **Alpha Compositing:** Treats sample values as opacity accumulated per unit distance. It computes the final color and alpha using the standard volume rendering integral approximation [S2, book].

Unlike the first three functions, the alpha compositing result is not a simple scalar or distance [S2, book].

## Subtopics

### [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md)
Uses sigma (opacity) for compositing.

### [Training Ray Compositing](3785-training-ray-compositing.md)
General compositing framework.

## Background
Ray functions are fundamental techniques in image-order volume rendering, specifically within direct volume ray casting [S1, unofficial]. They serve as alternatives to standard compositing methods for determining color along a ray [S1, unofficial]. Maximum Intensity Projection (MIP) is cited as probably the simplest way to visualize volumetric data [S2, book].

## Sources
[S1] PPT - Volume Rendering, Part 2 PowerPoint Presentation, free download - ID:2205448, https://slideserve.com/walter/volume-rendering-part-2-powerpoint-ppt-presentation, fetched 2026-10-24 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
