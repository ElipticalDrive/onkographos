# Sphere Ray Casting Benchmark Methods
Also called: Spherical glyph ray casting

## Overview
This family characterizes scalability and performance for drawing spherical glyphs in 3D space using proxy geometry for pixel-perfect ray casting [S1]. It distinguishes itself from direct tessellation approaches by evaluating various proxy-based ray casting techniques (triangles, quads, geometry-shader-generated polygons) and adaptive vertex distribution methods [S1].

## Shared principles
All methods in this family rely on proxy geometry to approximate spherical particles, followed by a ray-sphere intersection test to determine pixel coverage [S1]. The core problem addressed is the efficient rendering of large numbers of spherical particles (e.g., in molecular dynamics simulations) while maintaining pixel-perfect accuracy [S1].

Key shared characteristics:
- **Proxy Geometry**: Spheres are not rendered as full meshes but as simple geometric primitives (triangles, quads, or polygons) that serve as a basis for ray casting [S1].
- **Ray Casting**: A ray-sphere intersection test is performed, typically in the fragment shader, to determine if a pixel lies within the sphere [S1].
- **Adaptive Detail**: Some techniques adjust the complexity of the proxy geometry based on the distance to the camera [S1].
- **Hardware Constraints**: Performance measurements must account for platform-specific limitations, such as the lack of tessellation shader support on certain consoles (e.g., Xbox UWP) [S1].

## Subtopics

### [Adaptive Sphere/Hemisphere Techniques](9513-adaptive-sphere-hemisphere-techniques.md)
Computes the number of vertices for each particle based on camera distance. The hemisphere variant distributes vertices on the facing side, while the sphere variant distributes them over the whole surface.

### [Ray Casting on Instanced Triangles](9514-ray-casting-instanced-triangles.md)
Uses instanced triangle geometry as the proxy for ray casting.

### [Ray Casting on Instanced Quads](9515-ray-casting-instanced-quads.md)
Uses instanced quad geometry as the proxy for ray casting.

### [Ray Casting on Geometry-Shader-Generated Quads/Polygons](9516-ray-casting-geometry-shader-generated.md)
Uses geometry shaders to generate quads or polygons on the fly for ray casting.

### [Tessellation-Based Sphere/Hemisphere Geometry](9517-tessellation-sphere-hemisphere.md)
Creates spherical geometry on the fly using tessellation shaders. Note: This method was only measured on desktop machines due to UWP Xbox limitations.

## Background
Originating from research on SciVis on gaming consoles, this benchmark family was introduced to characterize scalability for spherical glyph rendering [S1]. The methods build on earlier work by Gumhold (2003) regarding pixel-perfect sphere ray casting using proxy geometry [S1].

The benchmarks were tested using real-world molecular dynamics data sets (liquid layer formation, laser ablation) and uniformly distributed random particles in a $10^3$ unit domain [S1]. A significant historical constraint noted is the inability to run tessellation shaders on the UWP platform for Xbox, limiting the scope of tessellation-based measurements to desktop environments [S1].

## Sources
[S1] Your visualisations are going places: SciVis on gaming consoles, https://doi.org/10.1007/s12650-024-01035-4, fetched 2026-10-26
