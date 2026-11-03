# glsl-ray-sphere [![stable](http://badges.github.io/stability-badges/dist/stable.svg)](http://github.com/badges/stability-badges)

GLSL Ray-Sphere intersection bypassing precision issues

Many ray-sphere intersection implementations suffer from precision issues
when the sphere is too small or the ray is too close, as described in this paper:

"Precision Improvements for Ray/Sphere Intersection"
By: Eric Haines, Johannes Gunther, and Tomas Akenine-Moller
<https://link.springer.com/content/pdf/10.1007/978-1-4842-4427-2_7.pdf>

This implementation uses the algorithm that bypasses those issues.

Intended for use with [glslify](http://github.com/chrisdickinson/glslify).

```sh
npm install glsl-ray-sphere
```

## Usage

``` glsl
// Add these lines:
#pragma glslify: Intersection = require(glsl-ray-sphere/intersection)
#pragma glslify: sphereIntersection = require(glsl-ray-sphere/rsi, Intersection=Intersection)

uniform vec3 ray; // normalized
uniform vec3 rayOrigin; // relative to the sphere
uniform float sphereRadius;

// Use it:
void main() {
  Intersection inter = sphereIntersection(rayOrigin, ray, sphereRadius);

  // number of hits in the forward direction:
  int numberOfHitsInFront = inter.hits;
  // is the ray inside?
  bool isInside = inter.isInside;
  // distances from ray origin are t0 and t1
  // so get the points by:
  vec3 first = rayOrigin + ray * inter.t0;
  vec3 second = rayOrigin + ray * inter.t1;
}
```
