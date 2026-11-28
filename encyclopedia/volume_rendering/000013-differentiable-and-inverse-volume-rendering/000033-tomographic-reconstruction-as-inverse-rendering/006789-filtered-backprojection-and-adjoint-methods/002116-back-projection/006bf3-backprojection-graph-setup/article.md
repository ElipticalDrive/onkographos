# Backprojection Graph Setup

Also called: backproject node configuration, task graph construction

## Overview
Constructs a processing graph node for backprojection, configuring its parameters from command-line arguments and connecting it to source and sink nodes [S1, unofficial]. It serves as the orchestration layer that initializes the backprojection operator within a larger tomographic reconstruction pipeline, bridging raw projection data ingestion with the core [Back-projection](2450-back-projection.md) computation.

## How it works
The method constructs a directed acyclic graph (DAG) representing the data flow for filtered backprojection (FBP) or iterative reconstruction. It instantiates a specific task node for backprojection, configures its geometric and computational parameters, and links it to upstream preprocessing and downstream output sinks.

### Inputs and Outputs
*   **Inputs**: Command-line argument namespace (`args`) containing geometry, processing, and I/O settings; an existing task graph object (`graph`); optional existing source node; optional processing node (`gpu`).
*   **Outputs**: A configured backprojection task node; the updated task graph with connected edges; references to the final sink node for pipeline continuation.

### Data Structures
The configuration is derived from a flat argument namespace. Key fields include:
*   **Geometry**: `center_position_x/y/z`, `source_position_x/y/z`, `detector_position_x/y/z`, `detector_angle_x/y/z`, `axis_angle_x/y/z`, `volume_angle_x/y/z`, `overall_angle`, `roll_angle` (misalignment, positive clockwise) [S1, unofficial].
*   **Computation**: `compute_type`, `result_type`, `store_type`, `addressing_mode`, `burst`, `z` (slice index), `region`, `x_region`, `y_region` [S1, unofficial].
*   **I/O**: `num_projections`, `do_output`, `dry_run`, `output` (filename pattern) [S1, unofficial].
*   **Preprocessing**: `lamino-padding-mode` (choices: 'none', 'clamp', 'clamp_to_edge', 'repeat', 'mirrored_repeat'; default 'clamp'), `cone_beam_weight` (boolean) [S1, unofficial].
*   **Reconstruction Specifics**: `crop-width` (final slice width), `projection-crop-after` ('filter' or 'backprojection'), `oversampling`, `num-iterations`, `relaxation-factor`, `lambda`, `mu`, `num-flats` (flat-field correction) [S1, unofficial].

### Steps
1.  **Instantiate Backprojection Node**: Create the backprojection task using the factory `get_task('general-backproject', processing_node=gpu)` [S1, unofficial].
2.  **Configure Parameters**: Assign all relevant fields from `args` to the backprojection node's properties. This includes geometric positions, angles, data types, regions, and reconstruction-specific flags like `cone_beam_weight` and padding modes [S1, unofficial].
3.  **Create Source Pipeline**: If a source node does not already exist, create a preprocessing pipeline via `create_preprocessing_pipeline(args, graph, source=source, processing_node=gpu, cone_beam_weight=not args.disable_cone_beam_weight, make_reader=make_reader)` [S1, unofficial]. This handles reading, flat-field correction, and optional filtering.
4.  **Connect Source to Backprojection**: If a source node exists (either pre-existing or newly created), connect it to the backprojection node using `graph.connect_nodes(source, backproject)`. If no source was provided and none was created, the backprojection node becomes the start of the pipeline (`source = backproject`) [S1, unofficial].
5.  **Create Sink (Optional)**: If `args.do_output` is true:
    *   If `args.dry_run` is true, create a 'null' sink with download capability.
    *   Otherwise, create a writer sink using `get_writer(args)` with the filename pattern `'{}-{:>03}-%04i.tif'.format(args.output, index)` [S1, unofficial].
6.  **Connect Backprojection to Sink**: If a sink was created, connect the backprojection node to it using `graph.connect_nodes(backproject, sink)`. Set the `last` variable to `sink`. Otherwise, set `last` to `backproject` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_backprojection_graph(args, graph, source=None, processing_node=None):
    # Step 1: Instantiate backprojection node
    backproject = get_task('general-backproject', processing_node=processing_node)
    
    # Step 2: Configure parameters from args
    # Map all relevant args to backproject properties
    backproject.parameter = args.parameter
    backproject.burst = args.burst
    backproject.z = args.z
    backproject.region = args.region
    backproject.x_region = args.x_region
    backproject.y_region = args.y_region
    backproject.center_position_x = args.center_position_x
    backproject.center_position_z = args.center_position_z
    backproject.source_position_x = args.source_position_x
    backproject.source_position_y = args.source_position_y
    backproject.source_position_z = args.source_position_z
    backproject.detector_position_x = args.detector_position_x
    backproject.detector_position_y = args.detector_position_y
    backproject.detector_position_z = args.detector_position_z
    backproject.detector_angle_x = args.detector_angle_x
    backproject.detector_angle_y = args.detector_angle_y
    backproject.detector_angle_z = args.detector_angle_z
    backproject.axis_angle_x = args.axis_angle_x
    backproject.axis_angle_y = args.axis_angle_y
    backproject.axis_angle_z = args.axis_angle_z
    backproject.volume_angle_x = args.volume_angle_x
    backproject.volume_angle_y = args.volume_angle_y
    backproject.volume_angle_z = args.volume_angle_z
    backproject.num_projections = args.num_projections
    backproject.compute_type = args.compute_type
    backproject.result_type = args.result_type
    backproject.store_type = args.store_type
    backproject.overall_angle = args.overall_angle
    backproject.addressing_mode = args.addressing_mode
    backproject.gray_map_min = args.gray_map_min
    backproject.gray_map_max = args.gray_map_max
    backproject.lamino_padding_mode = args.lamino_padding_mode
    backproject.cone_beam_weight = not args.disable_cone_beam_weight
    
    # Step 3: Create source pipeline if needed
    if source is None:
        source = create_preprocessing_pipeline(
            args=args,
            graph=graph,
            source=None,
            processing_node=processing_node,
            cone_beam_weight=not args.disable_cone_beam_weight,
            make_reader=make_reader
        )
    
    # Step 4: Connect source to backproject
    if source is not None:
        graph.connect_nodes(source, backproject)
    else:
        source = backproject
        
    # Step 5 & 6: Handle output sink
    sink = None
    if args.do_output:
        if args.dry_run:
            sink = get_task('null', download=True)
        else:
            # Filename pattern: '{output}-{index:03d}-{slice:04d}.tif'
            sink = get_writer(args)
            sink.filename_pattern = '{}-{:>03}-%04i.tif'.format(args.output, 0) # Index placeholder handled by writer internally
        
        graph.connect_nodes(backproject, sink)
        last = sink
    else:
        last = backproject
        
    return backproject, source, sink, last
```

### Edge Cases and Pitfalls
*   **Truncation Artifacts**: If `projection-crop-after` is set to 'filter', cropping projections before backprojection can cause truncation artifacts. The default 'backprojection' is safer [S1, unofficial].
*   **Padding Mode**: The `lamino-padding-mode` affects how filtered projections are padded. 'clamp' is the default, but 'mirrored_repeat' or 'repeat' may be preferred for specific boundary conditions [S1, unofficial].
*   **Cone-Beam Weighting**: Disabling cone-beam weighting (`disable_cone_beam_weight`) may lead to artifacts in cone-beam geometries. Ensure `cone_beam_weight` is enabled for non-parallel beam setups [S1, unofficial].
*   **Dry Run**: In dry run mode, the sink is 'null' with download, meaning no files are written, but the pipeline executes [S1, unofficial].

## Subtopics

### [Fourier-space Projection Operations](6356-fourier-space-projection-operations.md)
Uses Fourier domain for projection/backprojection; faster for large datasets but requires more memory.

### [Gradient w.r.t Source/Detector Geometry](8258-gradient-w-r-t-source-detector-geometry.md)
Computes gradients for inverse problems; essential for differentiable reconstruction.

### [Explicit Radon Matrix Construction](9109-explicit-radon-matrix-construction.md)
Builds the full Radon matrix; memory-intensive but allows iterative solvers.

### [Dirac Pixel Kernel](14354-dirac-pixel-kernel.md)
Assumes point-sampled projections; simpler but less accurate for large pixels.

### [Projective Texture Backprojection](16608-projective-texture-backprojection.md)
Uses GPU texture mapping; hardware-accelerated but limited by texture size.

### [Cone-beam Backprojection](17464-cone-beam-backprojection.md)
Specific to cone-beam geometry; requires weighting for accuracy.

### [Cone-Beam Projection/Backprojection via B-Spline Homographies](18791-cone-beam-projection-backprojection-via-b-spline-homographie.md)
Uses B-splines for interpolation; smoother results but computationally heavier.

### [Hardware Acceleration (GPU/OpenCL)](21075-hardware-acceleration-gpu-opencl.md)
Offloads computation to GPU; significant speedup but requires compatible hardware.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of voxels and projections, $O(V \cdot N)$, where $V$ is the number of voxels and $N$ is the number of projections.
*   **Memory Complexity**: $O(V + P)$, where $P$ is the size of the projection data. The graph structure adds minimal overhead.
*   **Strengths**: Modular design allows easy integration of preprocessing and postprocessing steps. Supports various geometries and reconstruction algorithms.
*   **Limitations**: Performance depends on the underlying backprojection implementation. Memory usage can be high for large volumes.
*   **When to Use**: When integrating backprojection into a larger reconstruction pipeline with custom preprocessing or output requirements.
*   **When Not to Use**: For simple, standalone backprojection tasks where a direct function call is sufficient.

## Background
The backprojection graph setup is part of the UFO-Kit/Tofu project, a tomographic reconstruction toolkit [S1, unofficial]. It provides a flexible way to configure and connect backprojection nodes within a task-based execution model.

## Sources
[S1] ufo-kit/tofu, https://github.com/ufo-kit/tofu, fetched 2026-10-26 (unofficial)
