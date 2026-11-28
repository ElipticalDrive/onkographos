# Mixed Precision GAN Generator Training

Also called: AMP GAN training, DeepScan3D GAN Trainer, GAN Training with Mixed Precision

## Overview
A GAN training loop variant that applies Automatic Mixed Precision (AMP) specifically to the generator update step to reduce memory usage and accelerate computation on CUDA devices [S1, unofficial]. It distinguishes itself from sibling methods by isolating AMP usage to the generator, often delegating discriminator updates to a base class or non-AMP path to avoid compatibility issues with specific regularization techniques like spectral normalization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Generator model $G$, Discriminator model $D$, Generator optimizer $\mathcal{O}_G$, latent vector $z$, condition $y$, batch of real images (implicitly for loss computation context), and a boolean flag `use_amp`.
- **Outputs:** Generator loss value, updated generator weights.

**Data Structures:**
- `scaler`: An instance of `torch.cuda.amp.GradScaler` used to scale gradients and manage overflow detection during backpropagation [S1, unofficial].
- `g_optimizer`: The optimizer instance for the generator [S1, unofficial].

**Parameters:**
- `use_amp` (bool): Enables AMP if `True`. Default behavior in the cited implementation checks this flag to wrap the forward pass [S1, unofficial].
- `scaler`: Initialized as `torch.cuda.amp.GradScaler()` when AMP is enabled [S1, unofficial].

**Steps:**
1. **Gradient State Setup:** Set the generator's gradients to be computed (`requires_grad=True`) and the discriminator's gradients to be disabled (`requires_grad=False`) to prevent updating $D$ during the generator step [S1, unofficial].
2. **Mode Setup:** Set both $G$ and $D$ to training mode (`train()`) to enable dropout and batch normalization updates [S1, unofficial].
3. **Optimization Reset:** Zero the gradients of the generator optimizer $\mathcal{O}_G$ [S1, unofficial].
4. **Forward Pass (AMP Context):** If `use_amp` is true, enter the `torch.cuda.amp.autocast()` context. Generate fake samples $x_{fake} = G(z, y)$ [S1, unofficial]. Compute the discriminator's output on these samples $d_{fake} = D(x_{fake}, y)$ [S1, unofficial].
5. **Loss Computation:** Calculate the generator loss `gloss`. In the standard GAN formulation described, this is typically computed as `compute_loss(d_fake, 1)`, implying a target label of 1 (real) for the generator's objective [S1, unofficial].
6. **Backpropagation and Update:**
   - If AMP is enabled:
     a. Scale the loss: `scaled_loss = scaler.scale(gloss)` [S1, unofficial].
     b. Perform backward pass: `scaled_loss.backward()` [S1, unofficial].
     c. Step the optimizer: `scaler.step(g_optimizer)` [S1, unofficial].
     d. Update the scaler: `scaler.update()` [S1, unofficial].
   - If AMP is disabled:
     a. Perform backward pass directly: `gloss.backward()` [S1, unofficial].
     b. Step the optimizer directly: `g_optimizer.step()` [S1, unofficial].
7. **Validation:** Assert that the batch size matches expected dimensions if required by the base class logic [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generator_trainstep(generator, discriminator, g_optimizer, scaler, use_amp, z, y):
    """
    Performs a single training step for the GAN generator using mixed precision if enabled.
    
    Args:
        generator (torch.nn.Module): The generator network.
        discriminator (torch.nn.Module): The discriminator network.
        g_optimizer (torch.optim.Optimizer): Optimizer for the generator.
        scaler (torch.cuda.amp.GradScaler): Gradient scaler for AMP.
        use_amp (bool): Whether to use Automatic Mixed Precision.
        z (torch.Tensor): Latent noise vector.
        y (torch.Tensor): Conditional input vector.
        
    Returns:
        float: The computed generator loss.
    """
    # 1. Set gradient requirements
    # Generator gradients ON, Discriminator gradients OFF
    for param in generator.parameters():
        param.requires_grad = True
    for param in discriminator.parameters():
        param.requires_grad = False
        
    # 2. Set models to train mode
    generator.train()
    discriminator.train()
    
    # 3. Zero gradients for generator optimizer
    g_optimizer.zero_grad()
    
    # 4. Forward pass
    if use_amp:
        with torch.cuda.amp.autocast():
            # Generate fake samples
            x_fake = generator(z, y)
            # Get discriminator output
            d_fake = discriminator(x_fake, y)
            # Compute loss (target label 1 for generator success)
            gloss = compute_loss(d_fake, 1)
            
            # 5. AMP Backward and Update
            # Scale loss to prevent underflow
            scaled_loss = scaler.scale(gloss)
            scaled_loss.backward()
            
            # Step optimizer
            scaler.step(g_optimizer)
            
            # Update scaler (unscales gradients and checks for overflow)
            scaler.update()
    else:
        # Non-AMP path
        x_fake = generator(z, y)
        d_fake = discriminator(x_fake, y)
        gloss = compute_loss(d_fake, 1)
        
        gloss.backward()
        g_optimizer.step()
        
    return gloss.item()

def compute_loss(d_fake, target_label):
    """
    Placeholder for the specific loss function (e.g., BCEWithLogitsLoss).
    In the source, this is abstracted as compute_loss(d_fake, 1).
    """
    # Assuming standard GAN loss where generator wants discriminator to output 1
    # Implementation detail: [G] Standard BCEWithLogitsLoss or similar is implied
    # by the context of "compute_loss(d_fake, 1)".
    return torch.nn.functional.binary_cross_entropy_with_logits(d_fake, target_label)
```

**Edge Cases and Pitfalls:**
- **Spectral Norm Compatibility:** The discriminator training step in this implementation does *not* use AMP. Spectral normalization layers raise errors when used with AMP in the specific PyTorch version/environment cited [S1, unofficial]. Consequently, the discriminator update is delegated to a base class implementation that likely handles this without AMP [S1, unofficial].
- **Batch Size Mismatch:** The implementation includes an assertion to ensure batch size consistency, which must be handled if custom batching logic is introduced [S1, unofficial].

## Subtopics

### Mixed Precision & Gradient Scaling/Clipping
Focuses on the mechanics of gradient scaling and clipping strategies to stabilize mixed precision training. Choose this when dealing with gradient overflow/underflow issues specifically. [Mixed Precision & Gradient Scaling/Clipping](8686-mixed-precision-gradient-scaling-clipping.md)

## Operating characteristics
- **Time Complexity:** Reduced per-step computation time due to FP16 operations in the generator forward/backward pass, provided the hardware supports Tensor Cores [S1, unofficial].
- **Memory Complexity:** Reduced memory footprint for activations and gradients in the generator, allowing for larger batch sizes or models [S1, unofficial].
- **Strengths:** Accelerates training speed and reduces VRAM usage for the generator component [S1, unofficial].
- **Limitations:** Requires careful handling of components incompatible with AMP (e.g., spectral norm in the discriminator in this specific codebase) [S1, unofficial]. Stability depends on proper gradient scaling [S1, unofficial].
- **When to use:** When training large GAN generators on CUDA hardware with limited VRAM and when the discriminator architecture is compatible with AMP or handled separately.
- **When not to use:** If the discriminator relies on operations strictly incompatible with AMP and cannot be offloaded to a non-AMP path, or if hardware lacks FP16 support.

## Background
This method originates from the DeepScan3D project, which reconstructs 3D CT volumes from single X-ray projections using Neural Radiance Fields (NeRF) and GANs [S1, unofficial]. The implementation is part of the `graf-main/render_xray_G.py` script within the DeepScan3D repository, leveraging the `GAN_stability` submodule for training utilities [S1, unofficial]. It was developed to enable efficient training of 3D-aware GANs on medical imaging data where memory constraints are significant [S1, unofficial].

## Sources
[S1] sanchit1606/DeepScan3D-Single-X-ray-to-3D-CT-Reconstruction-using-Neural-Radiance-Fields-, https://github.com/sanchit1606/DeepScan3D-Single-X-ray-to-3D-CT-Reconstruction-using-Neural-Radiance-Fields-, fetched 2026-10-26 (unofficial)
