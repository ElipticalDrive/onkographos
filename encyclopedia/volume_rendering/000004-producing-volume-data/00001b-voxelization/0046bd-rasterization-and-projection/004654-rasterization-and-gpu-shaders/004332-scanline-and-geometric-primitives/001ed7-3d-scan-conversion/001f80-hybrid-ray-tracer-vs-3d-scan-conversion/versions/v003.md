# Hybrid Ray Tracer vs 3D Scan-Conversion
## Overview
This family compares two approaches for rendering mixtures of geometric primitives and volumetric data: hybrid ray tracing and 3D scan-conversion [S1, unofficial]. Both methods are reasonably efficient and produce aliasing-free images, but differ in cost, image quality, and versatility [S1, unofficial].
## Shared principles
Both methods integrate explicit geometric primitives with volumetric datasets to produce a single coherent image without aliasing artifacts [S1, unofficial]. They are both considered reasonably efficient for their respective use cases [S1, unofficial]. Builds on [3D Scan-conversion](1895-3d-scan-conversion.md).
## Subtopics

### Hybrid Ray Tracing
Renders mixtures of geometric and volumetric data [S1, unofficial]. Reasonably efficient and aliasing-free [S1, unofficial]. Specific implementation details, computational cost relative to 3D scan-conversion, and versatility characteristics are not given in the sources [S1, unofficial].

### 3D Scan-Conversion
Renders mixtures by embedding geometric primitives into the volumetric dataset [S1, unofficial]. Polygons exhibit finite thickness when rotated, and their opacity varies with their angle relative to the view direction, enhancing shape and orientation comprehension [S1, unofficial]. Embedding polygons improves comprehension of the volume, as a backdrop of known shape and pattern aids appreciation of fine internal structures during rotation [S1, unofficial]. For example, bisecting a CT study of the head with a gridded, coronally-oriented polygon improves appreciation of fine structure in the sinuses and eye orbits [S1, unofficial]. The implementation uses tri-linear interpolation for resampling 3D data, which can blur polygonal edges, and the shadow casting algorithm includes three successive resampling steps [S1, unofficial].

## Background
The comparison was described in a DTIC report [S1, unofficial]. The work highlights trade-offs in cost, image quality, and versatility between the two methods [S1, unofficial]. It suggests improvements such as better filtering to reduce blurring during 3D scan-conversion and reorganizing shadow casting operations to eliminate one resampling step [S1, unofficial].
## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
