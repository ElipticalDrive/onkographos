# Latent Space Representation

Also called: Latent Scene Representation, Latent Gaussian Representation

## Overview
Latent Space Representation methods map input images to a compact, structured latent code (tokens, features, or distributions) which is subsequently decoded into explicit 3D Gaussian primitives, enabling generalizable reconstruction without per-scene optimization [16746-feed-forward-generalizable-reconstruction.md]. Unlike sibling methods that predict Gaussians directly from pixel-wise features or cost volumes, this family explicitly models the scene as a set of learnable latent entities (e.g., tokens or entropy-coded features) to enforce global consistency, reduce representation bloat, or handle ambiguity [16746-feed-forward-generalizable-reconstruction.md].

## Shared principles
All methods in this family share the core objective of mapping input image features to a set of 3D Gaussian parameters $\mathcal{G} = \{\mu, \Sigma, \alpha, c\}$ without per-scene gradient descent [16746-feed-forward-generalizable-reconstruction.md].

1. **Input Representation:** Inputs are typically one or more RGB images $I \in \mathbb{R}^{H \times W \times 3}$ [16746-feed-forward-generalizable-reconstruction.md]. Some methods require known camera poses $P$, while others (e.g., [Splatt3R](16423-splatt3r.md), [ZeroGS](16425-zerogs.md)) predict poses or operate on uncalibrated inputs [16746-feed-forward-generalizable-reconstruction.md].
2. **Feature Encoding:** A shared encoder (e.g., Vision Transformer, CNN) extracts multi-scale features from the input images [16746-feed-forward-generalizable-reconstruction.md]. Cross-attention or cost volumes are often used to aggregate multi-view information [16746-feed-forward-generalizable-reconstruction.md].
3. **Primitive Prediction:** A decoder head predicts Gaussian attributes [16746-feed-forward-generalizable-reconstruction.md]. These attributes are usually:
    * **Mean ($\mu$):** 3D position.
    * **Covariance ($\Sigma$):** Defined by scale $s$ and rotation $r$ (quaternion or 6D).
    * **Opacity ($\alpha$):** Scalar.
    * **Color ($c$):** Spherical Harmonics (SH) coefficients or constant RGB [16746-feed-forward-generalizable-reconstruction.md].
4. **Differentiable Rendering:** The predicted Gaussians are rendered into target views using differentiable rasterization to compute reconstruction losses (e.g., L1, SSIM, LPIPS) for training the network [16746-feed-forward-generalizable-reconstruction.md].
5. **Generalization:** The network is trained on a large dataset of scenes to learn priors about geometry and appearance, allowing inference on unseen scenes [16746-feed-forward-generalizable-reconstruction.md].

## Subtopics

### [GlobalSplat](16571-globalsplat.md)
GlobalSplat is a feed-forward 3D Gaussian Splatting framework that aligns multi-view inputs into a fixed set of global latent scene tokens before decoding explicit 3D geometry, preventing representation bloat and ensuring global consistency [16571-globalsplat.md]. Unlike sibling methods such as [CodecSplat](18143-codecsplat.md) and [latentSplat](18652-latentsplat.md) which may rely on dense view-aligned primitives or different grid structures, GlobalSplat aggregates multi-view inputs into a fixed number of latent tokens (2048) regardless of the number of input views, then decodes these tokens into a sparse, globally consistent set of Gaussians (16K) [16571-globalsplat.md]. This approach achieves a superior quality-efficiency trade-off compared to [C3G](16423-c3g.md), which uses full self-attention and single-Gaussian decoding, by employing an iterative, disentangled dual-branch architecture and a coarse-to-fine capacity curriculum [16571-globalsplat.md].

**Algorithm:**
1. **Initialization:** Initialize $M=2048$ latent tokens $l_j^{(0)}$ and register tokens.
2. **Dual-Branch Encoder:** Repeat for $B=4$ blocks $j=0 \dots B-1$: Project latent tokens into geometry and appearance streams, apply cross-attention with input patches, apply self-attention blocks, and fuse streams via MLP.
3. **Decoder Candidate Prediction:** For each latent token $p$, predict $K_s=16$ candidates for means, log-scales, 6D rotations, opacity logits, importance scores, and SH coefficients.
4. **Coarse-to-Fine Merging:** Partition 16 candidates into $G=8$ groups. Compute softmax weights using importance scores and temperature $\tau$. Merge attributes via weighted sum.
5. **Rendering:** Project Gaussians to 2D and accumulate color using standard alpha blending.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GlobalSplat:
    def __init__(self, num_tokens=2048, token_dim=512, num_candidates=16, num_encoder_blocks=4, num_self_attn_layers=2):
        self.M = num_tokens
        self.d = token_dim
        self.K = num_candidates
        self.B = num_encoder_blocks
        self.L = num_self_attn_layers
        self.G_final = 8
        self.temperature = 1.0
        
        self.latent_tokens = nn.Parameter(torch.randn(self.M, self.d))
        self.register_tokens = nn.Parameter(torch.randn(self.M, self.d))
        
        self.proj_geo = nn.Linear(self.d, self.d)
        self.proj_app = nn.Linear(self.d, self.d)
        self.cross_attn_geo = nn.MultiheadAttention(self.d, num_heads=8)
        self.cross_attn_app = nn.MultiheadAttention(self.d, num_heads=8)
        self.self_attn_geo = nn.ModuleList([nn.MultiheadAttention(self.d, num_heads=8) for _ in range(self.L)])
        self.self_attn_app = nn.ModuleList([nn.MultiheadAttention(self.d, num_heads=8) for _ in range(self.L)])
        self.mixer_mlp = nn.Sequential(nn.Linear(self.d * 2, self.d), nn.ReLU(), nn.Linear(self.d, self.d))
        
        self.dec_geo = nn.Linear(self.d, self.K * 14)
        self.dec_app = nn.Linear(self.d, self.K * 48)
        self.W_ray = nn.Linear(6, 256)

    def normalize_scene(self, poses, intrinsics):
        centers = poses[:, :3, 3]
        o_avg = centers.mean(dim=0)
        views = poses[:, :3, 2]
        view_avg = views.mean(dim=0)
        view_avg = view_avg / view_avg.norm()
        x_axis = torch.cross(view_avg, torch.tensor([0, 0, 1], device=views.device))
        if x_axis.norm() < 1e-6: x_axis = torch.tensor([1, 0, 0], device=views.device)
        x_axis = x_axis / x_axis.norm()
        y_axis = torch.cross(view_avg, x_axis)
        T_avg_inv = torch.eye(4, device=poses.device)
        T_avg_inv[:3, :3] = torch.stack([x_axis, y_axis, view_avg], dim=1)
        T_avg_inv[:3, 3] = -T_avg_inv[:3, :3] @ o_avg
        
        transformed_centers = (T_avg_inv[:3, :3] @ centers.T).T + T_avg_inv[:3, 3]
        max_dist = 0
        for i in range(len(centers)):
            for j in range(i + 1, len(centers)):
                dist = (transformed_centers[i] - transformed_centers[j]).norm()
                if dist > max_dist: max_dist = dist
        scale = max_dist if max_dist > 0 else 1.0
        scaled_centers = transformed_centers / scale
        return scaled_centers, scale

    def extract_features(self, images, poses, intrinsics):
        C, _, H, W = images.shape
        patch_size = 8
        num_patches = (H // patch_size) * (W // patch_size)
        u_rgb = self.vision_encoder(images)
        r = compute_plucker_rays(poses, intrinsics, H, W, patch_size)
        r_flat = r.view(-1, 6)
        r_mapped = self.W_ray(r_flat).view(C, num_patches, 256)
        e = self.camera_encoder(poses, intrinsics)
        u_cam = r_mapped + e.unsqueeze(1)
        u = torch.cat([u_cam, u_rgb], dim=-1)
        return u

    def encode(self, u):
        x = u.permute(1, 0, 2)
        l = self.latent_tokens.unsqueeze(1)
        for j in range(self.B):
            f_geo = self.proj_geo(l).permute(1, 0, 2)
            f_app = self.proj_app(l).permute(1, 0, 2)
            f_geo_cross, _ = self.cross_attn_geo(f_geo, x, x)
            f_app_cross, _ = self.cross_attn_app(f_app, x, x)
            for l_idx in range(self.L):
                f_geo, _ = self.self_attn_geo[l_idx](f_geo_cross, f_geo_cross, f_geo_cross)
                f_app, _ = self.self_attn_app[l_idx](f_app_cross, f_app_cross, f_app_cross)
            fused = torch.cat([f_geo, f_app], dim=-1)
            l = self.mixer_mlp(fused).permute(1, 0, 2)
        return l.squeeze(1)

    def decode(self, l):
        geo_out = self.dec_geo(l).view(-1, self.K, 14)
        app_out = self.dec_app(l).view(-1, self.K, 48)
        X = geo_out[:, :, :3]
        S = geo_out[:, :, 3:6]
        R = geo_out[:, :, 6:12]
        O = geo_out[:, :, 12:13]
        l_gate = geo_out[:, :, 13:14]
        return X, S, R, O, l_gate, app_out

    def merge_candidates(self, X, S, R, O, l_gate, C, stage_s=3):
        G = 2 ** stage_s
        b_s = self.K // G
        merged_X, merged_S, merged_R, merged_O, merged_C = [], [], [], [], []
        for p in range(X.shape[0]):
            for g in range(G):
                start_idx, end_idx = g * b_s, (g + 1) * b_s
                X_g, S_g, R_g, O_g, C_g = X[p, start_idx:end_idx], S[p, start_idx:end_idx], R[p, start_idx:end_idx], O[p, start_idx:end_idx], C[p, start_idx:end_idx]
                l_g = l_gate[p, start_idx:end_idx]
                weights = F.softmax(l_g / self.temperature, dim=0).squeeze(-1)
                merged_X.append(torch.sum(weights.unsqueeze(-1) * X_g, dim=0))
                merged_S.append(torch.sum(weights.unsqueeze(-1) * S_g, dim=0))
                merged_R.append(torch.sum(weights.unsqueeze(-1) * R_g, dim=0))
                merged_O.append(torch.sum(weights.unsqueeze(-1) * O_g, dim=0))
                merged_C.append(torch.sum(weights.unsqueeze(-1) * C_g, dim=0))
        return torch.stack(merged_X), torch.stack(merged_S), torch.stack(merged_R), torch.stack(merged_O), torch.stack(merged_C)

    def forward(self, images, poses, intrinsics):
        centers, scale = self.normalize_scene(poses, intrinsics)
        u = self.extract_features(images, poses, intrinsics)
        l = self.encode(u)
        X, S, R, O, l_gate, C = self.decode(l)
        return self.merge_candidates(X, S, R, O, l_gate, C, stage_s=3)
```

### [CodecSplat](18143-codecsplat.md)
CodecSplat is a compression framework for feed-forward 3D Gaussian Splatting (FF3DGS) that encodes an intermediate 2D Gaussian-generation feature into an entropy-coded scene bitstream, avoiding the inefficient compression of irregular 3D Gaussian primitives [18143-codecsplat.md]. Unlike existing 3DGS compression methods that compress Gaussian primitives after a structured-to-irregular transformation, CodecSplat places the coding bottleneck before 2D-to-3D Gaussian construction, allowing compression to exploit the structured intermediate feature representation [18143-codecsplat.md].

**Algorithm:**
1. **Multi-view Feature Extraction:** Extract multi-view features $F_{mv}$, encoder-side depth $D_{enc}$, and depth matching-probability features $P_{depth}$ via a multi-view encoder backbone.
2. **Gaussian-Generation Feature Construction:** Convert cues into a dense Gaussian-generation feature $F_g$ via a feature head network.
3. **Latent Encoding:** Transform $F_g$ into latents via analysis transform, quantize with rate control $\lambda$, entropy-code, and pack with compressed camera parameters into bitstream $B$.
4. **Latent Decoding:** Entropy-decode $B$, reconstruct feature map via synthesis transform, and recover camera metadata.
5. **Gaussian Reconstruction:** Predict depth, refine features, predict Gaussian parameters, and map to 3D primitives.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def codec_splat_encode(context_views):
    F_mv, D_enc, P_depth = MVEnc(context_views)
    F_g = FeatHead([v[0] for v in context_views], F_mv, P_depth)
    latent = analysis_transform(F_g)
    quantized_latent = quantize(latent, rate_param=0.1)
    bitstream_latents = entropy_encode(quantized_latent)
    compressed_cameras = compress_cameras([v[1:] for v in context_views])
    bitstream = pack(bitstream_latents, compressed_cameras)
    return bitstream

def codec_splat_decode(bitstream):
    bitstream_latents, compressed_cameras = unpack(bitstream)
    quantized_latent = entropy_decode(bitstream_latents)
    F_g_decoded = synthesis_transform(quantized_latent)
    cameras = decompress_cameras(compressed_cameras)
    depth_pred, refined_features = depth_and_refine(F_g_decoded)
    gaussians = predict_gaussians(refined_features, depth_pred, cameras)
    return gaussians

# Helper stubs for standard codec operations not detailed in the source
def analysis_transform(x): return x
def quantize(x, rate_param): return x
def entropy_encode(x): return b''
def compress_cameras(cameras): return b''
def pack(lats, cams): return b''
def unpack(bs): return b'', b''
def entropy_decode(bs): return None
def synthesis_transform(x): return x
def decompress_cameras(bs): return []
def depth_and_refine(x): return None, None
def predict_gaussians(feat, depth, cams): return None
```

### [latentSplat](18652-latentsplat.md)
latentSplat is a method for generalizable 3D reconstruction from two reference views that predicts semantic 3D Gaussians in a latent space [18652-latentsplat.md]. It models uncertainty explicitly by storing distributions (mean and variance) of spherical harmonic coefficients for each Gaussian, allowing sampling of specific instances for rendering [18652-latentsplat.md]. Unlike regression-based methods such as [pixelSplat](16424-pixelsplat.md) which provide a single low-variance solution, latentSplat handles high-ambiguity regions and view extrapolation better, avoiding blurry reconstructions in unobserved areas [18652-latentsplat.md]. Compared to generative diffusion-based methods like [GeNVS](16426-genvs.md), it is orders of magnitude faster and scales to high resolutions by using efficient Gaussian splatting for rendering instead of volume rendering or iterative diffusion sampling [18652-latentsplat.md].

**Algorithm:**
1. **Feature Encoding:** Encode two input views using a DINOv1 ViT-B/8 backbone, upscale patch tokens, add class token, and downscale by factor 4.
2. **Epipolar Aggregation:** Process 32 samples of each pixel's epipolar line using an Epipolar Transformer (2 blocks, 4 heads, dim 128). Add skip connection at original resolution.
3. **Gaussian Parameter Prediction:** Predict variational Gaussian parameters via a sampling head (normalized quaternions for rotation, scaled/shifted sigmoid for scale/opacity).
4. **Sampling:** Sample semantic feature coefficients $\mathbf{h}_i \sim \mathcal{N}(\mathbf{h}_{\mu,i}, \mathbf{h}_{\sigma,i})$ using reparameterization trick.
5. **Rendering & Decoding:** Render sampled Gaussians to target view via Gaussian splatting. Decode features into pixel space using a VAE-GAN decoder with zero-initialized skip connections.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class latentSplat:
    def __init__(self, N, token_dim=128):
        self.N = N
        self.vit_backbone = DINOv1_ViT_B_8()
        self.epipolar_transformer = EpipolarTransformer(num_blocks=2, num_heads=4, token_dim=token_dim)
        self.gaussian_head = GaussianSamplingHead()
        self.decoder = VAE_GAN_Decoder()
        self.skip_connections = nn.ModuleList([nn.Conv2d(token_dim, token_dim, 1) for _ in range(4)])
        for m in self.skip_connections: nn.init.zeros_(m.weight)
        self.discriminator = Discriminator()

    def encode(self, ref_views, poses):
        vit_features = self.vit_backbone(ref_views)
        feat_map = self._upscale_and_add_class_token(vit_features)
        feat_map_down = F.interpolate(feat_map, scale_factor=0.25, mode='bilinear')
        epipolar_features = self.epipolar_transformer(feat_map_down, poses, num_samples=32)
        epipolar_features = self._merge_skip_connection(epipolar_features, feat_map)
        gaussians = self.gaussian_head(epipolar_features)
        return gaussians

    def sample_gaussians(self, gaussians):
        h_mu = gaussians['h_mu']
        h_sigma = gaussians['h_sigma']
        eps = torch.randn_like(h_mu)
        h_sampled = h_mu + h_sigma * eps
        sampled_gaussians = {
            'x': gaussians['x'], 'S': gaussians['S'], 'R': gaussians['R'],
            'o': gaussians['o'], 'c': gaussians['c'], 'h': h_sampled
        }
        return sampled_gaussians

    def render(self, gaussians, target_pose):
        rgb_render, feat_render = gaussian_splatting(gaussians, target_pose)
        return rgb_render, feat_render

    def decode(self, rgb_render, feat_render):
        skip_inputs = [self.skip_connections[i](rgb_render) for i in range(4)]
        output = self.decoder(feat_render, skip_inputs)
        return output

    def forward(self, ref_views, poses, target_pose):
        gaussians = self.encode(ref_views, poses)
        sampled_gaussians = self.sample_gaussians(gaussians)
        rgb_render, feat_render = self.render(sampled_gaussians, target_pose)
        output = self.decode(rgb_render, feat_render)
        return output, rgb_render, feat_render

    def _upscale_and_add_class_token(self, vit_features): pass
    def _merge_skip_connection(self, epipolar_features, original_feat): pass

class DINOv1_ViT_B_8(nn.Module): pass
class EpipolarTransformer(nn.Module): pass
class GaussianSamplingHead(nn.Module): pass
class VAE_GAN_Decoder(nn.Module): pass
class Discriminator(nn.Module): pass
```

## Background
This approach emerged from the need to scale Gaussian Splatting to large contexts and generalizable settings where per-scene optimization is infeasible [G]. By decoupling feature aggregation from primitive instantiation via a latent space, methods like GlobalSplat and latentSplat address the "representation bloat" and local consistency issues of direct pixel-wise regression [16571-globalsplat.md, 18652-latentsplat.md]. CodecSplat extends this by addressing the storage and transmission inefficiencies of irregular Gaussian primitives [18143-codecsplat.md].

