# NoiseUtil

 Simple C# helper for creating:

- [Perlin Noise](https://en.wikipedia.org/wiki/Perlin_noise)
- [Fractal Brownian Motion](https://thebookofshaders.com/13/)
- [Domain Warping](https://www.iquilezles.org/www/articles/warp/warp.htm)

Both single-threaded (non-burst) and bursted multi-threaded options available.

Example (Fbm + Domain warping):

![](warp.gif)

Usage (single-threaded):

```c#
var noise = NoiseUtil.GenerateFbmValues(512, 512, scale);
var distortedTexture = NoiseUtil.DistortImage(noise, sourceImage, distortAmount);
```

Usage (burst):

```c#
using (var fbm = NoiseUtil.GenerateFbmValuesBurst(512, 512, fbmScale)) {
        distortedTexture = NoiseUtil.DistortImageBurst(512, 512, fbm, sourceImageArray, distortAmount);
}
```

Some rudimentary performance measurements of generating a 512x512 Fbm with 3 octaves and using it to distort a 512x512 source image on my machine:

| Single Threaded : | Burst + Multi-threaded: |
| ----------------- | ----------------------- |
| ~49.5ms           | ~7.2ms                  |

Lots of speedups can make made, especially on the burst side using [Texture2D](https://docs.unity3d.com/ScriptReference/Texture2D.html).GetRawTextureData, but I opted out of it for now, in the interest of time. Summaries for the bursted methods and helper methods are also on the TODO-list.

Ideally you would do this on the GPU and then Blit the results to a texture, but if speed isn't a concern then this is an option.
