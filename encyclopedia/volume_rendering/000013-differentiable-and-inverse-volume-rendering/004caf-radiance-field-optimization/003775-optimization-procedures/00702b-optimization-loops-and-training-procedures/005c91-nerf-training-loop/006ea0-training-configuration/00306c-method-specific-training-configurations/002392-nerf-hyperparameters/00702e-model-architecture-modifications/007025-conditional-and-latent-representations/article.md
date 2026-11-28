# Conditional and Latent Representations

Also called: Conditional NeRF, Latent NeRF, Disentangled Conditional NeRF

## Overview
Conditional and Latent Representations extend the standard NeRF architecture by introducing low-dimensional latent codes $\mathbf{z}$ that condition the radiance field, enabling generative modeling of object categories rather than single-scene reconstruction [S1]. Unlike standard NeRF which maps $(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, these methods map $(\mathbf{x}, \mathbf{d}, \mathbf{z}) \rightarrow (\sigma, \mathbf{c})$, allowing for the manipulation of shape, appearance, identity, or expression through the latent space [S1]. This family is distinguished from siblings like [Feature Fields](12995-feature-fields.md) by its focus on *generative* control via learned latent distributions (often trained within GAN frameworks) and *disentangled* control mechanisms that prevent cross-talk between geometric and photometric attributes [S1].

## Shared principles
The core mechanism involves augmenting the input space of the implicit function $F_\Theta$ with latent vectors $\mathbf{z}$ that represent semantic attributes (e.g., shape, appearance, identity) [S1][S2, unofficial].

1.  **Latent Conditioning:** The network input is augmented with $\mathbf{z}$. A trivial concatenation $F_{\theta'}(\mathbf{x}, \mathbf{v}, \mathbf{z}_s, \mathbf{z}_a) : \Gamma(\mathbf{x}) \oplus \mathbf{z}_s, \Gamma(\mathbf{v}) \oplus \mathbf{z}_a \rightarrow (\mathbf{c}, \sigma)$ often leads to mutual intervention, where shape changes affect color [S1].
2.  **Disentanglement Strategy:** To achieve independent control, methods typically separate the influence of $\mathbf{z}_s$ (shape) and $\mathbf{z}_a$ (appearance) [S1].
    *   **Shape Conditioning:** Often implemented via explicit volumetric deformation. A deformation network $T$ maps $(\mathbf{x}, \mathbf{z}_s) \rightarrow \Delta \mathbf{x}$, where $\Delta \mathbf{x}$ are displacement vectors applied to the positional encoding bands [S1].
    *   **Appearance Conditioning:** Often deferred to the color prediction branch. The appearance code $\mathbf{z}_a$ is concatenated with the view direction $\mathbf{v}$ or intermediate feature vectors, ensuring it does not influence the density prediction $\sigma$ [S1].
3.  **Generative Training:** Many conditional NeRFs are trained as generative models using adversarial losses (GANs) to ensure photorealism and consistency across the latent space [S2, unofficial].
4.  **Latent Space Manipulation:** Once trained, the latent codes can be manipulated directly (e.g., scaling, interpolation) or via feed-forward mappers (e.g., CLIP-driven) to edit the rendered output without per-sample optimization [S1].

## Subtopics

### [Disentangled Conditional NeRF](16793-disentangled-conditional-nerf.md)
A generative radiance field architecture that achieves individual control over the shape and appearance of 3D objects by isolating their conditioning mechanisms. Unlike standard conditional NeRFs where concatenating latent codes to position features causes mutual intervention (e.g., shape changes altering color), this method uses explicit volumetric deformation for shape conditioning and direct latent injection for appearance, preventing cross-talk between the two attributes. It is typically trained in two stages: first, a GAN-based training of the generator and deformation network, and second, the training of CLIP-driven mappers for text-based manipulation.

### [HeadNerf](18595-headnerf.md)
HeadNeRF is a NeRF-based parametric head model that integrates neural radiance fields into the parametric representation of the human head to render high-fidelity head images in real-time while supporting direct control over rendering pose and semantic attributes (identity, expression, albedo, illumination). It differs from standard NeRF by conditioning the implicit function on low-dimensional latent codes rather than just spatial coordinates and view directions, and it accelerates rendering by combining volumetric rendering with a 2D neural refinement network. This approach allows for the generation of novel views and semantic editing (e.g., changing expression or identity) without the computational cost of full volumetric integration for the final high-resolution image.

### CLIP-Driven Manipulation via Mappers
A feed-forward approach using shape mapper $M_s$ and appearance mapper $M_a$ to update initial shape/appearance codes $(z_{s0}, z_{a0})$ based on a text prompt $t$, avoiding per-sample optimization [S1]. The updated shape code $z_s$ is calculated as $z_s = M_s(\hat{E}_t(t)) + z_{s0}$, and the appearance code $z_a$ as $z_a = M_a(\hat{E}_t(t)) + z_{a0}$, where $\hat{E}_t$ is the pre-trained CLIP text encoder [S1]. The disentangled conditional NeRF $F_\theta$ maps deformed position and view/appearance codes to color and density: $F_\theta(\mathbf{x}, \mathbf{v}, z_s, z_a) : \Gamma^*(\mathbf{x}, z_s), \Gamma(\mathbf{v}) \oplus z_a \rightarrow (\mathbf{c}, \sigma)$ [S1]. The cross-modal CLIP distance $D_{CLIP}(\mathbf{I}, t)$ measures similarity between image patch $\mathbf{I}$ and text $t$: $D_{CLIP}(\mathbf{I}, t) = 1 - \langle \hat{E}_i(\mathbf{I}), \hat{E}_t(t) \rangle$ [S1].

### Continuous Manipulation via Latent Space Scaling
A technique to achieve gradual editing of shape and appearance by scaling the editing direction vectors in the latent space using a scalar factor $s$ [S1]. The codes are updated as $z_s = s \times \Delta z_s + z_{s0}$ and $z_a = s \times \Delta z_a + z_{a0}$, where $\Delta z_s$ and $\Delta z_a$ are the manipulation directions provided by the code mappers [S1]. The scalar $s$ controls the intensity of the edit, with a typical range of 0 to 1.6 with step 0.2; manipulation strength increases as $s$ increases [S1]. This is used for continuous editing of both shape and appearance given a single text prompt or exemplar, allowing users to adjust the degree of change [S1].

### CLIP-Driven Manipulation
A method that differs from optimization-based editing (e.g., EditNeRF) by using learned feed-forward networks mapping user edits to latent codes, enabling fast inference for interactive editing instead of slow iterative optimization per sample [S1]. The steps are: 1. Extract CLIP feature embedding from input text prompt or exemplar image. 2. Feed embedding into two separate feed-forward code mappers (shape mapper and appearance mapper). 3. Mappers output local displacements in the latent space to update $z_s$ and $z_a$. 4. Render image with updated codes. 5. Train mappers using a CLIP-based matching loss to enforce CLIP space consistency between input constraint and output renderings [S1].

### Deferred Appearance Conditioning
A technique where appearance conditioning is deferred to concatenate the appearance code with the view direction as input to the radiance prediction network, allowing appearance manipulation without affecting shape/density [S1]. In NeRF, density is predicted first as a function of position; by deferring appearance conditioning, the method ensures that changes to the appearance code do not alter the underlying geometry [S1].

### Conditional NeRF in GAN Framework
The first approach to leverage a conditional NeRF in the GAN framework, achieving significant improvements in photorealism for 3D scene generation given only 2D images [S2, unofficial]. This framework enables learning a 3D generative model of shape and appearance, building upon earlier voxel-grid parameterizations [S2, unofficial].

## Background
The concept of conditional NeRFs emerged as a solution to the limitation of standard NeRF, which requires per-scene optimization and cannot generalize across object categories [S1]. The foundational work on conditional NeRFs within a GAN framework was proposed by [GRAF](https://doi.org/10.1109/cvpr52688.2022.00381) (cited in [S2, unofficial]), which demonstrated significant improvements in photorealism over voxel-based generative models [S2, unofficial]. Subsequent works, such as [CLIP-NeRF](https://doi.org/10.1109/cvpr52688.2022.00381) (Wang et al., 2022), introduced disentangled latent codes for shape and appearance, enabling text-and-image driven manipulation via CLIP embeddings [S1]. [HeadNerf](https://doi.org/10.1109/cvpr52688.2022.00381) extended this to parametric head modeling, integrating latent codes for identity, expression, albedo, and illumination [S1]. These methods are widely used in 3D content generation, virtual try-on, and interactive editing of neural radiance fields [S1].

## Sources
[S1] CLIP-NeRF: Text-and-Image Driven Manipulation of Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00381, fetched 2026-10-29
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
