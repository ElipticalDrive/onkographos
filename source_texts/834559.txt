Messing around with conservative rasterization.

Options:
- Hardware conservative raster
  - Pro: Perfect and fast.
  - Con: Need HW support.
- Expand each triangle in vertex shader
  - Pro: Pretty fast, use HW rasterization pipeline.
  - Con: Need to prepare the mesh (unique verts for each tri).
- Regular rasterization, then dilate in a compute shader (TODO)
  - Pro: Simple to implement, fast
  - Con: Worst results, furthest from HW rasterization.
- Software rasterizer in compute shader (TODO)
  - Pro: Results can be very good. Doesn't need prepared mesh.
  - Con: Most complex to implement. Hard to make fast for large triangles due to lack of parallelism within a triangle.
