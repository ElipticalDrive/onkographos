# Procedural Terrain Generator (C, Perlin Noise, OpenMP)

This project generates terrain heightmaps using 2D Perlin or simplex noise and parallel processing with OpenMP. It showcases:

- Two noise functions on the same permutation table — classic Perlin, and
  simplex, which skews onto a triangular lattice so it needs three corners
  instead of four and has no axis-aligned directional bias
- Parallelising mathematical operations on large arrays
- Row ownership: each thread writes whole rows, so two threads never share a
  cache line. That is the useful property, rather than the traversal order
  itself being clever
- File I/O optimisation via buffered binary writes
- Parallel efficiency comparison between **static** and **dynamic** OpenMP scheduling

## Build

Requires a C compiler with OpenMP support (e.g. `gcc`):

    make

This produces:

    ./terrain_generator

Usage:

    ./terrain_generator [width] [height] [threads] [scale] [octaves] [chunk_size]

Example:

    ./terrain_generator 4096 4096 8 8.0 6 64


Outputs:

terrain_static.pgm – generated with schedule(static, chunk_size)
terrain_dynamic.pgm – generated with schedule(dynamic, chunk_size)

The program prints timing information for both runs and reports the relative speedup.
You can open the .pgm files in most image viewers or import them into tools such as GIMP or MATLAB.

**Parallelisation details**

Heightmap stored as a 1D float array in row-major order (heightmap[y * width + x]).
Outer loop parallelises over rows y; inner loop iterates over x for good cache locality.
OpenMP uses schedule (runtime) in the loop; omp_set_schedule switches between static and dynamic scheduling at runtime.
Noise evaluation uses multi-octave fractional Brownian motion built on top of 2D Perlin noise.

**Build and test locally**

In the repo:
make
./terrain_generator 1024 1024 8 8.0 6 64 (system friendly example)

## Noise selection

```bash
./terrain_generator 2048 2048 8 8 6 64 --noise simplex
./terrain_generator 2048 2048 8 8 6 64 --noise perlin    # default
```

## Scope

2048x2048 floats is 16 MB and 4096x4096 is 64 MB — both fit in RAM comfortably
and finish well under a second. This is a parallelism and noise exercise, not an
out-of-core terrain system, and the static-vs-dynamic scheduling comparison is
an experiment rather than a general result: the work per row is uniform here, so
dynamic scheduling has nothing to rebalance and mostly measures its own
overhead.
