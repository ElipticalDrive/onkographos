<!--

@license Apache-2.0

Copyright (c) 2026 The Stdlib Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

-->


<details>
  <summary>
    About stdlib...
  </summary>
  <p>We believe in a future in which the web is a preferred environment for numerical computation. To help realize this future, we've built stdlib. stdlib is a standard library, with an emphasis on numerical and scientific computation, written in JavaScript (and C) for execution in browsers and in Node.js.</p>
  <p>The library is fully decomposable, being architected in such a way that you can swap out and mix and match APIs and functionality to cater to your exact preferences and use cases.</p>
  <p>When you use stdlib, you can be absolutely certain that you are using the most thorough, rigorous, well-written, studied, documented, tested, measured, and high-quality code out there.</p>
  <p>To join us in bringing numerical computing to the web, get started by checking us out on <a href="https://github.com/stdlib-js/stdlib">GitHub</a>, and please consider <a href="https://opencollective.com/stdlib">financially supporting stdlib</a>. We greatly appreciate your continued support!</p>
</details>

# hc2c

[![NPM version][npm-image]][npm-url] [![Build Status][test-image]][test-url] [![Coverage Status][coverage-image]][coverage-url] <!-- [![dependencies][dependencies-image]][dependencies-url] -->

> Convert a real-valued double-precision floating-point array in [FFTPACK][fftpack] half-complex format to a double-precision complex floating-point array.

<!-- Section to include introductory text. Make sure to keep an empty line after the intro `section` element and another before the `/section` close. -->

<section class="intro">

</section>

<!-- /.intro -->

<!-- Package usage documentation. -->

<section class="installation">

## Installation

```bash
npm install @stdlib/fft-base-fftpack-float64-hc2c
```

Alternatively,

-   To load the package in a website via a `script` tag without installation and bundlers, use the [ES Module][es-module] available on the [`esm`][esm-url] branch (see [README][esm-readme]).
-   If you are using Deno, visit the [`deno`][deno-url] branch (see [README][deno-readme] for usage intructions).
-   For use in Observable, or in browser/node environments, use the [Universal Module Definition (UMD)][umd] build available on the [`umd`][umd-url] branch (see [README][umd-readme]).

The [branches.md][branches-url] file summarizes the available branches and displays a diagram illustrating their relationships.

To view installation and usage instructions specific to each branch build, be sure to explicitly navigate to the respective README files on each branch, as linked to above.

</section>

<section class="usage">

## Usage

```javascript
var hc2c = require( '@stdlib/fft-base-fftpack-float64-hc2c' );
```

#### hc2c( N, hc, strideHc, offsetHc, out, strideOut, offsetOut )

Converts a real-valued double-precision floating-point array in [FFTPACK][fftpack] half-complex format to a double-precision complex floating-point array.

```javascript
var Float64Array = require( '@stdlib/array-float64' );
var Complex128Array = require( '@stdlib/array-complex128' );
var floor = require( '@stdlib/math-base-special-floor' );
var rffti = require( '@stdlib/fft-base-fftpack-float64-rffti' );
var rfftf = require( '@stdlib/fft-base-fftpack-float64-rfftf' );

var N = 4;
var w = new Float64Array( ( 2*N ) + 34 );

rffti( N, w, 1, 0 );

var r = new Float64Array( [ 1.0, 2.0, 3.0, 4.0 ] );

rfftf( N, r, 1, 0, w, 1, 0 );
// r => <Float64Array>[ 10.0, -2.0, 2.0, -2.0 ]

var out = new Complex128Array( floor( N/2 ) + 1 );

hc2c( N, r, 1, 0, out, 1, 0 );
// out => <Complex128Array>[ 10.0, 0.0, -2.0, 2.0, -2.0, 0.0 ]
```

The function accepts the following arguments:

-   **N**: length of the original real sequence.
-   **hc**: input array in [FFTPACK][fftpack] half-complex format.
-   **strideHc**: stride length for `hc`.
-   **offsetHc**: starting index for `hc`.
-   **out**: output complex array.
-   **strideOut**: stride length for `out`.
-   **offsetOut**: starting index for `out`.

</section>

<!-- /.usage -->

<!-- Package usage notes. Make sure to keep an empty line after the `section` element and another before the `/section` close. -->

<section class="notes">

## Notes

-   [FFTPACK][fftpack]'s [`rfftf`][@stdlib/fft/base/fftpack/float64/rfftf] stores the forward DFT of a length `N` real sequence in `N` real values using the following layout:

    ```text
    Even N: [ DC, Re(1), Im(1), Re(2), Im(2), ..., Re(N/2-1), Im(N/2-1), Nyquist ]
    Odd N:  [ DC, Re(1), Im(1), Re(2), Im(2), ..., Re((N-1)/2), Im((N-1)/2) ]
    ```

    where `DC` is the zero-frequency component and `Nyquist` is the component at frequency `N/2` (even `N` only). Both are always purely real.

-   The output is a complex-valued array of length `floor(N/2) + 1`. The `DC` and `Nyquist` components are stored with an explicit zero imaginary part.

-   This half-complex format differs from other conventions (e.g., [FFTW][fftw-halfcomplex]). In FFTW's half-complex format, all real components are stored first, followed by imaginary components in reverse order.

</section>

<!-- /.notes -->

<section class="examples">

## Examples

<!-- eslint no-undef: "error" -->

```javascript
var Float64Array = require( '@stdlib/array-float64' );
var Complex128Array = require( '@stdlib/array-complex128' );
var floor = require( '@stdlib/math-base-special-floor' );
var discreteUniform = require( '@stdlib/random-array-discrete-uniform' );
var rffti = require( '@stdlib/fft-base-fftpack-float64-rffti' );
var rfftf = require( '@stdlib/fft-base-fftpack-float64-rfftf' );
var hc2c = require( '@stdlib/fft-base-fftpack-float64-hc2c' );

var N = 8;
var opts = {
    'dtype': 'float64'
};
var r = discreteUniform( N, -10, 10, opts );
var w = new Float64Array( ( 2*N ) + 34 );

console.log( r );

rffti( N, w, 1, 0 );
rfftf( N, r, 1, 0, w, 1, 0 );

console.log( r );

var out = new Complex128Array( floor( N/2 ) + 1 );

hc2c( N, r, 1, 0, out, 1, 0 );

console.log( out );
```

</section>

<!-- /.examples -->

<!-- Section to include cited references. If references are included, add a horizontal rule *before* the section. Make sure to keep an empty line after the `section` element and another before the `/section` close. -->

<section class="references">

</section>

<!-- /.references -->

<!-- Section for related `stdlib` packages. Do not manually edit this section, as it is automatically populated. -->

<section class="related">

</section>

<!-- /.related -->

<!-- Section for all links. Make sure to keep an empty line after the `section` element and another before the `/section` close. -->


<section class="main-repo" >

* * *

## Notice

This package is part of [stdlib][stdlib], a standard library for JavaScript and Node.js, with an emphasis on numerical and scientific computing. The library provides a collection of robust, high performance libraries for mathematics, statistics, streams, utilities, and more.

For more information on the project, filing bug reports and feature requests, and guidance on how to develop [stdlib][stdlib], see the main project [repository][stdlib].

#### Community

[![Chat][chat-image]][chat-url]

---

## License

See [LICENSE][stdlib-license].


## Copyright

Copyright &copy; 2016-2026. The Stdlib [Authors][stdlib-authors].

</section>

<!-- /.stdlib -->

<!-- Section for all links. Make sure to keep an empty line after the `section` element and another before the `/section` close. -->

<section class="links">

[npm-image]: http://img.shields.io/npm/v/@stdlib/fft-base-fftpack-float64-hc2c.svg
[npm-url]: https://npmjs.org/package/@stdlib/fft-base-fftpack-float64-hc2c

[test-image]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/actions/workflows/test.yml/badge.svg?branch=main
[test-url]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/actions/workflows/test.yml?query=branch:main

[coverage-image]: https://img.shields.io/codecov/c/github/stdlib-js/fft-base-fftpack-float64-hc2c/main.svg
[coverage-url]: https://codecov.io/github/stdlib-js/fft-base-fftpack-float64-hc2c?branch=main

<!--

[dependencies-image]: https://img.shields.io/david/stdlib-js/fft-base-fftpack-float64-hc2c.svg
[dependencies-url]: https://david-dm.org/stdlib-js/fft-base-fftpack-float64-hc2c/main

-->

[chat-image]: https://img.shields.io/badge/zulip-join_chat-brightgreen.svg
[chat-url]: https://stdlib.zulipchat.com

[stdlib]: https://github.com/stdlib-js/stdlib

[stdlib-authors]: https://github.com/stdlib-js/stdlib/graphs/contributors

[umd]: https://github.com/umdjs/umd
[es-module]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules

[deno-url]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/tree/deno
[deno-readme]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/blob/deno/README.md
[umd-url]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/tree/umd
[umd-readme]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/blob/umd/README.md
[esm-url]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/tree/esm
[esm-readme]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/blob/esm/README.md
[branches-url]: https://github.com/stdlib-js/fft-base-fftpack-float64-hc2c/blob/main/branches.md

[stdlib-license]: https://raw.githubusercontent.com/stdlib-js/fft-base-fftpack-float64-hc2c/main/LICENSE

[fftpack]: https://www.netlib.org/fftpack/

[fftw-halfcomplex]: https://www.fftw.org/fftw3_doc/The-Halfcomplex_002dformat-DFT.html

[@stdlib/fft/base/fftpack/float64/rfftf]: https://github.com/stdlib-js/fft-base-fftpack-float64-rfftf

</section>

<!-- /.links -->
