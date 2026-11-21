# Shape-from-Projections Optimization Pipeline

Also called: run/ours.py main loop

## Overview
An iterative optimization pipeline that initializes a 3D mesh, refines its geometry through subdivision and cleaning, and performs differentiable rendering optimization to align the mesh with projection data [S1, unofficial]. It is distinguished from general radiance-field optimization by its explicit mesh-based representation and the use of external mesh-cleaning tools to maintain manifold validity during refinement [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Projection dataset (images/angles), physical dimensions, initial mesh (or point cloud), and hyperparameters (`niter0`, `niter`, `nmu0`, `wlap`, `wflat`, `subdiv`).
*   **Outputs:** Optimized mesh (vertices, faces, labels) and attenuation values, saved to HDF5.

**Data Structures:**
*   `Model`: Encapsulates the differentiable renderer and mesh representation.
*   `mus`: Array of initial attenuation values for the mesh materials.
*   `args`: Configuration object containing optimization parameters.

**Numbered Steps:**
1.  **Initialization:** Load the dataset and compute physical dimensions. Initialize the coarse mesh using `save_init_mesh` [S1, unofficial].
2.  **Coarse Refinement Loop:** If `args.niter0 > 0`:
    a. Create a `Model` instance.
    b. Estimate and set initial attenuation values `mu` using `args.nmu0` [S1, unofficial].
    c. Run the optimization routine `optimize.run` for `args.niter0` iterations [S1, unofficial].
    d. Extract the optimized vertices, faces, and material labels from the model [S1, unofficial].
    e. Update the `mus` values based on the refined geometry [S1, unofficial].
    f. Save the subdivided mesh.
    g. Clean the mesh using `pymeshfix` (with `max_iters=10, inner_loops=3`) and the `manifold` tool to ensure manifold validity [S1, unofficial]. Note: `manifold` requires the `outresolution` from the first refinement pass [S1, unofficial].
3.  **Final Optimization:** Run the final optimization phase with `args.niter` iterations using the cleaned mesh from the coarse stage [S1, unofficial].
4.  **Saving:** Save the final results (mesh, labels, parameters) to an HDF5 file [S1, unofficial].

**Parameters:**
*   `args.niter0`: Number of coarse refinement iterations [S1, unofficial].
*   `args.niter`: Number of final optimization iterations [S1, unofficial].
*   `args.nmu0`: Initial attenuation values [S1, unofficial].
*   `args.wlap`: Laplacian regularization weight [S1, unofficial].
*   `args.wflat`: Flatness regularization weight [S1, unofficial].
*   `args.subdiv`: Subdivision level for mesh refinement [S1, unofficial].

**Pitfalls:**
*   Mesh cleaning relies on external executables (`pymeshfix`, `manifold`). Failure to install these or incorrect argument passing (e.g., missing `outresolution` for `manifold`) will break the pipeline [S1, unofficial].
*   Precision issues may arise if the initial `mu` values are poorly estimated, leading to convergence failures in the coarse stage [S1, unofficial].

## Subtopics

### Differentiable Object Pose Optimization
A variant used in room reconstruction where object poses are jointly optimized using differentiable rendering [S2]. It optimizes scaling, horizontal translation, and rotation around the vertical axis (preserving floor/ceiling alignment) for each object [S2]. For wall-mounted objects, optimization is restricted to transformations along the wall plane [S2]. The loss function combines L1 differences of rendered vs. target masks weighted by depth, and L1 differences of Gaussian-blurred masks:
$$
L_{obj} = \sum_i L1(M_{ir}(\theta_{obj}) * D_r(\theta_{obj}), M_{it} * D_i) + \sum_i L1(G(M_{ir}(\theta_{obj})), G(M_{it}))
$$
where $\theta_{obj}$ are pose parameters, $M_{ir}/M_{it}$ are rendered/target masks, $D_r/D_t$ are rendered/target depths (normalized to unit range), and $G$ is a 2D Gaussian convolution pyramid [S2]. Anti-aliasing in the differentiable renderer is critical to ensure non-zero gradients for mask comparison losses [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by the differentiable rendering passes in each iteration. The coarse stage adds overhead for subdivision and mesh cleaning.
*   **Memory Complexity:** Depends on the mesh size and the number of samples per ray in the differentiable renderer.
*   **Strengths:** Explicit mesh representation allows for easy downstream processing (e.g., texturing, collision detection). Mesh cleaning ensures topological validity.
*   **Limitations:** Relies on external tools for mesh cleaning, which can be a bottleneck or source of failure. Coarse-to-fine strategy requires careful tuning of `niter0` and `niter`.
*   **When to use:** When an explicit mesh is required and the input data consists of multiple projections (e.g., CT scans, multi-view stereo).
*   **When not to use:** When a neural field representation is sufficient or when real-time performance is critical and external tool dependencies are unacceptable.

## Background
The pipeline originates from the "ShapeFromProjections" project by Jakeoung [S1, unofficial]. It is designed for reconstructing 3D shapes from 2D projections, often in the context of medical imaging or industrial CT. The method integrates differentiable rendering with traditional mesh processing techniques to refine geometry iteratively.

## Sources
[S1] jakeoung/ShapeFromProjections, https://github.com/jakeoung/ShapeFromProjections, fetched 2026-10-26 (unofficial)
[S2] PSDR-Room: Single Photo to Scene using Differentiable Rendering, https://doi.org/10.1145/3610548.3618165, fetched 2026-11-08
