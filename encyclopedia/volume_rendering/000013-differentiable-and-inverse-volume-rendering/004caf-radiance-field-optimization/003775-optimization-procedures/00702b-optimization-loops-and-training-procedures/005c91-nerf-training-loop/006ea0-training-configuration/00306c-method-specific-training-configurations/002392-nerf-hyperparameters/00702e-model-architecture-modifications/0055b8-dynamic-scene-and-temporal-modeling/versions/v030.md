# Dynamic Scene and Temporal Modeling

Also called: Dynamic NeRF, 4D Radiance Fields, Time-Conditioned NeRF

## Overview
Dynamic Scene and Temporal Modeling extends the static Neural Radiance Field (NeRF) architecture [Parent](3065-nerf-hyperparameters.md) to represent scenes that change over time, solving the "ghosting" and blurriness artifacts that occur when static NeRFs are applied to video sequences [S1][S2, unofficial]. It differs from sibling methods by augmenting the input space with temporal signals or by explicitly modeling deformation fields, rather than treating the scene as a fixed 3D volume [S1][S2, unofficial].

## Shared principles
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

Most modifications in this family retain the core volume rendering equation but alter one or more of the following components:
1.  **Input Space:** Augmenting $\mathbf{x}$ and $\mathbf{d}$ with additional signals such as latent codes $\mathbf{z}$, time $t$, illumination embeddings $\mathbf{z}_l$, or external feature vectors $\mathbf{f}$ from vision models [S1][S2, unofficial].
2.  **Network Topology:** Replacing the monolithic MLP with specialized branches (e.g., separate density and color networks), modular experts, or hybrid structures combining MLPs with convolutional or transformer backbones [S1][S2, unofficial].
3.  **Output Space:** Extending the output tuple $(\sigma, \mathbf{c})$ to include additional fields such as uncertainty, semantic logits, or physical properties like albedo and normals [S1].

## Subtopics

### [Gear-NeRF](13848-gear-nerf.md)
Gear-NeRF is a method for free-viewpoint rendering and tracking of dynamic scenes that leverages semantic information from image segmentation models to stratify the scene into motion-based 'gears', allowing for region-specific spatio-temporal sampling resolution. It addresses the inefficiency of uniform sampling in dynamic scenes by assigning higher-resolution feature volumes to regions with larger motion scales. Unlike sibling methods that use a single unified 4D volume or explicit time-parameterized MLPs, Gear-NeRF maintains a series of parallel 4D feature volumes, one for each gear level, enabling adaptive resource allocation based on local motion intensity.

### [NeRF with Time Parameter](14705-nerf-with-time-parameter.md)
This method adapts the static Neural Radiance Field (NeRF) architecture [Parent](3065-nerf-hyperparameters.md) to represent dynamic scenes by introducing a temporal dimension. It solves the problem of "ghosting" and blurry predictions in static NeRFs when applied to video sequences by allowing the scene representation to vary over time. It differs from sibling methods like [ST-NeRF](18515-st-nerf.md) or [KFD-NeRF](15551-kfd-nerf.md) by using a minimal modification: simply concatenating a positional encoding of time to the spatial input, rather than employing complex warping fields, latent codes, or separate temporal modules.

### [Forward Warping for Novel View Synthesis](15541-forward-warping-for-novel-view-synthesis.md)
Forward Warping for Novel View Synthesis is a method for rendering dynamic scenes by modeling a static scene at a canonical time step and forward warping the entire canonical field to other time steps. It addresses the many-to-one and one-to-many mapping difficulties inherent in dynamic scene reconstruction using differentiable forward warping with average splatting and an inpaint network. Unlike backward flow methods such as NDVG, this approach leverages a canonical representation to simplify the deformation field, achieving state-of-the-art performance on public datasets, particularly for canonical frames.

### [Dynamic Appearance Particle Neural Radiance Field](15550-dynamic-appearance-particle-neural-radiance-field.md)
Dynamic Appearance Particle Neural Radiance Field (DAP-NeRF) is a method for novel view synthesis in dynamic scenes using monocular video input. It solves the problem of modeling temporal dynamics without requiring multi-view data or prior geometric knowledge of moving objects. Unlike ParticleNeRF, which encodes the entire scene with particles and requires multiple camera views per frame, DAP-NeRF applies appearance particles only to dynamic elements while maintaining a static grid-based field for the background, enabling efficient linear interpolation and monocular training.

### [KFD-NeRF](15551-kfd-nerf.md)
KFD-NeRF is a dynamic neural radiance field method that models the 4D radiance field as a dynamic system with temporally varying states, estimated using a Kalman filter to combine observations from input data and predictions based on local linear motion equations. It differs from sibling methods like [Gear-NeRF](13848-gear-nerf.md) and [NeRF with Time Parameter](14705-nerf-with-time-parameter.md) by explicitly separating motion estimation into an observer branch (direct observation) and a predictor branch (history-based prediction), weighted by a learnable Kalman gain, rather than relying solely on time-conditioned feature interpolation or rigid deformation priors.

### [SLS4D](16576-sls4d.md)
SLS4D is a method for 4D novel view synthesis that represents dynamic scenes using a sparse, compact, and informative latent feature space, significantly reducing the number of network parameters compared to previous state-of-the-art methods like V4D. It distinguishes itself by abandoning direct coordinate-based feature determination in favor of relative relationships in a latent space, utilizing dense learnable time slots for temporal continuity and an attention-based interpolator decoder for spatial feature aggregation.

### [ST-NeRF](18515-st-nerf.md)
ST-NeRF is a layered radiance field representation designed for editable free-viewpoint videos of dynamic scenes containing independently moving or articulated objects. It solves the problem of spatial and temporal consistency in dynamic neural rendering by disentangling the position, deformation, and appearance of each entity into separate continuous functions spanning space and time. Unlike monolithic dynamic NeRFs that model the entire scene as a single time-dependent field, ST-NeRF uses a layered architecture where each object is a distinct entity, enabling specific editing operations such as rescaling, shifting, duplication, removal, and re-timing.

#### ER-NeRF
ER-NeRF is a Conditional NeRF framework for effective, high-accuracy dynamic portrait creation, producing dynamic portrait videos synchronized with audio [S1]. It utilizes spatial regions’ distinct contributions to steer dynamic head modeling and develops a concise and articulate Tri-Plane Hash Representation [S1]. It introduces rapid and straightforward Adaptive Pose Encoding to enhance the separation of the head and torso [S1]. ER-NeRF attains instantaneous rendering and rapid convergence using a compact model size [S1]. Additional fine-tuning might be necessary to sustain superior rendering outcomes for poses that exceed the scope of the training dataset [S1].

#### NeRF-MS
NeRF-MS is a method that diminishes vagueness in visual alterations by implementing a ternary loss to standardize the spread of visual codes across each image [S1]. It suggests a temporary decomposition unit to more effectively distinguish still scenes from dynamic objects [S1]. Trials using both real-life outdoor and artificial datasets reveal that NeRF-MS attains cutting-edge results in multi-sequence scenarios, markedly enhancing rendering efficiency through the reconstruction of 3D-consistent visuals and minimizing ghost effects [S1]. The technique might encounter difficulties in managing significant sequence discrepancies, like severe alterations in the visual aspect of scenes caused by the activation of lights [S1].

#### DCT-NeRF
DCT-NeRF is a method that learns smooth and stable trajectories for each point in space in an input video sequence using Discrete Cosine Transform (DCT) parameterization, allowing for high-quality reconstruction and consistent motion extrapolation beyond the last available frame [S2, unofficial]. It parameterizes trajectories with the Discrete Cosine Transform (DCT) to model motion [S2, unofficial]. The method handles temporal occlusions and reflects changes in reflectance over time [S2, unofficial].

## Background
The concept of extending NeRF to dynamic scenes emerged shortly after the initial publication of static NeRF [Parent](3065-nerf-hyperparameters.md). Early approaches focused on simple time concatenation, while later methods introduced complex deformation fields, canonical spaces, and hybrid representations to handle non-rigid motion and occlusions [S1][S2, unofficial]. These methods are widely used in free-viewpoint video synthesis, digital humans, and 4D scene reconstruction.

## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S2] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
