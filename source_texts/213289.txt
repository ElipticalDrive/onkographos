# GeometrySDF.jl

*Simple signed distance field (SDF) generation from 3D triangle meshes and 2D polylines in Julia*

[![CI](https://github.com/KeitaNakamura/GeometrySDF.jl/actions/workflows/CI.yml/badge.svg)](https://github.com/KeitaNakamura/GeometrySDF.jl/actions/workflows/CI.yml)
[![codecov](https://codecov.io/gh/KeitaNakamura/GeometrySDF.jl/graph/badge.svg?token=hKUB58zHkF)](https://codecov.io/gh/KeitaNakamura/GeometrySDF.jl)
[![Stable](https://img.shields.io/badge/docs-latest%20release-blue.svg)](https://KeitaNakamura.github.io/GeometrySDF.jl/dev)

## Quick start

```julia
julia> using GeometrySDF

julia> sdf3d = GeometrySDF.sdf("test/stanford_bunny.stl");

julia> GeometrySDF.writevtk("bunny", sdf3d)
1-element Vector{String}:
 "bunny.vti"

julia> sdf2d = GeometrySDF.sdf([0.0 1.0; 0.2886751346 0.5; 0.8660254038 0.5; 0.5773502692 0.0; 0.8660254038 -0.5; 0.2886751346 -0.5; 0.0 -1.0; -0.2886751346 -0.5; -0.8660254038 -0.5; -0.5773502692 0.0; -0.8660254038 0.5; -0.2886751346 0.5]');

julia> GeometrySDF.writevtk("hexagram", sdf2d)
1-element Vector{String}:
 "hexagram.vti"
```

## Performance

Benchmarks were run on a MacBook Air (M4) using 4 threads. Timings report end-to-end SDF generation (including BVH build) for the same input mesh, with grid resolution controlled by `minres`. The STL files used below are included under `test/` (e.g. `test/stanford_bunny.stl`).

| Model  | \#Triangles | `minres` | \#Grid nodes | Time       | Throughput     |
|--------|------------:|---------:|-------------:|-----------:|---------------:|
| Bunny  | 112,402     | 128      | 1,822,210    | 2 s        | 0.91 M nodes/s |
| Bunny  | 112,402     | 256      | 14,099,442   | 14 s       | 1.01 M nodes/s |
| Dragon | 318,460     | 512      | 40,799,845   | 43 s       | 0.95 M nodes/s |
| Dragon | 318,460     | 1024     | 321,908,015  | 4 min 40 s | 1.15 M nodes/s |

## Gallery

<p align="center">
    <img width="180" src="https://github.com/user-attachments/assets/706348b6-c149-4688-9894-84d949d82f4b" />
    <img width="210" src="https://github.com/user-attachments/assets/9408b4e9-c812-486d-bece-8c8f8c33af8e" />
    <img width="170" src="https://github.com/user-attachments/assets/806915fd-a750-4eac-8946-3ba778258e5e" />
    <img width="150" src="https://github.com/user-attachments/assets/71a89afe-54d7-4732-ada0-d744b496a5d0" />
</p>
