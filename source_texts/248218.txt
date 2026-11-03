# 42_fil_de_fer
A computer graphics project for the 42 curriculum. The first of many.

*Fil de fer* is French for "wire-frame models".

It envolves learning how to call functions in the [X11 windowing system](https://en.wikipedia.org/wiki/X_Window_System) through the [Minilibx API](https://github.com/42paris/minilibx-linux.git) and involves ```-lm``` to calculate pixel coordinates that render 3D points onto the 2D screen pixel by pixel through multiple perspectives and rotation.

**Description:** Given a map with file format .fdf containing altitude values, parse and render these values onto the X11 Window in a way that shows an isometric perspective.

**Extra credit:** Allow rotation, translation (panning), zooming in and out, one different projection, and a bonus of my choice.

## Usage

Clone with the generator submodules
```sh
git clone --recurse-submodules https://github.com/Dowiw/42_fil_de_fer.git
```

Tested with:
- GCC 10.5
- C17
- Ubuntu Linux (natively and in Windows 11 WSL)

To compile ```./fdf```:
```sh
make
```

To compile binary with additonal features:
```sh
make bonus
```

In the case where libraries are missing:
```sh
sudo apt-get update
sudo apt-get install libbsd-dev
```

To test:
```sh
./fdf maps/ft_maps/42.fdf
```
You may run any file with the .fdf format. These maps should have both equal row and column sizes throughout, and contain altitude values *(optionally: colors for each value delimited by a comma)*. View files inside maps for format.

Technically, .fdf maps are based on the standard [ARC/INFO Grid Format](https://en.wikipedia.org/wiki/Esri_grid).

## Results

TODO

## References & Appreciation

- Huge thanks to **jgigault** for the map_generator. Without the 42MapGenerator project, it becomes very hard to manually parse and turn real geologic data into valid .fdf maps.
- Thank you to **Arthur Sobreira** as well for the image to fdf map generator. It is very cool to see a couple .jpg and .png images come alive inside the fdf window.

- Thanks to **Oceano** for a [video on MLX](https://www.youtube.com/watch?v=bYS93r6U0zg&t=1379s) that helped introduce the API.
- To **Pen & Paper Science** for videos on vectors and rotation that helped build intuition for point calculation:
  - [Components](https://www.youtube.com/watch?v=IG462zbaAvQ)
  - [Rotation](https://youtu.be/EZufiIwwqFA?si=8ehUxfqb-5297pOd)
- To **No BSCode** for a lovely [explanation](https://www.youtube.com/watch?v=CceepU1vIKo) of the Bresenham Line Algorithm.
- To **Jordan West** for a [look](https://www.youtube.com/watch?v=04oQ2jOUjkU) at the isometric projection.
- To **3Blue1Brown** for an [interesting video](https://www.youtube.com/watch?v=kYB8IZa5AuE) about linear transformations that helped get a feeling for matrix manipulators like rotation matrixes.
- To the **contributers of 42Docs** for a [manual](https://harm-smits.github.io/42docs/libs/minilibx) on the mlx library.
- To **The Coding Train** on a [hands-on video](https://youtu.be/p4Iz0XJY-Qk?si=cI7ZTh10sMXKqa3l) about 3D rotation and projection.
- To **Wikipedia** on an [article](https://en.wikipedia.org/wiki/Bilinear_interpolation) about interpolation. Rendering each pixel with their respective color based on a ratio between the two parent pixels as the line is built. There are also references in the article, thank you to those researchers as well.

Interesting digs:
- [Video on X11](https://www.youtube.com/watch?v=R-N-fgKWYGU)
- If ever want to do rasterization (filling in blocks of pixels):
  - [Rasterization explanation by HuCE](https://www.youtube.com/watch?v=t7Ztio8cwqM)
  - [Triangle raterization by Pikuma](https://www.youtube.com/watch?v=k5wtuKWmV48)
  - [Rasterization algorithms by Sunshine (2012)](https://www.sunshine2k.de/coding/java/TriangleRasterization/TriangleRasterization.html)

> Made in C
