# Object Decomposition and Composition

Also called: Scene Decomposition, Compositional NeRF, Object-Centric NeRF

## Overview
Object Decomposition and Composition methods address the limitation of monolithic NeRFs in handling individual objects within a scene, enabling instance-level editing, segmentation, and interpolation [S1]. Unlike standard NeRFs that treat the scene as a single continuous field, these methods decompose the representation into distinct components (foreground/background, object instances, or articulated parts) and compose them for rendering [S1]. This family is distinguished from sibling methods like [Feature Fields](12995-feature-fields.md) or [Conditional and Latent Representations](15989-conditional-and-latent-representations.md) by its explicit focus on structural separation of scene entities rather than just feature augmentation or conditional generation.

## Shared principles
The family shares the core principle of splitting the scene representation into multiple, distinct radiance fields or latent codes that correspond to semantic or geometric entities.
1.  **Decomposition:** The scene is partitioned into $K$ object instances and a background component. This can be achieved via:
    *   **Explicit Fields:** Separate NeRFs or MLP heads for each entity [S1].
    *   **Latent Codes:** Learnable vectors $\mathbf{z}_i$ associated with each instance, concatenated with shared features [S1].
    *   **Object Codes:** One-hot or soft assignment vectors predicting instance membership for each 3D point [S1].
2.  **Composition:** The final radiance $(\sigma, \mathbf{c})$ is computed by combining the outputs of the decomposed components. Common strategies include:
    *   **Weighted Sum:** Using softmax or Gumbel-Softmax weights based on density or learned probabilities [S1].
    *   **Dominant Selection:** Selecting the component with the highest density or probability [S1].
    *   **Residual Addition:** Adding a global background field to local object fields [S1].
3.  **Supervision:** Training often relies on 2D object masks or semantic labels to supervise the decomposition, as 3D ground truth for object boundaries is rarely available [S1].

## Subtopics

### [Fig-NeRF](12006-fig-nerf.md)
Fig-NeRF decomposes the scene into a foreground object and a background, modeling the object as a deformable template conditioned on a latent code $z$. It enables shape interpolation and amodal segmentation by learning a deformation field $T(x, z)$ that maps a template shape to the observed instance. It is chosen when shape variation explanation and object-background separation are required.

### [DM-NeRF](13525-dm-nerf.md)
DM-NeRF introduces an explicit "object field" that predicts a one-hot object code for every 3D point, enabling instance-level decomposition and manipulation (translation, rotation, scaling) from 2D images alone. It distinguishes itself by supporting simultaneous reconstruction, decomposition, and *manipulation* of objects, including handling occlusions during editing. It is chosen when interactive object editing is a primary goal.

### [Unified Decompositional and Compositional NeRF](14413-unified-decompositional-and-compositional-nerf.md)
UDC-NeRF unites global synthesis and local editing in a single pipeline using a coarse global field to guide sampling for a fine-grained stage that decomposes the scene into distinct object and background components. It uses Gumbel-Softmax for composition and 3D one-hot density regularization to separate objects from the background. It is chosen when joint novel view synthesis and object editing are needed without separate training stages.

### [Detail-based Segmentation](15935-detail-based-segmentation.md)
This method decomposes a scene into multiple NeRFs based on the detail frequency of detected objects, assigning high-frequency objects to dedicated NeRFs and low-frequency objects to a joint NeRF. It reduces the learning burden by scaling up high-frequency objects during training. It is chosen when computational efficiency for complex scenes with varying object detail levels is prioritized.

### [Neural Articulated Radiance Field (NARF)](18597-neural-articulated-radiance-field-narf.md)
NARF models articulated objects by decomposing them into rigid parts, each represented by a separate RT-NeRF, and combines them using a differentiable softmax weighting scheme based on density. It solves part dependency by explicitly estimating rigid transformations via forward kinematics. It is chosen for modeling articulated objects with known kinematic structures.

#### Dual-Payway NeRF
A division of NeRF into a dual-pathway architecture comprising a scene (background) branch for encoding scene geometry/appearance and an object (foreground) branch for individual object encoding [S1]. It eliminates the need for separate training per object by assigning activation codes to training rays for each object [S1]. For view generation with manipulation, it renders transformed objects using the conditioned object branch and the background from the scene branch together, using an object manipulator to handle collisions and occlusions [S1].

#### CompoNeRF / Set-the-Scene
Methods that address guidance collapse in scene-level editing by representing scenes as compositions of multiple locally optimized NeRFs integrated globally [S1]. They decompose and reassemble scenes through proxy manipulation for editing without additional fine-tuning, ensuring entity distinctiveness while maintaining scene coherence [S1].

## Background
The concept of decomposing NeRFs emerged from the need to edit individual objects within a scene, a task difficult for monolithic representations [S1]. Early works like [Fig-NeRF](12006-fig-nerf.md) focused on figure-ground separation and shape deformation . Subsequent research, including works cited in [S1] (e.g., [204, 248, 257, 8]), developed dual-pathway architectures for simultaneous object encoding [S1]. More recent methods like [DM-NeRF](13525-dm-nerf.md) and [UDC-NeRF](14413-unified-decompositional-and-compositional-nerf.md) have refined this by introducing explicit object fields and unified decomposition-composition pipelines [S1]. Methods like [CompoNeRF](12492-activation-feature-density-approximation.md) and [Set-the-Scene](12492-activation-feature-density-approximation.md) address guidance collapse in text-guided editing by composing multiple local NeRFs [S1].

## Sources
[S1] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
