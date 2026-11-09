# Variational Autoencoder-based Compression

Also called: Autoencoders and Variational Autoencoders (VAEs), Probabilistic Autoencoder Compression, VAE-based Compression

## Overview
Variational Autoencoder-based Compression uses an encoder-decoder pair to map volumetric data into a probabilistic latent space, enabling lossy compression where the latent code replaces the original grid [S1, unofficial]. It distinguishes itself from deterministic autoencoders by imposing a probabilistic structure on the latent variables, which enables sampling and interpolation capabilities [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volumetric dataset $V$ (e.g., a 3D grid of scalar or vector values).
- Output: A compressed representation consisting of the latent code $z$ (and potentially hyperparameters for the decoder), and a reconstructed volume $\hat{V}$.

**Data Structures:**
- **Encoder Network ($E$):** A neural network (typically convolutional) that maps $V$ to latent parameters.
- **Latent Space:** A continuous vector space where each point $z$ represents a compressed version of the input.
- **Decoder Network ($D$):** A neural network that maps $z$ back to the reconstruction space.
- **Latent Code:** The compressed output, typically a tensor of lower dimensionality than $V$.

**Method Steps:**
1. **Encoding:** The input volume $V$ is passed through the encoder $E$ to produce latent parameters, typically a mean $\mu$ and variance $\sigma^2$ for a Gaussian distribution [S1, unofficial].
2. **Reparameterization:** To enable backpropagation through the stochastic sampling process, the latent vector $z$ is sampled using the reparameterization trick:
   $$ z = \mu + \sigma \odot \epsilon $$
   where $\epsilon \sim \mathcal{N}(0, I)$ is noise sampled from a standard normal distribution [S1, unofficial].
3. **Compression:** The latent vector $z$ (and potentially $\mu, \sigma$ if exact reconstruction is required, though often only $z$ is stored for compression) replaces the original grid $V$. This constitutes the compressed form [S1, unofficial].
4. **Decoding:** The latent code $z$ is passed through the decoder $D$ to reconstruct the volume:
   $$ \hat{V} = D(z) $$
5. **Training:** The network is trained to minimize a loss function that balances reconstruction fidelity and latent space regularization (e.g., KL divergence between the learned latent distribution and a prior, typically $\mathcal{N}(0, I)$) [S1, unofficial].

**Parameters:**
- **Latent Dimensionality:** Not given in the sources.
- **Network Architecture:** Typically 3D convolutions [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Decoding Bottleneck:** Standard autoencoders or diffusion-based methods in data space can suffer from severe decoding bottlenecks, making them impractical for interactive use [S1, unofficial].
- **Precision:** The probabilistic nature allows for interpolation but introduces inherent reconstruction error dependent on the training data distribution [S1, unofficial].

## Subtopics

### Latent Space Diffusion
Moves diffusion into latent space to address decoding bottlenecks. Compresses only selected keyframes via a VAE with hyperprior, then uses a conditional latent diffusion model with factorized space-time attention to interpolate latent representations for non-keyframes [S1, unofficial]. Fine-tunes from 1,000 to 32 denoising steps to preserve compression quality [S1, unofficial]. Achieved up to 10× higher compression ratios than SZ3 on S3D combustion data and 4× on E3SM climate data [S1, unofficial]. Raised decoding throughput to 61 MB/s on an A100 GPU, 200× faster than GCDTC’s data-space diffusion [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** The latent code replaces the original grid, reducing memory footprint. Specific compression ratios vary by dataset (e.g., up to 10× higher than SZ3 on S3D combustion data) [S1, unofficial].
- **Measured Performance:** On an A100 GPU, decoding throughput reached 61 MB/s for latent space diffusion variants, compared to 978 seconds for data-space diffusion (GCDTC) and 12 seconds for SZ3 on comparable tasks [S1, unofficial].
- **Strengths:** Enables lossy compression of large volumes and captures semantically meaningful structure in the latent space for feature learning [S1, unofficial].
- **Limitations:** Standard autoencoders may have decoding bottlenecks; probabilistic models require careful training to ensure latent space continuity [S1, unofficial].

## Background
Autoencoders and Variational Autoencoders (VAEs) have been explored for scientific data compression before more recent frameworks matured. Earlier works included feature-aware in-situ autoencoders achieving 32× reduction on particle data, GAN in-situ compressors reaching 64:1 on CFD pressure fields, and hierarchical vector-quantized autoencoders achieving 231× compression at 50 dB on climate data [S1, unofficial]. VAEs specifically impose a probabilistic latent structure, enabling sampling and interpolation, which distinguishes them from deterministic autoencoders [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
