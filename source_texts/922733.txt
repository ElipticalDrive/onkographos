> Upstream acknowledgment: [KdotJPG][kdotjpg-source] created OpenSimplex Noise.
> [digitalshadow][digitalshadow-source] produced the refactored C# port used by
> VoxelEngine and this package. The port remains available under its original
> Unlicense and public-domain dedication.

# Supprocom.OpenSimplexNoise

`Supprocom.OpenSimplexNoise` supplies deterministic OpenSimplex Noise for C#.
The public `OpenSimplexNoise` type evaluates two-dimensional, three-dimensional,
and four-dimensional coordinates.

The parameterless constructor uses the current clock value as its seed. The
`OpenSimplexNoise(long seed)` constructor gives the same output for the same seed.
These compatibility constructors allocate managed arrays.

## Installation

Use this command to install version 0.1.2.

```shell
dotnet add package Supprocom.OpenSimplexNoise --version 0.1.2
```

## Example

```csharp
using Supprocom.OpenSimplexNoise;

var noise = new OpenSimplexNoise(123456);
var height = noise.Evaluate(12.5, -4.25);
var density = noise.Evaluate(12.5, -4.25, 8.0);
var sample = noise.Evaluate(12.5, -4.25, 8.0, 0.5);
```

## Allocation-free state

Use the static API when the sampling boundary cannot allocate managed memory.
The caller owns four permutation tables and one source scratch buffer. Each
buffer must contain at least 256 bytes, and the buffers must not overlap.

```csharp
using Supprocom.OpenSimplexNoise;

Span<byte> permutation = stackalloc byte[OpenSimplexNoise.PermutationTableLength];
Span<byte> permutation2D = stackalloc byte[OpenSimplexNoise.PermutationTableLength];
Span<byte> permutation3D = stackalloc byte[OpenSimplexNoise.PermutationTableLength];
Span<byte> permutation4D = stackalloc byte[OpenSimplexNoise.PermutationTableLength];
Span<byte> sourceScratch = stackalloc byte[OpenSimplexNoise.SourceScratchLength];

OpenSimplexNoise.Initialize(
    123456,
    permutation,
    permutation2D,
    permutation3D,
    permutation4D,
    sourceScratch);

var height = OpenSimplexNoise.Evaluate(permutation, permutation2D, 12.5, -4.25);
var density = OpenSimplexNoise.Evaluate(permutation, permutation3D, 12.5, -4.25, 8.0);
var sample = OpenSimplexNoise.Evaluate(permutation, permutation4D, 12.5, -4.25, 8.0, 0.5);
```

Initialization writes all five buffers. The source scratch buffer is not
required during evaluation. Successful initialization and evaluation do not
allocate managed memory.

Package tests compare exact output bits with the published version 0.1.0 output.
The tests include zero, negative, large, epsilon, and finite double-limit inputs.
They also measure managed allocations for initialization and evaluation.

## Development

Build and test the solution in Release mode with the .NET 10 SDK:

```shell
dotnet test Supprocom.OpenSimplexNoise.slnx --configuration Release
```

The build runs the complete SDK analyzer rule set and pinned Meziantou,
Roslynator, threading, banned API, public API, and xUnit analyzers. Enabled
diagnostics, including code style suggestions, are treated as build errors.
The tracked public API files preserve the released constructors, constants,
and evaluation overloads. Analyzer dependencies are private to the build.

## Source

The [public repository][repository] contains the preferred source form, tests,
build files, and package documents. This source remains available while the
binary package remains available.

## License

The digitalshadow source remains available under its original Unlicense and
public-domain dedication. Supprocom does not change that upstream license.

Supprocom publishes only its combined work and modifications under
AGPL-3.0-only. See [LICENSE](LICENSE), [NOTICE](NOTICE), and
[THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md) for the complete terms and
notices.

[digitalshadow-source]: https://gist.github.com/digitalshadow/134a3a02b67cecd72181
[kdotjpg-source]: https://gist.github.com/KdotJPG/b1270127455a94ac5d19
[repository]: https://github.com/Supprocom/OpenSimplexNoise
