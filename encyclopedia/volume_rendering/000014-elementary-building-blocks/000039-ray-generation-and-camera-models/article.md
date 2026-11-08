# Ray generation and camera models


## Overview
Ray generation and camera models define the geometric mapping from discrete 2D pixel coordinates to continuous 3D world-space rays, serving as the foundational input for volume rendering, ray marching, and ray tracing pipelines [S2][S3, unofficial]. This family solves the problem of establishing a valid view frustum by constructing camera basis vectors and projecting pixel locations onto a virtual image plane [S3, unofficial]. It differs from sibling methods like [Ray-primitive intersection](769-ray-primitive-intersection.md) by focusing solely on the creation of the ray geometry rather than its traversal, and from [Parallel and GPU primitives](771-parallel-and-gpu-primitives.md) by defining the logical data flow rather than the hardware execution model [G].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].

**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].

**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Sky Gradient](1262-sky-gradient.md)
Sky Gradient generates a background color for rays that miss the primary volume or reach infinite distance by interpolating between a horizon color and a zenith color based on the ray's vertical direction component. It is a lightweight, non-geometry effect often implemented as a handful of sine waves or simple color blending in a sky shader.

### [Arcball Camera](1639-arcball-camera.md)
The Arcball Camera is an interactive camera control technique that maps 2D screen coordinates to a 3D virtual sphere to compute rotation quaternions for smooth orbiting around a center point. It decouples 2D input from 3D orientation, allowing intuitive rotation without gimbal lock. It differs from [First-Person Camera Control](2721-first-person-camera-control.md) by orbiting a fixed center rather than moving the eye position directly.

### [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md)
Uniform Sphere Sampling generates a random direction vector uniformly distributed over the surface of a unit sphere. It solves the problem of creating isotropic ray directions for global illumination, sky sampling, or volume scattering where no preferred direction exists. It sets itself apart from [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md) by covering the full $4\pi$ steradians rather than restricting samples to a half-sphere, and from [Rejection Sampling for Uniform Sphere Sampling](3286-rejection-sampling-for-uniform-sphere-sampling.md) by using a direct transformation method that avoids discarding samples.

### [First-Person Camera Control](2721-first-person-camera-control.md)
First-Person Camera Control manages camera position, orientation (yaw and pitch), and projection parameters, translating discrete user input (mouse movement, keyboard presses) into continuous 3D transformations. It packs these state variables into a GPU-friendly structure to minimize bandwidth overhead during rendering passes.

### [Arbitrary Source-Detector Geometry Projector](2975-arbitrary-source-detector-geometry-projector.md)
The Arbitrary Source-Detector Geometry Projector is a CT geometry type that allows the user to define the 3D location and orientation of every individual source-detector pair, overriding standard geometric assumptions. It supports flexible specification including arbitrary 3D detector shifts and non-equispaced projection angles, covering planar or curved detectors (axial) and arbitrary source/detector placement (flexible). It differs from fixed-camera models by explicitly defining every source-detector pair, supporting complex trajectories and parameterized detector surfaces while maintaining exact cell-constant integration.

### [Camera Class Structure](3022-camera-class-structure.md)
The Camera Class Structure defines the geometric state and rendering parameters required to generate rays for volume rendering. It serves as the primary interface between the application's scene graph and the ray generation kernel, encapsulating position, orientation, field of view, and clipping boundaries. Unlike sibling methods that focus on specific sampling strategies or traversal setups, this structure provides the foundational data layout necessary to compute ray origins and directions for any camera model.

### [Ray Angle Computation](3156-ray-angle-computation.md)
Ray Angle Computation calculates the geometric angular relationship between a query ray and reference rays intersecting the same 3D point. It is primarily used in multi-view stereo and NeRF-based methods to generate features that encode viewing geometry. Unlike methods that rely solely on coordinate transformations, this approach explicitly computes the dot product and difference vectors between ray directions to capture angular similarity.

### [Sky Radiance Model](3256-sky-radiance-model.md)
The Sky Radiance Model generates background illumination for ray-traced scenes by sampling a precomputed equirectangular sky texture and adding a direct sun disk contribution based on ray direction relative to the sun direction. It solves the problem of providing sky lighting without performing real-time atmospheric integration per ray. It differs from the [Sky Gradient](1262-sky-gradient.md) by using a high-resolution precomputed texture for complex atmospheric effects rather than a simple analytic gradient, and from the [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md) by including directional sun intensity and atmospheric attenuation.

### [Compute Shader Ray Marching Setup](3487-compute-shader-ray-marching-setup.md)
This method initializes the compute shader pipeline for volumetric ray marching, specifically handling camera ray reconstruction and dispatch parameters for a voxel-based smoke renderer. It differs from general ray generation by explicitly detailing the reconstruction of world-space rays from screen coordinates via inverse projection matrices and the two-stage marching logic (empty-space skipping vs. density accumulation). It serves as the computational core that bridges the camera model and the voxel data structures.

### [Grid Ray Sampler](3930-grid-ray-sampler.md)
The Grid Ray Sampler generates camera rays by mapping pixel coordinates from an image grid to 3D space, producing either a uniform set of rays covering the entire resolution or a random subset of rays for sparse sampling. It differs from its siblings by providing the foundational geometric mapping from 2D pixel indices to 3D ray origins and directions, whereas methods like [Sky Gradient](1262-sky-gradient.md) or [Arcball Camera](1639-arcball-camera.md) focus on specific camera models or environmental lighting, and [Refined Ray Generation via PDF Sampling](6837-refined-ray-generation-via-pdf-sampling.md) builds upon this uniform base by re-weighting samples based on opacity.

### [Fisheye Distortion Correction](4724-fisheye-distortion-correction.md)
Fisheye Distortion Correction eliminates the visual bowing of flat surfaces in ray-cast scenes caused by using raw Euclidean ray lengths for perspective projection. It replaces the naive distance with the perpendicular distance to the projection plane, ensuring that a flat wall renders as a straight line across the entire field of view. Unlike [Sky Gradient](1262-sky-gradient.md) or [Arcball Camera](1639-arcball-camera.md) which define viewing geometry, this method specifically corrects the depth value used for screen-space projection in planar scenes.

### [Ray Casting](6080-ray-casting.md)
Ray casting is the fundamental image synthesis technique that generates primary rays through image pixels to determine visible geometry and accumulate light contributions, serving as the base layer for more complex ray tracing algorithms. It is distinguished from full ray tracing by its exclusive use of primary (viewing) rays, omitting secondary rays for shadows, reflections, or refractions.

### [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md)
This family defines the geometric mapping from discrete pixel coordinates to continuous 3D rays, establishing the camera's extrinsic parameters and projection model. It solves the problem of generating valid ray origins and directions for volume rendering by constructing an orthonormal basis from eye/target/up vectors and applying the appropriate projection, differing from sibling methods by focusing on fundamental algebraic construction rather than traversal optimizations or control interfaces.

### [Depth of Field and Lens Effects](6232-depth-of-field-and-lens-effects.md)
This family implements optical effects that deviate from the ideal pinhole camera model, primarily depth of field (blur based on focal distance) and geometric lens distortions. It sets itself apart from sibling ray generation methods by introducing a finite aperture for ray origins, requiring sampling over a 2D area rather than a single point, and by modifying ray directions to simulate optical phenomena like magnification or shift.

### [Camera Trajectory and Path Generation](6410-camera-trajectory-and-path-generation.md)
Camera Trajectory and Path Generation defines the sequence of camera poses (positions and orientations) used to sample rays for volume rendering or novel view synthesis. It solves the problem of determining where to place virtual cameras to capture a scene from desired viewpoints, ranging from static orbits to complex helical fly-throughs. It differs from [Ray generation and camera models](770-ray-generation-and-camera-models.md) by focusing on the temporal/spatial sequence of poses rather than the instantaneous ray construction, and from [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md) by excluding the actual rendering and encoding steps, providing only the geometric path.

### [Camera Ray Generation](7456-camera-ray-generation.md)
Camera Ray Generation computes the 3D origin $\mathbf{o}$ and direction $\mathbf{d}$ for primary rays corresponding to each pixel (or sample) in an image, mapping 2D raster coordinates to 3D world-space rays for volume rendering or ray tracing. It differs from [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) by supporting perspective projection (converging rays) and from [Pinhole Camera Ray Generation](5058-pinhole-camera-ray-generation.md) by explicitly constructing and utilizing camera parameters (intrinsic/extrinsic or basis vectors) rather than assuming a canonical pinhole model. It serves as the foundational step for [Ray Casting](6080-ray-casting.md) and [Ray Marching](3487-compute-shader-ray-marching-setup.md).

### Viewing Plane Distance Adjustment
**Definition:** Minimizes aliasing caused by ray divergence by adjusting the distance between the viewer's eye and the viewing plane, reducing the angle between neighboring rays [S6, unofficial].
**Steps:** Increase the distance between the viewer's eye and the viewing plane. As distance increases, the angle of divergence between rays decreases. As distance approaches infinity, ray divergence approaches zero [S6, unofficial].

### Scene View Estimation
**Steps:** Initialize bounds with `scene.sceneBounds`. Iterate over `scene.splatSets`, merging each set's bounds into bounds using `MergeBounds`. If bounds is invalid, return default view with center `{0, 0, 2.5}` and radius `1.0`. Otherwise, return view with `center = ComputeAabbCenter(bounds)` and `radius = max(ComputeAabbRadius(bounds), 0.5f)`. Call `EstimateSceneView(scene)` to get view. Calculate `aspect = width / height`. Calculate `distance = max(view.radius * 2.5f, 1.5f)`. Set `eye = {view.center.x, view.center.y, view.center.z - distance}`. Set `input.view = LookAt(eye, view.center, {0.0f, 1.0f, 0.0f})`. Set `input.proj = Perspective(1.0f, aspect, 0.01f, max(100.0f, distance + view.radius * 4.0f))`. Set `input.cameraPosition = eye`. Set `input.viewportWidth = width`, `input.viewportHeight = height`. Set `input.nearPlane = 0.01f`. Set `input.farPlane = max(100.0f, distance + view.radius * 4.0f)`. Set `input.frameIndex = frameIndex`. Set `input.settings.antialiasing = true`, `input.settings.fastCulling = true` [S7, unofficial].

## Background
The pinhole camera model, which simulates depth by positioning the eye behind the image plane at a distance $f$ (focal length), is the standard geometric foundation for ray generation in computer graphics [S1, S3, unofficial]. Early implementations in ray marching focused on simple perspective projection to create depth illusions, moving beyond orthographic parallel casting [S1, unofficial]. The field has expanded to include arbitrary source-detector geometries for medical imaging (CT) [S5, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S3] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S4] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S5] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S6] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S7] pbkx/DirectXSplat, https://github.com/pbkx/DirectXSplat, fetched 2026-10-27 (unofficial)
