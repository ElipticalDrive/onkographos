# Contrastive Diffusion for Volume Super-Resolution (CD-TVD)
Also called: CD-TVD, Contrastive Diffusion for Volume SR

## Overview
CD-TVD is the first diffusion-based approach for scientific volume super-resolution, designed to upscale low-resolution volumetric data to higher resolutions [S1, unofficial]. It distinguishes itself from prior GAN-based methods by combining contrastive pre-training on historical simulation data with fine-tuning from as few as one high-resolution timestep, achieving higher fidelity (43–48 dB PSNR) on fluid dynamics datasets [S1, unofficial]. Unlike standard image SR methods that rely on VGG-based perceptual losses, CD-TVD employs a contrastive encoding module to learn degradation-aware features in latent space, providing domain-specific representations better suited for scientific data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Low-resolution (LR) 3D volume data (e.g., fluid dynamics simulations).
*   **Output:** High-resolution (HR) 3D volume data.
*   **Scaling:** Typically 4× upscaling in spatial dimensions [S1, unofficial].

**Data Structures:**
*   **Latent Space:** A compressed representation of the volume where degradation-aware features are learned via contrastive encoding [S1, unofficial].
*   **Denoising Network:** A 3D Swin Transformer-based architecture that operates in the latent space to predict the clean signal from noisy inputs [S1, unofficial].

**Algorithm Steps:**
1.  **Contrastive Pre-training:**
    *   Train a contrastive encoding module on historical simulation data.
    *   The goal is to learn degradation-aware features in the latent space.
    *   This replaces the VGG-based perceptual losses typically used in image SR with representations specific to the scientific domain [S1, unofficial].
2.  **Fine-tuning:**
    *   Adapt the pre-trained model to a new simulation scenario using as few as one high-resolution timestep as an exemplar [S1, unofficial].
    *   This step allows for rapid adaptation to new data distributions without extensive retraining [S1, unofficial].
3.  **Entropy-based Key-Timestep Selection:**
    *   To facilitate fine-tuning, select a key timestep from the new simulation data based on entropy metrics [S1, unofficial].
    *   This selection identifies the most informative timestep for adaptation [S1, unofficial].
4.  **Diffusion Upsampling:**
    *   Use the Swin Transformer-based 3D denoising network to upscale the LR volume.
    *   The network iteratively denoises the latent representation to recover the HR volume [S1, unofficial].

**Parameters:**
*   **Upscaling Factor:** 4× (spatial) [S1, unofficial].
*   **Training Data:** Historical simulation data for pre-training; single HR timestep for fine-tuning [S1, unofficial].
*   **Selection Metric:** Entropy-based for key-timestep selection [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Single Exemplar Adaptation:** The method relies on the quality of the single high-resolution timestep used for fine-tuning. If this exemplar is not representative of the target distribution, performance may degrade [S1, unofficial].
*   **Volume Size Limitations:** Like other methods in this family, CD-TVD is limited to volumes up to $1024^3$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for CD-TVD Upsampling
# Note: Specific implementation details of the Swin Transformer and Contrastive Encoder are abstracted as they are standard components.

def cd_tvd_upsample(lr_volume, pre_trained_model, single_hr_timestep):
    """
    Upsamples a low-resolution volume using CD-TVD.
    
    Args:
        lr_volume: Low-resolution 3D volume tensor.
        pre_trained_model: Model pre-trained on historical data with contrastive encoding.
        single_hr_timestep: Single high-resolution timestep for fine-tuning.
        
    Returns:
        hr_volume: High-resolution 3D volume tensor.
    """
    # Step 1: Select key timestep based on entropy
    # This step identifies the most informative timestep for adaptation
    key_timestep = select_key_timestep_entropy(single_hr_timestep)
    
    # Step 2: Fine-tune the pre-trained model
    # Adapt the model to the new simulation scenario using the selected timestep
    fine_tuned_model = fine_tune(pre_trained_model, key_timestep)
    
    # Step 3: Encode LR volume into latent space using contrastive encoder
    # The encoder learns degradation-aware features
    latent_lr = fine_tuned_model.contrastive_encoder(lr_volume)
    
    # Step 4: Apply diffusion process with Swin Transformer denoising
    # The denoising network iteratively refines the latent representation
    latent_hr = diffusion_denoise(latent_lr, fine_tuned_model.swin_denoiser)
    
    # Step 5: Decode latent HR volume back to spatial domain
    hr_volume = fine_tuned_model.decoder(latent_hr)
    
    return hr_volume

def select_key_timestep_entropy(hr_data):
    """
    Selects the key timestep based on entropy.
    Args:
        hr_data: High-resolution data (potentially a sequence of timesteps).
    Returns:
        Selected timestep.
    """
    # Calculate entropy for each timestep
    entropies = calculate_entropy(hr_data)
    # Select timestep with highest entropy (most informative)
    # Implementation detail: [G] Assuming entropy is calculated as a scalar metric per timestep
    key_idx = argmax(entropies)
    return hr_data[key_idx]

def calculate_entropy(data):
    """
    Calculates entropy for each timestep in the data.
    Args:
        data: 3D or 4D tensor of volume data.
    Returns:
        Array of entropy values.
    """
    # Implementation detail: [G] Entropy calculation depends on data distribution.
    # Common approach: Histogram-based entropy or differential entropy.
    # For simplicity, assume a standard entropy function is available.
    return [entropy(timestep) for timestep in data]
```

## Subtopics

### Gradient Vector Flow Diffusion with CNN Denoising
An alternative formulation that recovers grid-based vector fields from representative streamlines rather than direct grid upscaling [S1, unofficial]. It uses a two-stage approach: gradient vector flow diffusion followed by CNN-based denoising with recurrent residual blocks [S1, unofficial]. This method achieved 31–36 dB PSNR with compression rates of 12–51× on supercurrent, supernova, and tornado data [S1, unofficial]. It is distinct from CD-TVD in its input representation (streamlines vs. grids) and is chosen when data is available in streamline format rather than dense grids [S1, unofficial].

## Operating characteristics
*   **Performance:** Achieves 43–48 dB PSNR on four fluid dynamics datasets at 4× upscaling [S1, unofficial].
*   **Comparison:** Outperforms GAN-based methods in terms of PSNR [S1, unofficial].
*   **Data Efficiency:** Requires as few as one high-resolution timestep for fine-tuning, making it suitable for scenarios with limited HR data [S1, unofficial].
*   **Limitations:** Limited to volumes up to $1024^3$ [S1, unofficial]. Training is per-dataset, meaning models are not universally applicable across different simulation types without retraining or fine-tuning [S1, unofficial].
*   **Hardware:** Not specified in the sources [S1, unofficial].

## Background
CD-TVD was introduced as the first diffusion-based approach for scientific volume super-resolution [S1, unofficial]. It addresses the limitations of previous methods, such as GAN-based approaches, by leveraging the strengths of diffusion models and contrastive learning. The method builds on the trend of moving from single-modality CNNs to diffusion models in spatial scalar SR [S1, unofficial]. It was developed to handle the specific challenges of scientific volume data, such as the need for domain-specific representations and the scarcity of high-resolution simulation data [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
