# Explicit Neural Feature Volume

Also called: Explicit Neural Feature Tensor, Purely Explicit Neural Representation

## Overview
Explicit Neural Feature Volume stores neural features as a large, directly addressable tensor, avoiding the compression into an implicit neural network found in hybrid explicit-implicit models [S1]. It solves the problem of reconstruction artifacts and excessive training overhead associated with hybrid approaches, particularly in limited-angle tomography where implicit regularization exacerbates blurring in missing wedge directions [S1]. It sets itself apart from siblings like [ACORN](8467-hybrid-volume-mesh-representation.md) by prioritizing reconstruction fidelity and training stability over final storage compactness, accepting higher intermediate memory consumption for better PSNR [S1].

## How it works
**Inputs and Outputs:**
- Input: A set of 2D projections or views of a scene/volume.
- Output: A 3D (or 4D) tensor of neural features, which is subsequently decoded to render images or reconstruct geometry.

**Data Structures:**
- **Feature Tensor:** A large, explicit tensor storing the latent features. The sources describe this as a "large tensor" [S1]. In the context of the parent family, this tensor is stored explicitly in memory, as opposed to being implicitly defined by a network [S1].
- **Octree Structure:** The explicit tensor may be organized hierarchically using an octree to manage empty space and allow for structural refinement (merging, splitting, culling leaf nodes) [S1]. This structure enables empty space skipping during rendering/tracing.

**Steps:**
1. **Initialization:** Initialize an explicit feature tensor. The sources do not specify the exact initialization values, but note that it is stored explicitly [S1].
2. **Training/Reconstruction:**
   - For each view, trace rays through the explicit feature volume.
   - Decode the features along the ray using a decoder network to obtain density and color (or other signal values) [S1].
   - Compute the loss between rendered images and ground truth.
   - Update the explicit feature tensor values directly via gradient descent.
   - Periodically perform structural octree optimization (merge, split, cull nodes) to refine the representation [S1].
3. **Rendering:**
   - Trace rays through the explicit feature tensor.
   - Decode features on-the-fly to compute the final pixel values.

**Formulas:**
- The rendering equation involves integrating the signal along the ray, where the signal is derived from the explicit features $F$ via a decoder $D$:
  $$ C = \int_{t_n}^{t_f} T(t) \mathbf{c}(t) dt $$
  where $\mathbf{c}(t) = D(F(\mathbf{x}(t)))$ [G, inferred from parent principles and S1].
- The sources do not provide a specific mathematical formula for the tensor storage or the octree operations, only describing them conceptually [S1].

**Parameters:**
- **Octree Depth/Resolution:** Not given in the sources.
- **Decoder Network Architecture:** Not given in the sources, but noted as a separate component [S1].

**Edge Cases and Pitfalls:**
- **Hybrid Explicit-Implicit Models:** Using a hybrid model (like ACORN) where features are compressed into an implicit network increases training time and intermediate memory consumption because all feature volumes must be decoded to trace rays for one view [S1].
- **Limited Angle Tomography:** Hybrid models worsen PSNR in limited angle tomography by encouraging blur in the missing wedge direction, whereas the purely explicit representation avoids this additional regularization-induced blur [S1].
- **Memory Consumption:** While the final explicit representation may be less compact than a hybrid one, the intermediate memory consumption during training is higher due to the need to decode all features for ray tracing [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for training loop with explicit neural feature volume
# Note: Specific tensor operations and decoder details are abstracted as per source limitations [S1]

def train_explicit_neural_feature_volume(projections, ground_truth_images, num_epochs):
    # Initialize explicit feature tensor and octree structure
    # The exact initialization is not specified in sources [S1]
    feature_tensor = initialize_explicit_tensor()
    octree = initialize_octree()
    
    for epoch in range(num_epochs):
        for view_index, (projection, gt_image) in enumerate(zip(projections, ground_truth_images)):
            # Trace rays through the explicit feature volume
            # This involves decoding features on-the-fly
            rendered_images = []
            for ray in rays_in_view(projection):
                # Sample points along the ray
                points = sample_points_along_ray(ray)
                
                # Retrieve features from the explicit tensor
                # This is the key difference from implicit/hybrid methods [S1]
                features = retrieve_features(feature_tensor, points)
                
                # Decode features to density and color
                # The decoder is a separate component [S1]
                densities, colors = decode_features(features)
                
                # Volume rendering integration
                color = volume_render(points, densities, colors)
                rendered_images.append(color)
            
            # Compute loss
            loss = compute_loss(rendered_images, gt_image)
            
            # Update explicit feature tensor via backpropagation
            # The gradients flow directly into the explicit tensor values [S1]
            update_tensor(feature_tensor, loss)
        
        # Periodic structural refinement
        if epoch % refinement_interval == 0:
            octree.optimize(feature_tensor) # Merge, split, cull nodes [S1]
```

## Subtopics

### [Decoder Network](2301-decoder-network.md)
A neural network component that transforms a low-dimensional feature vector into a single scalar density value, acting as a regularizer to improve reconstruction quality in ill-posed or difficult conditions. It distinguishes itself from sibling methods by explicitly mapping a stored tensor feature to output via a learnable non-linear function, whereas [Density Activation](2421-density-activation.md) may use fixed or simpler activation functions, and [Explicit Neural Feature Volume](1838-explicit-neural-feature-volume.md) defines the storage mechanism rather than the decoding logic.

### [Density Activation](2421-density-activation.md)
Density Activation converts raw neural feature outputs (pre-activations) into valid, non-negative density values suitable for volume rendering. It solves the problem of ensuring that density values remain physically meaningful (non-negative) and differentiable for gradient-based optimization. It differs from its parent [Explicit Neural Feature Volume](1838-explicit-neural-feature-volume.md) by specifically handling the final non-linear mapping step from feature space to scalar density, whereas the parent focuses on the storage and retrieval of the feature tensor itself [G].

### [DINO ViT Feature Extraction](6425-dino-vit-feature-extraction.md)
DINO ViT Feature Extraction is a method for defining transfer functions in volume rendering by leveraging high-level semantic features extracted from volumetric data using a frozen, self-supervised pre-trained Vision Transformer (ViT). It solves the problem of locality in traditional intensity-based features by providing semantic context, allowing users to select structures of interest via slice annotations and automatically identifying similar structures based on feature similarity without requiring model training. Unlike supervised learning or generative adversarial network approaches that require costly per-volume training or large labeled datasets, this method enables interactive exploration with quick inference on consumer hardware.

### [Neural Volume Construction](8453-neural-volume-construction.md)
Neural Volume Construction is the process of generating a dense, spatially aligned neural feature volume from sparse, noisy 3D primitives (such as Gaussian centers or point clouds). It solves the problem of initializing a generalizable 3D latent representation that integrates local and global scene information, which is required for subsequent decoding of geometry and appearance parameters. It differs from sibling methods by explicitly constructing a 3D tensor via a sparse 3D ConvNet, whereas [TensoRF Vector-Matrix Decomposition](9759-tensorf-vector-matrix-decomposition.md) uses factorized matrices and [DINO ViT Feature Extraction](6425-dino-vit-feature-extraction.md) relies on 2D image-based encoders.

### [TensoRF Vector-Matrix Decomposition](9759-tensorf-vector-matrix-decomposition.md)
A factorization technique for the 4D feature volume in TensoRF, factorizing it as the sum of three outer products between a matrix and a vector, leveraging vector-matrix decomposition.

## Operating characteristics
- **Time Complexity:** Training time is lower for purely explicit representations compared to hybrid explicit-implicit models, which dramatically increase training times [S1].
- **Memory Complexity:** Intermediate memory consumption during training is higher for explicit representations because all feature volumes need to be decoded to trace rays for one view [S1]. Final storage may be less compact than hybrid methods, but this is traded for better reconstruction quality [S1].
- **Measured Performance:** In limited angle tomography, purely explicit hierarchical representations achieve higher PSNR than hybrid explicit-implicit representations, which suffer from increased blur in the missing wedge direction [S1].
- **Strengths:** Better reconstruction quality in difficult conditions (e.g., limited angle tomography), lower training time compared to hybrid models, avoids additional regularization-induced blur.
- **Limitations:** Higher intermediate memory consumption, potentially less compact final storage.
- **When to use:** When reconstruction fidelity is critical, especially in limited-angle scenarios, and training time is a concern.
- **When not to use:** When final storage compactness is the primary constraint and training time/memory are less critical, or when hybrid models' regularization is beneficial (though sources suggest this is rare in tomography).

## Background
The method is described in the context of NeAT: Neural Adaptive Tomography [S1]. It was proposed as an alternative to hybrid explicit-implicit models like ACORN [Martel et al. 2021] to address issues with reconstruction quality and training efficiency in tomographic reconstruction.

## Sources
[S1] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
