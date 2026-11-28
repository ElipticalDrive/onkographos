# 3D Language Gaussian Splatting

Also called: LangSplat

## Overview
3D Language Gaussian Splatting (LangSplat) constructs a 3D language field using 3D Gaussians to enable precise, efficient open-vocabulary querying in 3D spaces [S1]. It addresses the imprecise object boundaries and slow rendering speeds of NeRF-based methods like LERF by leveraging tile-based splatting and hierarchical semantic supervision from SAM [S1]. Unlike its siblings that may use direct high-dimensional embedding storage or auxiliary DINO features, LangSplat employs a scene-specific autoencoder to compress CLIP embeddings into a low-dimensional latent space, significantly reducing memory overhead while maintaining multi-view consistency [S1].

## How it works
**Inputs:** A set of $T$ multi-view images $\{I_t\}_{t=1}^T$ of a 3D scene.
**Outputs:** A set of 3D language Gaussians $\mathcal{G}$, each parameterized by position $\mu$, covariance $\Sigma$, opacity $\alpha$, color $c$, and three latent language embeddings $\{f^s, f^p, f^w\}$ corresponding to subpart, part, and whole semantic levels. A trained scene-specific autoencoder (Encoder $E$, Decoder $\Psi$).

**Data Structures:**
1.  **3D Language Gaussian $G_i$:**
    *   $\mu_i \in \mathbb{R}^3$: Mean position.
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized via rotation $q$ and scale $s$).
    *   $\alpha_i \in [0, 1]$: Opacity.
    *   $c_i$: Color (RGB).
    *   $f^s_i, f^p_i, f^w_i \in \mathbb{R}^d$: Latent language embeddings for subpart, part, and whole levels. $d$ is the latent dimension.
2.  **Autoencoder:**
    *   Encoder $E: \mathbb{R}^D \to \mathbb{R}^d$.
    *   Decoder $\Psi: \mathbb{R}^d \to \mathbb{R}^D$.
    *   $D=512$ (CLIP embedding dimension), $d=3$ (latent dimension) [S1].

**Steps:**

1.  **Hierarchical Semantic Extraction:**
    For each input image $I_t$:
    a.  Feed a regular grid of $32 \times 32$ point prompts into the Segment Anything Model (SAM) to obtain initial mask sets $\{M^0_s, M^0_p, M^0_w\}$ for subpart, part, and whole levels [S1].
    b.  Filter redundant masks in each set based on predicted IoU score, stability score, and overlap rate [S1].
    c.  Perform comprehensive full-image segmentation for each filtered mask set to generate three segmentation maps: $M^s_t, M^p_t, M^w_t$ [S1].
    d.  Extract CLIP image embeddings for each segmented region using the CLIP image encoder $V$:
        $$ L^l_t(v) = V(I_t \odot M^l(v)), \quad l \in \{s, p, w\} $$
        where $M^l(v)$ is the mask region pixel $v$ belongs to at semantic level $l$, and $\odot$ denotes element-wise multiplication (masking) [S1].

2.  **Scene-Specific Autoencoder Training:**
    a.  Train a lightweight autoencoder to map high-dimensional CLIP features to a low-dimensional latent space.
    b.  The encoder $E$ maps $L^l_t(v) \in \mathbb{R}^D$ to $H^l_t(v) = E(L^l_t(v)) \in \mathbb{R}^d$ [S1].
    c.  The decoder $\Psi$ reconstructs the original embeddings from the latent representation.
    d.  Optimize the autoencoder using the loss:
        $$ \mathcal{L}_{ae} = \sum_{l \in \{s,p,w\}} \sum_{t=1}^T d_{ae}(\Psi(E(L^l_t(v))), L^l_t(v)) $$
        where $d_{ae}$ combines L1 distance and cosine distance loss [S1].
    e.  Transform all training CLIP embeddings $\{L^l_t\}$ into scene-specific latent features $\{H^l_t\}$ using the trained encoder $E$ [S1].

3.  **3D Language Gaussian Initialization and Optimization:**
    a.  Initialize 3D Gaussians (e.g., from SfM points). Augment each Gaussian $G_i$ with three latent language embeddings $\{f^s_i, f^p_i, f^w_i\}$ initialized from the projected latent features $H^l_t$ [S1].
    b.  Render the 3D language scene to 2D using tile-based rasterization. For a pixel $v$ and semantic level $l$, the rendered language embedding $F^l(v)$ is:
        $$ F^l(v) = \sum_{i \in \mathcal{N}} f^l_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $\mathcal{N}$ is the set of Gaussians in the tile, $\alpha_i$ is the opacity, and the product term represents transmittance [S1].
    c.  Optimize the Gaussian parameters (including $f^l_i$) by minimizing the language loss:
        $$ \mathcal{L}_{lang} = \sum_{l \in \{s,p,w\}} \sum_{t=1}^T d_{lang}(F^l_t(v), H^l_t(v)) $$
        where $d_{lang}$ is a distance function (typically cosine or L1) between the rendered latent feature and the target latent feature $H^l_t(v)$ [S1].
    d.  Jointly optimize with photometric reconstruction losses (L1/SSIM on color $c$) to ensure visual fidelity [S1, 15930-semantic-and-language-integration.md].

4.  **Inference and Querying:**
    a.  Render the language embeddings $F^l$ from the optimized 3D Gaussians to 2D.
    b.  Decode the rendered latent features back to CLIP space using the trained decoder: $\hat{L}^l = \Psi(F^l) \in \mathbb{R}^D$ [S1].
    c.  For a text query, compute the CLIP text embedding $\phi_{qry}$.
    d.  Compute the relevancy score for each pixel/point. Let $\phi_{img}$ be the decoded CLIP embedding and $\phi_{canon}$ be the CLIP embedding of a predefined canonical phrase (e.g., "object", "things", "stuff", "texture") [S1]. The relevancy score is:
        $$ \text{score} = \min_i \frac{\exp(\phi_{img} \cdot \phi_{qry})}{\exp(\phi_{img} \cdot \phi_{qry}) + \exp(\phi_{img} \cdot \phi_{canon})} $$
        (Note: The source formula uses a min operator over indices $i$, likely referring to a normalization or specific scoring mechanism described in the appendix, but the core cosine similarity structure is explicit) [S1].
    e.  Select the semantic level ($s, p, w$) that yields the highest relevancy score for the query [S1].
    f.  **3D Object Localization:** Choose the 3D point with the highest relevance score [S1].
    g.  **3D Semantic Segmentation:** Filter out points with relevancy scores lower than a chosen threshold and predict object masks with the remaining regions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class LangSplat:
    def __init__(self, d_latent=3, d_clip=512):
        self.d_latent = d_latent
        self.d_clip = d_clip
        # Scene-specific autoencoder
        self.encoder = nn.Sequential(
            nn.Linear(d_clip, 256),
            nn.ReLU(),
            nn.Linear(256, d_latent)
        )
        self.decoder = nn.Sequential(
            nn.Linear(d_latent, 256),
            nn.ReLU(),
            nn.Linear(256, d_clip)
        )
        # 3D Gaussians: stored as tensors
        # mu: (N, 3), cov: (N, 3, 3) or q,s,
        # alpha: (N,), color: (N, 3)
        # lang_features: (N, 3, d_latent) -> [f_s, f_p, f_w]
        self.gaussians = None 
        self.is_trained = False

    def train_autoencoder(self, clip_embeddings):
        """
        clip_embeddings: (T, H, W, 3, D) or flattened (N_masks, D)
        """
        optimizer = torch.optim.Adam(list(self.encoder.parameters()) + 
                                     list(self.decoder.parameters()), lr=1e-3)
        
        # Flatten for training
        flat_embs = clip_embeddings.view(-1, self.d_clip)
        
        for epoch in range(100): # Placeholder for convergence
            latent = self.encoder(flat_embs)
            recon = self.decoder(latent)
            
            l1_loss = F.l1_loss(recon, flat_embs)
            # Cosine distance
            cos_sim = F.cosine_similarity(recon, flat_embs, dim=1)
            cosine_loss = 1 - cos_sim.mean()
            
            loss = l1_loss + cosine_loss
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        self.is_autoencoder_trained = True

    def get_target_latents(self, clip_embeddings):
        """Transform CLIP embeddings to latent space using trained encoder."""
        if not self.is_autoencoder_trained:
            raise Exception("Autoencoder not trained")
        flat_embs = clip_embeddings.view(-1, self.d_clip)
        return self.encoder(flat_embs).view_as(clip_embeddings)

    def render_language(self, gaussians, camera_params):
        """
        Render language features using tile-based splatting.
        Returns rendered latent features F^l for each level l.
        """
        # Pseudocode for standard 3DGS rasterization adapted for language features
        # 1. Project 3D Gaussians to 2D tiles
        # 2. Sort Gaussians by depth within each tile
        # 3. Alpha blend language features
        
        # F^l(v) = sum_i f_i^l * alpha_i * prod_{j<i} (1 - alpha_j)
        # This is handled by the rasterizer. 
        # Here we assume a function `rasterize` that returns (H, W, d_latent) for each level
        pass 

    def optimize_gaussians(self, images, masks, camera_params, target_latents):
        """
        images: (T, 3, H, W)
        masks: (T, 3, H, W, 3) -> [M_s, M_p, M_w] boolean masks
        target_latents: (T, 3, H, W, d_latent) -> [H_s, H_p, H_w]
        """
        optimizer = torch.optim.Adam(self.gaussians.parameters(), lr=1e-4)
        
        for step in range(10000):
            # Forward pass
            rendered_latents = self.render_language(self.gaussians, camera_params)
            
            # Loss calculation
            # L_lang = sum_l sum_t d_lang(F^l_t, H^l_t)
            lang_loss = 0
            for l in range(3): # s, p, w
                # Masked loss or full image loss as per source
                # Source implies supervision using SAM-based masks
                mask = masks[:, l] 
                rendered = rendered_latents[:, l]
                target = target_latents[:, l]
                
                # Apply mask
                if mask.any():
                    lang_loss += F.mse_loss(rendered[mask], target[mask])
            
            # Photometric loss (L1/SSIM on color)
            rendered_colors = self.render_color(self.gaussians, camera_params)
            photo_loss = F.l1_loss(rendered_colors, images)
            
            total_loss = lang_loss + photo_loss
            
            optimizer.zero_grad()
            total_loss.backward()
            optimizer.step()
            
            # Densification/Pruning steps would go here (standard 3DGS)

    def query(self, text_query, rendered_latents_3d):
        """
        text_query: string
        rendered_latents_3d: (N, 3, d_latent) -> latent features for each 3D point
        Returns: relevancy scores and best semantic level
        """
        # Encode text query
        text_emb = clip_text_encoder(text_query) # (D,)
        
        # Canonical phrases
        canonical_phrases = ["object", "things", "stuff", "texture"]
        canon_embs = [clip_text_encoder(p) for p in canonical_phrases]
        
        # Decode 3D latents to CLIP space
        decoded_latents = self.decoder(rendered_latents_3d) # (N, 3, D)
        
        scores = []
        for l in range(3):
            point_embs = decoded_latents[:, l] # (N, D)
            # Cosine similarity with query
            sim_query = F.cosine_similarity(point_embs, text_emb, dim=1) # (N,)
            
            # Relevancy score calculation
            # score = min_i (exp(sim) / (exp(sim) + exp(sim_canon)))
            # Note: Source formula is slightly ambiguous on 'min_i'. 
            # Assuming it refers to normalization against canonical phrases.
            # We compute similarity against each canonical phrase
            canon_sims = torch.stack([F.cosine_similarity(point_embs, c, dim=1) for c in canon_embs]) # (4, N)
            
            # Numerator
            num = torch.exp(sim_query)
            # Denominator: sum over canonical phrases? Or min? 
            # Source: "min_i exp(...) / (exp(...) + exp(...))"
            # Let's implement the explicit formula structure:
            # For each point, we compare query vs canonical.
            # The source says "min_i" where i likely indexes the canonical phrases or a specific normalization.
            # Given "chosen from 'object', 'things'...", we take the max canonical similarity as the denominator baseline?
            # Or strictly follow the text: min over i of the ratio.
            
            ratios = []
            for c_sim in canon_sims:
                ratio = torch.exp(sim_query) / (torch.exp(sim_query) + torch.exp(c_sim))
                ratios.append(ratio)
            
            # Min over canonical phrases
            level_score = torch.min(torch.stack(ratios), dim=0)[0]
            scores.append(level_score)
            
        # Select best level
        scores_tensor = torch.stack(scores) # (3, N)
        best_level_idx = torch.argmax(scores_tensor, dim=0) # (N,)
        best_scores = torch.gather(scores_tensor, 0, best_level_idx.unsqueeze(0)).squeeze(0) # (N,)
        
        return best_scores, best_level_idx
```

## Operating characteristics
*   **Memory:** Significantly reduced compared to direct CLIP storage. Storing 512-dimensional CLIP features increases memory requirements by over 35 times compared to RGB colors (without spherical harmonics), leading to L1 cache exhaustion [S1]. LangSplat uses a latent dimension $d=3$, drastically lowering this cost [S1].
*   **Speed:** 199x faster than LERF at 1440x1080 resolution [S1]. This efficiency comes from tile-based splatting instead of volume rendering [S1].
*   **Accuracy:** 
    *   Localization accuracy on LERF dataset: LangSplat 84.3% overall vs LERF 73.6% vs LSeg 21.1% [S1].
    *   3D semantic segmentation IoU on LERF dataset: LangSplat 51.4% overall vs LERF 37.4% vs LSeg 16.6% [S1].
*   **Strengths:** Precise object boundaries, efficient rendering, open-vocabulary querying, no need for DINO regularization [S1].
*   **Limitations:** Requires SAM for initial mask generation; performance depends on the quality of the autoencoder reconstruction; limited to predefined semantic levels (subpart, part, whole) during training [S1].

## Background
LangSplat was introduced in 2024 by Wang et al. [S1]. It builds on the principles of 3D Gaussian Splatting and addresses limitations in NeRF-based language fields like LERF, which suffer from imprecise features and slow rendering [S1]. It incorporates hierarchical semantics from SAM and uses a scene-specific autoencoder to compress CLIP embeddings [S1].

## Sources
[S1] LangSplat: 3D Language Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.01895, fetched 2026-10-26
