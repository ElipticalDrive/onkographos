# Transformer and Vision Model Integration

Also called: Vision-Transformer NeRF, Generalizable NeRF with Vision Backbones

## Overview
This family integrates Vision Transformers (ViT) and other vision models into NeRF architectures to replace or augment the standard coordinate-based MLP, enabling generalizable novel view synthesis from single or few images without per-scene optimization [S1]. It solves the limitation of standard NeRF, which requires expensive per-scene training, by leveraging global semantic and local appearance features extracted from pre-trained vision backbones to condition the radiance field [S1]. Unlike sibling methods that rely on latent codes or explicit voxel grids, this approach directly maps 2D image features to 3D spatial queries via attention or interpolation mechanisms [S1].

## Shared principles
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

Most modifications in this family retain the core volume rendering equation but alter one or more of the following components:
1.  **Input Space:** Augmenting $\mathbf{x}$ and $\mathbf{d}$ with additional signals such as latent codes $\mathbf{z}$, time $t$, illumination embeddings $\mathbf{z}_l$, or external feature vectors $\mathbf{f}$ from vision models [S1].
2.  **Network Topology:** Replacing the monolithic MLP with specialized branches (e.g., separate density and color networks), modular experts, or hybrid structures combining MLPs with convolutional or transformer backbones [S1].
3.  **Output Space:** Extending the output tuple $(\sigma, \mathbf{c})$ to include additional fields such as surface normals $\mathbf{n}$, semantic logits $\mathbf{s}$, uncertainty variance $\nu$, or blend weights $\Delta w$ [S1].
4.  **Representation Basis:** Substituting the coordinate-based implicit function with explicit structures like voxel grids, hash tables, Gaussian primitives, or planar surfaces, which change how features are stored and interpolated [S1].

## Subtopics

### [ViT-NeRF](13878-vit-nerf.md)
ViT-NeRF combines a Vision Transformer (ViT) to encode global shape information and a 2D CNN to extract local appearance features, conditioning volume rendering on hybrid per-pixel features. It addresses the limitation of using only global features, which compromises rendering quality for views close to the input view, by fusing them with local CNN features. Unlike CodeNeRF and DISN, it does not require a canonical coordinate system to utilize global features. See [ViT-NeRF](13878-vit-nerf.md).

### [VisionNeRF](13885-visionnerf.md)
VisionNeRF infers a 3D radiance field from a single unposed input image by combining global features learned from a ViT and local appearance features extracted from a 2D CNN. It solves the problem of rendering quality degradation in occluded regions, where standard image-conditioned models incorrectly query features from visible surfaces that do not correspond to the target view's geometry. Unlike sibling methods that rely on per-scene optimization or latent codes, VisionNeRF uses an explicit mapping between input image pixels and 3D point queries, enabling generalization across object categories without requiring camera pose information. See [VisionNeRF](13885-visionnerf.md).

### [TransNeRF](13907-transnerf.md)
TransNeRF is a generalizable neural radiance field method that learns a generic scene representation for novel view synthesis using an encoder-decoder Transformer framework, avoiding per-scene optimization. Unlike standard NeRF which optimizes a scene-specific MLP, TransNeRF aggregates multi-view information from source images into a coordinate-based representation via attention mechanisms, enabling generalization to unseen scenes without fine-tuning. It differs from previous pooling-based multi-view representations by exploring deep relationships amongst multiple views and aggregating them into the radiance field via a single Transformer-based architecture. See [TransNeRF](13907-transnerf.md).

### [NeRF Analogies](18133-nerf-analogies.md)
NeRF Analogies transfers visual attributes (appearance) from a source NeRF to a target NeRF by establishing dense semantic correspondences using Vision Transformer (ViT) features. It solves the problem of appearance editing in 3D by creating a "NeRF analogy" where the target object adopts the texture and material properties of the source object based on semantic similarity rather than geometric proximity. It is distinguished from sibling methods like [ViT-NeRF](13878-vit-nerf.md) and [VisionNeRF](13885-visionnerf.md) by its explicit use of cross-scene semantic matching to drive a differentiable loss function, rather than using vision features solely for conditioning a single-scene reconstruction. See [NeRF Analogies](18133-nerf-analogies.md).

#### Mixture of View Experts (MoE) in GNT
- **Definition:** A modification to the GNT view transformer where dense MLP layers are replaced with sparsely activated Mixture-of-Experts (MoE) layers to specialize different components for specific rendering properties (e.g., diffuse vs. specular reflection) [S1].
- **Step:** Replace the dense MLP layer in the view transformer with a sparsely activated MoE layer composed of a set of half-sized MLP experts $\{f_e\}_{e=1}^E$ [S1].
- **Formula:** The routing weights are computed as $R(x) = \text{softmax}(\text{TopK}(G(x), K))$, where $G$ is the learnable network within the router [S1]. The output is the weighted summation of the selected experts. A diversity regularizer $L_{div} = \text{CV}(\mathbb{E}_{r \sim R} \mathbb{E}_{x \in r} R(x))$ is applied, where $\text{CV}(g) = \text{mean}(g)/\text{var}(g)$, to encourage different views to exploit the expert space and different experts to capture nuances [S1].
- **Parameters:** $K=2$ (number of top experts selected), $E=4$ (total expert candidates), Expert Size is half the dense MLP size, and $\lambda_{div} = 1 \times 10^{-3}$ [S1].

## Background
The integration of vision transformers into NeRFs emerged to address the computational cost of per-scene optimization in standard NeRF [S1]. Early methods like ViT-NeRF and VisionNeRF demonstrated that pre-trained vision backbones could provide robust global and local features for single-image novel view synthesis [S1]. TransNeRF extended this by using transformer decoders to aggregate multi-view information without per-scene tuning [S1]. NeRF Analogies later applied these semantic features for cross-scene appearance transfer [S1]. The use of Mixture-of-Experts in generalizable NeRFs (GNT) further refined this by specializing transformer layers for different material properties [S1].

## Sources
[S1] Enhancing NeRF akin to Enhancing LLMs: Generalizable NeRF Transformer with Mixture-of-View-Experts, https://doi.org/10.1109/iccv51070.2023.00296, fetched 2026-10-26
