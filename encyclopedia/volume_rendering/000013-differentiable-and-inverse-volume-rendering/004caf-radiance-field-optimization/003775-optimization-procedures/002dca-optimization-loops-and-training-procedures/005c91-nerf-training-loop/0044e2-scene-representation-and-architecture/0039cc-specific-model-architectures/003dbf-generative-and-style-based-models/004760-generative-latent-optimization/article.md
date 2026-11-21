# Generative Latent Optimization


## Overview
Generative Latent Optimization (GLO) is a training procedure for implicit scene representations where a set of latent codes is jointly optimized with the neural network parameters to minimize a reconstruction loss between rendered outputs and target images [S1]. It functions as a discriminator-less GAN, allowing the model to learn a mapping from a fixed latent distribution to the scene space without adversarial training [S1]. Unlike standard NeRF training which optimizes only network weights for a fixed scene, GLO treats the latent codes as learnable parameters, enabling the model to capture view-dependent variations or specific scene instances [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A dataset of $N$ images $\{I_1, \dots, I_N\}$, a generator network $G$ (the radiance field model), and a set of initial latent codes $\{z_1, \dots, z_N\}$ sampled from a prior distribution (typically Gaussian) [S1].
- **Outputs:** Optimized network parameters $\theta$ and optimized latent codes $\{z_1^*, \dots, z_N^*\}$ [S1].

**Data Structures:**
- **Latent Code $z_i$:** A vector of fixed dimensionality, initialized from a normal distribution $\mathcal{N}(0, I)$ [S1].
- **Generator $G$:** A neural network (e.g., MLP) parameterized by $\theta$, taking $z_i$ and positional/directional inputs $u_i$ to predict radiance field properties [S1].

**Algorithm:**
The optimization minimizes the reconstruction loss over both the generator parameters and the latent codes:

$$
\min_{G, z_1, \dots, z_N} \sum_{i=1}^{N} \mathcal{L}(G(z_i, u_i), I_i)
$$

where:
- $\mathcal{L}$ is a reconstruction loss, typically $L_2$ or $L_1$ [S1].
- $u_i$ represents auxiliary inputs not optimized over, such as 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$, which are required for NeRF-style rendering but are fixed per sample [S1].
- $G(z_i, u_i)$ denotes the rendered image or pixel value predicted by the generator for latent code $z_i$ given inputs $u_i$ [S1].

**Steps:**
1. Initialize latent codes $z_i \sim \mathcal{N}(0, I)$ for all $i=1 \dots N$ [S1].
2. Initialize generator parameters $\theta$ [S1].
3. For each optimization step:
   a. Sample a batch of images $\{I_j\}$ and corresponding auxiliary inputs $\{u_j\}$ [S1].
   b. Render predictions $\hat{I}_j = G(z_j, u_j)$ using current $z_j$ and $\theta$ [S1].
   c. Compute loss $\mathcal{L}(\hat{I}_j, I_j)$ [S1].
   d. Compute gradients with respect to both $\theta$ and $z_j$ [S1].
   e. Update $\theta$ and $z_j$ using an optimizer (e.g., Adam) [S1].

**Parameters:**
- **Loss Function:** $L_2$ is specified as the standard reconstruction loss [S1].
- **Optimizer:** Adam is commonly used [S1].
- **Latent Initialization:** Randomly sampled from a normal distribution [S1].

**Edge Cases and Pitfalls:**
- **Optimization Stability:** Jointly optimizing latent codes and network weights can lead to instability if not regularized. The latent codes may drift or collapse if the learning rates are not balanced [S1].
- **Overfitting:** Since latent codes are optimized per image, the model may overfit to the training set if the latent dimension is too large or training is excessive [S1].
- **Auxiliary Inputs:** In NeRF contexts, $u_i$ includes positional encoding and viewing direction. These must be correctly formatted and passed to the generator [S1].

## Subtopics

### [GAN Inversion Encoder and Optimization](16692-gan-inversion-encoder-and-optimization.md)
This method enables view synthesis and scene completion by inverting a trained generative radiance field (specifically a Generative Scene Network, GSN) into a latent code grid. It solves the problem of generating unseen parts of a scene or editing scenes by optimizing local latent codes rather than sampling from the prior distribution. Unlike methods relying on global coordinate systems which suffer rendering quality degradation as rotation angles increase, this approach uses a local coordinate system that is robust to rearrangements of latent codes, such as 2D rotations.

### Patch-based Latent Code Initialization
An optimization-based initialization procedure that finds a good starting latent code for finetuning by reconstructing random image patches [S2].
- **Step 1:** In each optimization step, sample 8 random patches of size $32 \times 32$ from all available input views [S2].
- **Step 2:** Initialize the new latent code vector with zeros [S2].
- **Step 3:** Optimize the latent code using Adam with $\beta_1=0.9$, $\beta_2=0.999$, and a fixed learning rate of $0.001$ for $1,500$ steps on 4 TPUs [S2].

## Operating characteristics
- **Time Complexity:** The optimization is iterative and requires backpropagation through the latent codes, which can be memory-intensive. The patch-based initialization takes approximately 10 minutes on 4 TPUs for the inversion step [S2].
- **Memory Complexity:** Storing and optimizing $N$ latent codes alongside network weights increases memory usage compared to standard NeRF training [S1].
- **Strengths:** Allows for explicit control over scene variations via latent codes; enables scene editing and completion [S1].
- **Limitations:** Requires joint optimization which can be slower to converge; prone to overfitting if not regularized [S1].
- **When to Use:** When the goal is to model view-dependent effects, transient phenomena, or to enable scene editing via latent manipulation [S1].

## Background
Generative Latent Optimization was introduced as a key aspect of view synthesis, jointly optimizing latent codes with the scene model [S1]. It is described as a discriminator-less GAN [S1]. In the 2020–2023 era, many NeRF models use latent codes to condition aspects of the scene, such as appearance and transient embeddings in NeRF-W, typically optimized using GLO [S1]. Edit-NeRF (June 2021) allowed for scene editing using image conditioning from user input, leveraging similar principles [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
