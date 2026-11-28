![Example image showing a heart obtained via domain coloring, as shown in Figure 9 of the underlying paper.](https://ms-math-computer.science/images/publications/2024_RLPS_HeartOfDomainColoring.png)

# HeartOfDomainColoring

Domain coloring allows for the visualization of complex functions. 
This repository accompanies the 2024 paper "[Heart of Domain Coloring](https://archive.bridgesmathart.org/2024/bridges2024-115.html)," published by Ulrich Reitebuch, Henriette Lipschütz, Konrad Polthier, and Martin Skrodzki at the Bridges conference for Mathematics and Arts. Specifically, it re-implements some figures from the paper.

## Structure of the repository

This repository holds the code that re-implements several figures from the paper.
Notably, the files corresponding to Figures 5 to 8 implement animations that the paper reports on with several snapshots.
The code in `figure9.glsl` cycles through the gallery items that are printed side-by-side in the original paper.

The codes are meant to be run with [Shadertoy](https://shadertoy.com).
The file `common.glsl` holds what should go into the `Common` tab of Shadertoy, whereas the file `figurex.glsl` has the code to go into the `Image` tab.
Here are links to the different figures on Shadertoy:

- [Figure 5 on Shadertoy](https://www.shadertoy.com/view/wXVfzy)
- [Figure 6 on Shadertoy](https://www.shadertoy.com/view/ffBGDy)
- [Figure 7 on Shadertoy](https://www.shadertoy.com/view/fcS3Dy)
- [Figure 8 on Shadertoy](https://www.shadertoy.com/view/NcS3Dy)
- [Figure 9 on Shadertoy](https://www.shadertoy.com/view/t3yfz3)

## How to reference?

In your publications, please cite the paper:
```bib
@inproceedings{bridges2024:115,
  author      = {Reitebuch, Ulrich and Lipschuetz, Henriette-Sophie and Polthier, Konrad and Skrodzki, Martin},
  title       = {Heart of Domain Coloring},
  pages       = {115--122},
  booktitle   = {Proceedings of Bridges 2024: Mathematics, Art, Music, Architecture, Culture},
  year        = {2024},
  editor      = {Verrill, Helena and Kattchee, Karl and Gould, S. Louise and Torrence, Eve},
  isbn        = {978-1-938664-49-6},
  issn        = {1099-6702},
  publisher   = {Tessellations Publishing},
  address     = {Phoenix, Arizona},

  url         = {http://archive.bridgesmathart.org/2024/bridges2024-115.html}
}
```
