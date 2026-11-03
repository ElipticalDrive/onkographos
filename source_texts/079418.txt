# PlatonicSolid

[![Julia version](https://img.shields.io/badge/Julia-1.8-informational?logo=julia&logoColor=white&style=flat)](https://julialang.org/)
[![MIT license](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/subinbg/PlatonicSolid.jl/blob/main/LICENSE)
[![Build Status](https://github.com/subinbg/PlatonicSolid.jl/actions/workflows/CI.yml/badge.svg?branch=main)](https://github.com/subinbg/PlatonicSolid.jl/actions/workflows/CI.yml?query=branch%3Amain)
[![Coverage](https://codecov.io/gh/subinbg/PlatonicSolid.jl/branch/main/graph/badge.svg)](https://codecov.io/gh/subinbg/PlatonicSolid.jl)

Voxelization of the platonic solids: tetrahedron (`:tera`), cube (`:cube`), octahedron (`:octa`), dodecahedron (`:dodeca`), and icosahedron (`:icosa`). Specifically, this package produces a three-dimensional array whose voxel value is 1 if a voxel is inside a polyhedron and 0 otherwise. To generate a polyhedron,

```julia
using PlatonicSolid

# Number of pixels should be same in all dimensions
polyhedron = Array{Float32}(undef, 128,128,128)
# The radius of the circumscribed sphere with respect to the size of the array
radius = 0.5
# Rotation axis, if you want to rotate polyhedron
axis = [rand() rand() rand();]
# Rotation angle
angle = [rand()*π/2]
# Translation as [-0.5, 0.5]^3
translation = [0.0,0.0,0.0]
generate!(:tetra, polyhedron, radius, translation=translation, axis=axis, angle=angle)
```
