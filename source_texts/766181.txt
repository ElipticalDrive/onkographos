Ray Tracing em Rust

Este projeto é uma adaptação em Rust do famoso tutorial "Ray Tracing in One Weekend" de Peter Shirley, originalmente escrito em C++. O objetivo é implementar um renderizador de ray tracing simples, focando em aprendizado e experimentação.

## Como rodar

Devido à quantidade de cálculos envolvidos, recomendo executar o projeto em modo release para melhor performance, especialmente em computadores sem GPU dedicada:

```bash
cargo run --release
```

O resultado será salvo no arquivo `output.ppm`.

## Referência

- [Peter Shirley — Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)

---
Adaptação e implementação em Rust.
