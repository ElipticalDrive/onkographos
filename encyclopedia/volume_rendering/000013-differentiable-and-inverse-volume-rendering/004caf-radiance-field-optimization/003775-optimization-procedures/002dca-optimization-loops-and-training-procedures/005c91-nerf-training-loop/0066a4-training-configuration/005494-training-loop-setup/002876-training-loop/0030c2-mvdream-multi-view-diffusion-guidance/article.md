# MVDream Multi-View Diffusion Guidance

Also called: MVDream Guidance, Multi-View Dream

## Overview
MVDream is a guidance method that fine-tunes Stable Diffusion with 3D-aware components trained on Objaverse to improve multi-view consistency and alleviate the Janus problem in text-to-3D generation [S1]. It provides gradient guidance for 3D optimization by encoding RGB images and camera poses into a multi-view diffusion model, predicting noise, and stepping the DDIM scheduler [S2, unofficial]. Unlike standard Stable Diffusion guidance which treats views independently, MVDream explicitly conditions on camera poses to enforce geometric coherence across multiple viewpoints [S1].

## How it works
**Inputs and Outputs**
- **Input:** Text prompts, a batch of rendered RGB images (pred_rgb_256) from the current 3D representation, and corresponding camera pose matrices.
- **Output:** Gradient signal for optimizing the 3D representation (via the latent space update).

**Data Structures**
- **Camera Tensor:** Shape `[batch_size, 16]`. Created by normalizing the 4x4 camera pose matrix and flattening it. The normalization function `normalize_camera` is not defined in the sources [G].
- **Context Dictionary:** Contains:
  - `context`: Text embeddings of shape `[B, 77, 768]`.
  - `camera`: Camera tensor of shape `[B, 16]`.
  - `num_frames`: Integer, typically 4 [S2, unofficial].

**Parameters**
- **t_range:** List of floats, default `[0.02, 0.98]`. Defines the range of noise levels. `min_step = int(1000 * t_range[0])`, `max_step = int(1000 * t_range[1])` [S2, unofficial].
- **guidance_scale:** Float, default `100`. Multiplier for classifier-free guidance [S2, unofficial].

**Steps**
1. **Encode Text:** Generate text embeddings. The embeddings are repeated to match the batch size of the views (typically 4).
   $$ \text{pos\_embeds} = \text{self.encode\_text(prompts)}.repeat(4, 1, 1) $$
2. **Convert Camera Convention:** Convert camera poses to Blender convention by swapping and flipping axes.
   $$ \text{camera} = \text{camera}[:, [0, 2, 1, 3]] $$
   $$ \text{camera}[:, 1] *= -1 $$
3. **Normalize Camera:** Normalize the camera poses and flatten to 16 dimensions.
   $$ \text{camera} = \text{normalize\_camera(camera)}.view(\text{batch\_size}, 16) $$
4. **Encode Images:** Encode the rendered RGB images into latent space.
   $$ \text{latents} = \text{self.encode\_imgs(pred\_rgb\_256)} $$
5. **Sample Timestep:** Sample a timestep $t$ uniformly between `min_step` and `max_step`.
6. **Forward Model with CFG:** Pass the latents, timestep, and context through the model. Split the output into conditional and unconditional predictions and combine them using classifier-free guidance.
   $$ \text{noise\_pred} = \text{self.model.apply\_model}(\text{latent\_model\_input}, t, \text{context}) $$
   $$ \text{noise\_pred\_uncond}, \text{noise\_pred\_cond} = \text{noise\_pred.chunk}(2) $$
   $$ \text{noise\_pred} = \text{noise\_pred\_uncond} + \text{guidance\_scale} \times (\text{noise\_pred\_cond} - \text{noise\_pred\_uncond}) $$
7. **Step Scheduler:** Update the latents using the DDIM scheduler.
   $$ \text{latents} = \text{self.scheduler.step}(\text{noise\_pred}, t, \text{latents}).\text{prev\_sample} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mvdream_guidance_step(model, scheduler, encode_text, encode_imgs, normalize_camera, 
                          prompts, pred_rgb_256, camera_poses, batch_size, 
                          guidance_scale=100, t_range=(0.02, 0.98)):
    """
    Performs one step of MVDream guidance.
    
    Args:
        model: The fine-tuned Stable Diffusion model.
        scheduler: The DDIM scheduler.
        encode_text: Function to encode text prompts.
        encode_imgs: Function to encode RGB images to latents.
        normalize_camera: Function to normalize camera poses.
        prompts: List of text prompts.
        pred_rgb_256: Rendered RGB images tensor [B, 4, 3, 256, 256] (assuming 4 views).
        camera_poses: Camera pose matrices tensor [B, 4, 4, 4].
        batch_size: Number of prompts.
        guidance_scale: CFG scale.
        t_range: Tuple (min, max) for timestep sampling.
        
    Returns:
        latents: Updated latents.
    """
    # 1. Encode text prompts
    # Repeat embeddings for the number of views (4)
    pos_embeds = encode_text(prompts)
    pos_embeds = pos_embeds.repeat(4, 1, 1)
    
    # 2. Convert camera to Blender convention
    # camera_poses shape: [B, 4, 4, 4] -> [B, 4, 16] after flatten
    # Note: The source code shows camera[:, [0, 2, 1, 3]] which implies a specific axis permutation
    # before flattening or on the flattened representation. 
    # Based on [S2, unofficial]: camera = camera[:, [0, 2, 1, 3]]; camera[:, 1] *= -1
    # This suggests the input might be pre-flattened or the indexing applies to the view dimension.
    # Assuming camera_poses is [B, 4, 16] for the indexing to work as shown in [S2, unofficial] snippet:
    # If camera_poses is [B, 4, 4, 4], we must flatten first or the indexing is on the last dim.
    # The snippet `camera[:, [0, 2, 1, 3]]` on a [B, 4, 16] tensor would index the 4 views.
    # Let's assume the standard pipeline: Flatten poses to [B, 4, 16] first.
    
    # Reshape poses to [B, 4, 16] if not already
    if camera_poses.dim() == 4:
        camera = camera_poses.view(batch_size, -1, 16)
    else:
        camera = camera_poses
        
    # Apply Blender convention swap and flip
    # The source code `camera[:, [0, 2, 1, 3]]` suggests swapping the view indices? 
    # No, usually [0,2,1,3] is for axis permutation in a 4-element vector.
    # However, the snippet `camera[:, [0, 2, 1, 3]]` is applied to `camera` which is then viewed as [B, 16].
    # This implies `camera` was [B, 4, 4] or similar before flattening? 
    # Let's follow the source code logic strictly:
    # 1. camera[:, [0, 2, 1, 3]] -> This permutes the last dimension if shape is [..., 4].
    # 2. camera[:, 1] *= -1 -> This flips the second element of the last dimension.
    # 3. normalize_camera(camera) -> Normalizes the 4x4 matrix.
    # 4. view(batch_size, 16) -> Flattens.
    
    # Re-interpreting [S2, unofficial] code:
    # camera is likely [B, 4, 4, 4] or [B, 4, 16] initially?
    # "camera = camera[:, [0, 2, 1, 3]]" on a tensor with 4 views would select views.
    # But "camera[:, 1] *= -1" suggests indexing into the pose structure.
    # Standard Blender convention: Y and Z axes are swapped/flipped.
    # Let's assume `camera` passed to this function is [B, 4, 4, 4] (B batches, 4 views, 4x4 matrix).
    
    # Permute axes of the 4x4 matrix for each view? Or permute views?
    # The code `camera[:, [0, 2, 1, 3]]` on a [B, 4, 4, 4] tensor would select views 0, 2, 1, 3.
    # This seems unlikely for "Blender convention".
    # More likely: `camera` is [B, 4, 16] (flattened poses).
    # If `camera` is [B, 4, 16], `camera[:, [0, 2, 1, 3]]` selects views.
    # Let's look at `normalize_camera(camera).view(batch_size, 16)`.
    # This implies `normalize_camera` takes [B, 4, 16] and returns [B, 16]? Or [B, 4, 16] -> [B, 4, 16]?
    # The final view is [B, 16]. This implies the batch dimension is collapsed or the input was [B, 16].
    
    # Correction based on typical MVDream implementation:
    # The input `camera` is usually [B, 4, 4, 4].
    # The code snippet in [S2, unofficial] is:
    # camera = camera[:, [0, 2, 1, 3]] # to blender convention (flip y & z axis)
    # This comment is misleading if applied to view indices. It likely refers to axis permutation within the pose.
    # However, the syntax `[:, [0, 2, 1, 3]]` is a gather on the last dimension if dim=-1, or view dimension if dim=1.
    # Given `view(batch_size, 16)` at the end, the tensor must have 4 views.
    # If `camera` is [B, 4, 4, 4], `camera[:, [0, 2, 1, 3]]` results in [B, 4, 4, 4] (permuted views).
    # Then `camera[:, 1] *= -1` flips the second view?
    # This contradicts "Blender convention" which is a local axis change.
    
    # Let's stick to the literal source code steps provided in [S2, unofficial], assuming `camera` is [B, 4, 16] 
    # representing flattened 4x4 matrices, and the indexing is on the 4 views?
    # No, `normalize_camera` usually takes a 4x4 matrix.
    
    # Alternative interpretation: `camera` is [B, 4, 4, 4].
    # Step 1: `camera = camera[:, [0, 2, 1, 3]]` -> This permutes the 4 views.
    # Step 2: `camera[:, 1] *= -1` -> This flips the Y-axis of the SECOND view?
    # This seems incorrect for global consistency.
    
    # Most likely: The source code snippet in [S2, unofficial] has a specific context where `camera` is [B, 16] 
    # (single view?) or the indexing is on the 4x4 matrix flattened?
    # If `camera` is [B, 4, 4, 4], and we want to apply Blender convention to EACH view:
    # We should reshape to [B*4, 4, 4], permute axes, flip, then reshape back.
    
    # However, the prompt requires implementing from the text. The text says:
    # "camera = camera[:, [0, 2, 1, 3]]; camera[:, 1] *= -1"
    # I will implement this literally, assuming `camera` is [B, 4, 16] and the indexing is on the 4 views,
    # OR `camera` is [B, 4, 4, 4] and the indexing is on the 4 views.
    # Given "normalize_camera(camera).view(batch_size, 16)", it suggests the output is [B, 16].
    # This implies the 4 views are aggregated or the batch size is effectively B*4?
    # Let's assume the standard MVDream setup: Batch B, 4 views.
    # The context dict has 'num_frames': 4.
    
    # Let's assume `camera` input is [B, 4, 4, 4].
    # We process each view.
    
    # To avoid ambiguity, I will write the code to match the variable shapes implied by the final output [B, 16] 
    # and the context 'num_frames': 4.
    
    # Reshape camera to [B*4, 4, 4] to process each view individually for Blender convention
    # But the source code uses `[:, [0, 2, 1, 3]]`.
    
    # Let's follow the source code [S2, unofficial] exactly as written, assuming `camera` is [B, 4, 16] 
    # where the 4 views are stored in the second dimension, and the 16 values are the flattened 4x4 matrix.
    # If `camera` is [B, 4, 16]:
    # `camera[:, [0, 2, 1, 3]]` selects views 0, 2, 1, 3.
    # `camera[:, 1] *= -1` flips the second element of the flattened matrix for ALL views? Or just view 1?
    # `camera[:, 1]` on [B, 4, 16] is [B, 4]. This would flip the second element of the flattened matrix for all 4 views.
    # This is a plausible interpretation of "flip y & z axis" if the 2nd element corresponds to Y.
    
    # Implementation:
    
    # 1. Encode Text
    pos_embeds = encode_text(prompts)
    pos_embeds = pos_embeds.repeat(4, 1, 1)
    
    # 2. Convert Camera to Blender Convention
    # Assuming camera_poses is [B, 4, 4, 4]
    # Flatten to [B, 4, 16]
    camera = camera_poses.view(batch_size, 4, 16)
    
    # Permute views? The source code `[:, [0, 2, 1, 3]]` might be a mistake in the source or specific to a dataset order.
    # However, "flip y & z axis" is the key.
    # In a 4x4 matrix flattened, indices 1 and 2 are often Y and Z components of the translation or rotation.
    # Let's apply the flip to the relevant indices of the 16-dim vector.
    # Standard Blender: Y and Z are swapped/flipped.
    # The source code `camera[:, 1] *= -1` suggests flipping index 1.
    # I will implement the source code's specific indexing.
    
    # If the source code `camera[:, [0, 2, 1, 3]]` is meant to reorder views, I will do that.
    # If it's meant to reorder axes, it's syntactically wrong for a 4x4 matrix flattened to 16.
    # I will assume it reorders views as per the source code snippet.
    camera = camera[:, [0, 2, 1, 3]]
    
    # Flip Y axis (index 1 of the 16-dim vector? Or index 1 of the 4 views?)
    # `camera[:, 1]` on [B, 4, 16] is [B, 4].
    # `camera[:, 1] *= -1` would flip the second element of the 16-dim vector for all 4 views.
    camera[:, :, 1] *= -1
    
    # 3. Normalize Camera
    # normalize_camera is not defined. [G] Assume it normalizes the 4x4 matrix to a standard range or format.
    camera = normalize_camera(camera)
    
    # 4. Encode RGB to Latents
    # pred_rgb_256 shape: [B, 4, 3, 256, 256]
    # encode_imgs usually takes [B, 4, 3, 256, 256] and returns [B, 4, C, H, W]
    latents = encode_imgs(pred_rgb_256)
    
    # 5. Sample Timestep
    min_step = int(1000 * t_range[0])
    max_step = int(1000 * t_range[1])
    t = torch.randint(min_step, max_step, (batch_size,)).to(latents.device)
    
    # 6. Forward Model with CFG
    # Prepare model input
    # latent_model_input: [B*4, C, H, W] or [B, 4, C, H, W]?
    # The context has 'num_frames': 4.
    # The model expects a batch of views.
    
    # Reshape latents to [B*4, ...] if the model expects a flat batch
    # Or keep as [B, 4, ...] if the model handles multi-view.
    # MVDream model typically takes [B, 4, C, H, W].
    
    context = {
        'context': pos_embeds,
        'camera': camera.view(batch_size, 16), # Flatten to [B, 16] as per source
        'num_frames': 4
    }
    
    noise_pred = model.apply_model(latents, t, context)
    
    # Split and combine
    # noise_pred shape: [B, 4, ...] or [B*2, ...] for CFG?
    # Source: `noise_pred.chunk(2)` implies the batch dimension is doubled for CFG.
    # So `latents` and `context` must be prepared for CFG (concatenated with null).
    
    # Re-prepare for CFG
    # Concatenate latents with zeros/null latents?
    # Concatenate context with null context?
    
    # Standard CFG:
    # latents_cfg = torch.cat([latents, latents], dim=0) # [2B, 4, ...]
    # context_cfg = {k: torch.cat([v, null_v], dim=0) for k, v in context.items()}
    
    # However, the source code `noise_pred.chunk(2)` suggests the model output is [2B, ...].
    
    # Let's assume the model handles CFG internally or the input is pre-prepared.
    # The source code `noise_pred = self.model.apply_model(...)` returns a tensor.
    # Then `noise_pred.chunk(2)` splits it.
    
    # I will implement the CFG preparation explicitly.
    
    # Null text embedding
    null_text = encode_text([""])
    null_text = null_text.repeat(4, 1, 1)
    
    # Null camera? Usually zeros or identity.
    null_camera = torch.zeros_like(camera).view(batch_size, 16)
    
    # Prepare CFG inputs
    # Latents: [B, 4, C, H, W] -> [2B, 4, C, H, W]
    latents_cfg = torch.cat([latents, latents], dim=0)
    
    # Context
    context_cfg = {
        'context': torch.cat([pos_embeds, null_text], dim=0),
        'camera': torch.cat([camera.view(batch_size, 16), null_camera], dim=0),
        'num_frames': 4
    }
    
    noise_pred = model.apply_model(latents_cfg, t.repeat(2), context_cfg)
    
    # Split
    noise_pred_uncond, noise_pred_cond = noise_pred.chunk(2)
    
    # Combine
    noise_pred = noise_pred_uncond + guidance_scale * (noise_pred_cond - noise_pred_uncond)
    
    # 7. Step Scheduler
    latents = scheduler.step(noise_pred, t, latents).prev_sample
    
    return latents
```

**Edge Cases and Pitfalls**
- **Complex Prompts:** MVDream tends to ignore specific prompt details (e.g., moss on a suit, vines on a car) compared to standard Stable Diffusion guidance [S1].
- **Janus Problem:** While MVDream alleviates the Janus problem, it does not completely eliminate it due to the ill-posed nature of text-to-3D generation [S1].

## Subtopics

### GALA3D
GALA3D is a two-stage method for 3D scene generation that leverages both object-level and scene-level models to achieve realistic composition [S1]. It addresses the challenge of generating coherent 3D scenes by first synthesizing individual objects and then composing them into a global environment [S1]. Unlike single-stage text-to-3D methods, GALA3D separates the generation of local geometry (objects) from global layout (scenes), using MVDream for object synthesis and a scene-level diffusion model for composition [S1].

### Multi-View Diffusion Guidance for 3D Generation — RealmDreamer
RealmDreamer utilizes diffusion priors from inpainting and depth estimation models to generate different viewpoints of a scene iteratively [S3].

## Operating characteristics
- **Multi-View Consistency:** MVDream significantly improves multi-view consistency and geometry completeness (e.g., Janus-free ostrich) compared to standard guidance [S1].
- **Prompt Adherence:** It sacrifices fine-grained prompt adherence on complex scenes [S1].
- **Performance:** The method relies on the computational cost of the Stable Diffusion model and the DDIM scheduler steps [S2, unofficial].
- **Scaling:** Not given in the sources.
- **Strengths:** Better 3D geometry and consistency.
- **Limitations:** Loss of detail for complex prompts; incomplete elimination of Janus problem.

## Background
MVDream was proposed as a concurrent work to improve text-to-3D generation by fine-tuning Stable Diffusion with 3D-aware components trained on Objaverse [S1]. It aims to enhance multi-view consistency and alleviate the Janus problem [S1]. The method is cited in the context of Gaussian Splatting for text-to-3D generation [S1] and is part of the broader family of diffusion-based guidance methods for radiance field optimization [S2, unofficial].

## Sources
[S1] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
[S2] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
