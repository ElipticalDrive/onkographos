Zobrazování volumetrických mraků v pohybu
-----------------------------------------

Projekt do předmětu PGP - Pokročilá počítačová grafika.

Požadavky
=========

Grafická karta s podporou `OpenGL 4.3`.

Kompilace
=========

Projekt využívá knihovny `OpenGL`, `SDL2` a `glew`. Všechny tyto knihovny
jsou multiplatformní, takže by neměl být problém projekt přeložit na Windows.
Nicméně přiložený makefile je pouze pro Linuxové prostředí.

Pro překlad stačí spustit příkaz `make`. Výsledný spustitelný soubor pak je
`bin/ray-marching`.

Spuštení
========

Po překladu spusťte soubor `bin/ray-marching` z adresáře projektu.
Program nepředpokládá žádné vstupní parametry.

Ovládání
========

Program se ovládá klávesami `W`, `A`, `S`, `D` pro pohyb závislý na směru kamery,
klávesy `SHIFT` a `CTRL` pak slouží pro vertikální posun.

Primárním tlačítkem myši pak lze kamerou rotovat.

Github
======

https://github.com/JOndra91/volumetric-clouds
