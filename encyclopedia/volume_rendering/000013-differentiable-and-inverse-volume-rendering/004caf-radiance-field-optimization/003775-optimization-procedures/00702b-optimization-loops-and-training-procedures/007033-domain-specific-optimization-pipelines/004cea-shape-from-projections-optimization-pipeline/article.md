# Shape-from-Projections Optimization Pipeline

Also called: run/ours.py main loop

## Overview
An iterative optimization pipeline that initializes a 3D mesh, refines its geometry through subdivision and cleaning, and performs differentiable rendering optimization to align the mesh with projection data [S1, unofficial]. It is distinguished from general radiance-field optimization by its explicit mesh-based representation and the use of external mesh-cleaning tools to maintain manifold validity during refinement [S1, unofficial].

## How it works
**Inputs and Outputs:**
* **Inputs:** Projection dataset, physical dimensions, initial mesh, and hyperparameters (`args.niter0`, `args.niter`, `args.nmu0`, `args.wlap`, `args.wflat`, `args.subdiv`) [S1, unofficial].
* **Outputs:** Optimized mesh (vertices, faces, labels) and attenuation values, saved to HDF5 [S1, unofficial].

**Data Structures:**
* `Model`: Encapsulates the differentiable renderer and mesh representation [S1, unofficial].
* `mus`: Array of initial attenuation values for the mesh materials [S1, unofficial].
* `args`: Configuration object containing optimization parameters [S1, unofficial].

**Numbered Steps:**
1. **Initialization:** Load the dataset and compute physical dimensions. Initialize the coarse mesh using `save_init_mesh` [S1, unofficial].
2. **Coarse Refinement Loop:** If `args.niter0 > 0`:
   a. Create a `Model` instance [S1, unofficial].
   b. Estimate and set initial attenuation values `mu` using `args.nmu0` [S1, unofficial].
   c. Run the optimization routine `optimize.run` [S1, unofficial].
   d. Extract the optimized vertices, faces, and material labels from the model [S1, unofficial].
   e. Update the `mus` values based on the refined geometry [S1, unofficial].
   f. Save the subdivided mesh [S1, unofficial].
   g. Clean the mesh using `pymeshfix` (with `max_iters=10, inner_loops=3`) and the `manifold` tool to ensure manifold validity [S1, unofficial]. Note: `manifold` requires the `outresolution` from the first refinement pass [S1, unofficial].
3. **Final Optimization:** Run the final optimization phase with `args.niter` iterations using the cleaned mesh from the coarse stage [S1, unofficial].
4. **Saving:** Save the final results (mesh, labels, parameters) to an HDF5 file [S1, unofficial].

**Parameters:**
* `args.niter0`: Threshold controlling whether the coarse refinement loop runs [S1, unofficial].
* `args.niter`: Number of iterations for the final optimization phase [S1, unofficial].
* `args.nmu0`: Initial attenuation values [S1, unofficial].
* `args.wlap`: Laplacian regularization weight [S1, unofficial].
* `args.wflat`: Flatness regularization weight [S1, unofficial].
* `args.subdiv`: Subdivision level for mesh refinement [S1, unofficial].

**Pitfalls:**
* Mesh cleaning relies on external executables (`pymeshfix`, `manifold`) [S1, unofficial].
* `manifold` requires the `outresolution` parameter from the first refinement pass [S1, unofficial].
* `pymeshfix` cleans with `max_iters=10, inner_loops=3` [S1, unofficial].

## Subtopics

### [Differentiable Object Pose Optimization](17340-differentiable-object-pose-optimization.md)
Refines the 6D pose (position and orientation) and scale of segmented 3D objects within a scene by minimizing the discrepancy between rendered and observed image data. It is distinguished from general radiance-field optimization by focusing specifically on rigid body transformations of pre-segmented objects rather than optimizing continuous volumetric fields. It is chosen when coarse pose initialization must be solved by optimizing object parameters in image space. [Differentiable Object Pose Optimization](17340-differentiable-object-pose-optimization.md)

## Operating characteristics
* **Time Complexity:** Not given in the sources.
* **Memory Complexity:** Not given in the sources.
* **Strengths:** Explicit mesh representation allows for downstream processing. Mesh cleaning ensures topological validity [S1, unofficial].
* **Limitations:** Relies on external tools for mesh cleaning, which can be a bottleneck or source of failure [S1, unofficial].
* **When to use:** Not given in the sources.
* **When not to use:** Not given in the sources.
* **Measured Performance:** With QEM initialization, training takes an average of 8 minutes and 40 seconds, achieving comparable reconstruction fidelity with a drastically reduced triangle count and significantly less time than retraining MobileNeRF (which takes over 6 hours) [S2, unofficial].

## Background
The pipeline originates from the "ShapeFromProjections" project by Jakeoung [S1, unofficial]. It is designed for reconstructing 3D shapes from 2D projections [S1, unofficial]. The method integrates differentiable rendering with traditional mesh processing techniques to refine geometry iteratively [S1, unofficial].

## Sources
[S1] jakeoung/ShapeFromProjections, https://github.com/jakeoung/ShapeFromProjections, fetched 2026-10-26 (unofficial)
[S2] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
