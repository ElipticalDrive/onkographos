<div align="center">

# Multi-view Consistent 3D Gaussian Head Avatars 'without' Multi-view Generation
<div>
    <a href='https://aviralchharia.github.io/' target='_blank'>Aviral Chharia</a>&emsp;
    <a href='https://www.cs.cmu.edu/~ftorre/' target='_blank'>Fernando De la Torre</a>&emsp;
</div>
<div>
    Carnegie Mellon University&emsp;<br>
</div>
<div>
    <b>CVPR 2026</b>
</div>
<br>

  <a href="https://humansensinglab.github.io/MVCHead"><img alt="Webpage" src="https://img.shields.io/badge/Webpage-up-yellow"></a>
  [![GitHub Stars](https://img.shields.io/github/stars/humansensinglab/MVCHead?style=social)](https://github.com/humansensinglab/MVCHead)

---

<img src="docs/Teasor.jpeg" width="90%"/>

MVCHead achieves state-of-the-art for unconditional generation of high fidelity, multi-view consistent 3D Gaussian head avatars in minimal resource setting, without requiring intermediate views, or even 3D data. The generated Gaussian heads capture complex textures and fine facial micro-structure, including wrinkles, hair wisps, ear rims, lip contours, skin blemishes, eyes, and accessories.
</div>

---

## :rocket: **Updates**
- ✅ **Coming Soon**: MVCHead Codes, Weights and FaceGS-10K dataset. Stay Tuned!
- ✅ **June. 7, 2026**: We will be presenting our Poster at CVPR 2026. Stop to check our [Poster](https://cvpr.thecvf.com/media/PosterPDFs/CVPR%202026/39979.png). See everyone at Denver!</i>
- ✅ **May. 25, 2026**: MVCHead [project page](https://humansensinglab.github.io/MVCHead) is now live!
- ✅ **May. 25, 2026**: We released the MVCHead Paper on arXiv. Check the preprint!

## :open_book: **Abstract**

High-fidelity 3D Gaussian head avatar generation is critical for applications such as AR/VR, telepresence, and digital humans. Existing methods depend on multi-view datasets, 3D captures, or intermediate 2D view synthesis. In contrast, we learn both conditional and unconditional 3D head models from randomly sampled 2D images alone, without using multi-view data, 3D supervision, or intermediate view generation. We introduce MVCHead, a single-shot state space model that enforces multi-view consistency (MVC) directly in the 3D representation while regressing 3D Gaussians under these constraints. At its core, we propose a Hierarchical State Space (HiSS) block that progressively refines Gaussians from coarse to fine, while capturing long-range dependencies. Within each HiSS block, we modify Mamba's standard unidirectional scan with the proposed Hierarchical Bi-directional State Scan (HiBiSS) that aligns recurrence with the axes along which multi-view inconsistencies are strongest. Finally, we design an SE(3) Multi-view Critic that judges whether a set of self-renders arises from a single underlying 3D configuration, rewarding cross-view pixel alignment without observing real multi-view pairs. MVCHead achieves state-of-the-art perceptual quality, surpasses prior methods in both texture and geometric consistency, and maintains comparable shape consistency. To demonstrate scalability, we release FaceGS-10K, the first large-scale dataset of ready-to-use 3D Gaussian head assets for training and evaluation of 3D head models.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=humansensinglab/MVCHead&type=Date)](https://www.star-history.com/#humansensinglab/MVCHead&Date)

## License

Shield: [![CC BY-NC 4.0][cc-by-nc-shield]][cc-by-nc]

This work is licensed under a [Creative Commons Attribution-NonCommercial 4.0 International License][cc-by-nc]. 
Permission is granted for non-commercial research. For commerical use, please reachout to our Lab.

[![CC BY-NC 4.0][cc-by-nc-image]][cc-by-nc]

[cc-by-nc]: https://creativecommons.org/licenses/by-nc/4.0/
[cc-by-nc-image]: https://licensebuttons.net/l/by-nc/4.0/88x31.png
[cc-by-nc-shield]: https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg

## Acknowledgements

Parts of the codes have been taken and adapted from the below repos. Please acknowledge and adhere to the licenses of each repository that MVCHead builds upon.
- [Mamba](https://github.com/state-spaces/mamba)
- [VMamba](https://github.com/MzeroMiko/VMamba)
- [VisionMamba](https://github.com/hustvl/Vim)
- [FFHQ](https://github.com/NVlabs/ffhq-dataset)
- [EG3D](https://github.com/NVlabs/eg3d)
- [GSGAN](https://github.com/hse1032/GSGAN)
- [CGSGAN](https://github.com/fraunhoferhhi/cgs-gan)
- [GGHead](https://github.com/tobias-kirschstein/gghead)
- [3DGS](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)
- [Diff. 3DGS Rasterizer](https://github.com/slothfulxtx/diff-gaussian-rasterization)
- [MASt3R](https://github.com/naver/mast3r)
- [DINO](https://github.com/facebookresearch/dino)
- [FeatUp](https://github.com/mhamilton723/FeatUp)
- [MEt3R](https://github.com/mohammadasim98/met3r)
- [MVGBench](https://github.com/xiexh20/MVGBench)

## :bookmark_tabs: Citation
If you find our work useful for your project, please consider adding a star to this repo and citing our paper:
```bibtex
    @inproceedings{chharia2026multiviewconsistent,
        title={Multi-view Consistent 3D Gaussian Head Avatars 'without' Multi-view Generation},
        author={Aviral Chharia and Fernando De la Torre},
        archivePrefix={arXiv},
        primaryClass={cs.CV},
        year={2026}
    } 
```
