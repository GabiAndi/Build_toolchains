# Construir toolchains para C/C++

## ¿Por qué construir mis propias cadenas de herramientas?

Existen varios motivos por los cuales uno se encuentra con la necesidad de construir un conjunto de herramientas personalizado:

* La versión que se distribuye con el sistema operativo no esta actualizada.
* Es necesario un compilador cruzado para una arquitectura en especifico.
* Las versiones precompiladas no están configuradas como se necesita.

Yo me vi en la necesidad de crear aplicaciones **Qt** para la **Raspberry Pi 1 B+**, y como las versiones modernas de **GCC** no soportan la arquitectura **armv6** por defecto, tuve que ponerme en acción y configurarlo por mi mismo.

Realice un sin fin de intentos que fallaron, pero cada vez que fallaba, aprendia algo nuevo. Seguí insistiendo, buscado, leyendo y probando, hasta que por fin pude entender el proceso de configuración y construcción.

Es por eso que realizo esta serie de instructivos. Para acordarme de lo que hice, por si en un futuro tengo que hacer lo mismo (en vez de hacer memoria puedo leer este post), pero también para ayudar a los interezados en el tema a que no se encuentren con los mismos inconvenientes que yo tuve.

Aclaro que no soy ningún experto en el tema, y que seguramente, todo lo que hice se pueda mejorar. Todas las guías están orientadas a un entorno GNU/Linux, no hay nada de Windows por aquí. Tomé esta decición debido a la complejidad excesiva y a la falta de documentación para generar cadenas de herramientas GNU en Windows. Probe hacer esto con MinGW, MSYS2, Cygwin, etc, pero nada me funciono. Tal vez en un futuro lo intente nuevamente, pero por ahora, no esta en mis planes, yo con mi Linux estoy mas que contento.

Esta seríe de tutoriales detalla el proceso que seguí para poder construir distintas herramientas para compilación nativa o cruzada. Basé mi investigación en la documentación oficial y en varios artículos de terceros, para poder armar una guía base que ayude a los interezados en el tema.

## Palabras clave

- **Build:** es el sistema donde se está ejecutando el proceso de construcción de la cadena de herramientas.

- **Host:** sistema en donde se ejecutará la cadena de herramientas creada.

- **Target:** sistema en el que se ejecutarán los binarios producidos por la cadena de herramientas del Host.

## Experiencia personal

Crear un conjunto de herramientas de compilación es jugar un poco a la loteria. Existen multitud de configuraciones y versiones de paquetes disponibles, así que tendrá que encontrar la que funcione.

Es importante saber que no todas las versiones de las librerías estandar son compatibles con todas las versiones de los compiladores. Para asegurar un buen grado de compatibilidad, recomiendo que se generé una cadena de herramientas de aproximadamente la misma fecha de lanzamiento que las utilidades.

# Linux

## GCC

### Compilación nativa

* [Compilación nativa (Linux x86_64).](docs/native/gcc/x86_64-linux.md) (**Funcionando**)

### Compilación cruzada

* [Compilación cruzada (Linux ARM).](docs/cross/gcc/arm-linux.md) (**Funcionando**)
* [Compilación cruzada (Bare metal ARM).](docs/cross/gcc/arm-none.md) (**Funcionando**)

## Clang/LLVM

### Compilación nativa

* [Compilación nativa (Linux x86_64).](docs/native/clang/x86_64-linux.md) (**Funcionando**)

## Qt

### Compilación nativa

* [Compilación nativa (Linux x86_64).](docs/native/qt/x86_64-linux.md) (**Funcionando**)

### Compilación cruzada

* [Compilación cruzada (Linux ARM).](docs/cross/qt/arm-linux.md) (**Funcionando**)


# Referencias

- [Sobre el kernel de la Raspberry Pi (variación de Linux).](https://www.raspberrypi.org/documentation/linux/kernel/building.md)
- [Artículo de compilación de GCC osdev.org.](https://wiki.osdev.org/Building_GCC)
- [Artículo de compilación cruzada de GCC osdev.org.](https://wiki.osdev.org/GCC_Cross-Compiler)
- [Documentación oficial de GCC.](https://gcc.gnu.org/install/configure.html)
- [Building GCC as a cross compiler for Raspberry Pi.](https://solarianprogrammer.com/2018/05/06/building-gcc-cross-compiler-raspberry-pi/)
- [How to Build a GCC Cross-Compiler.](https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/)
- [Very Simple Guide for Building Cross Compilers Tips.](http://www.ifp.illinois.edu/~nakazato/tips/xgcc.html)
- [Building GDB and GDBserver for cross debugging.](https://sourceware.org/gdb/wiki/BuildingCrossGDBandGDBserver)
- [Raspberry Pi Toolchains.](https://sourceforge.net/projects/raspberry-pi-cross-compilers/files/)
- [Raspberry Pi 3 have CPU armv7l instead Armv8.](https://www.raspberrypi.org/forums/viewtopic.php?t=140572)
- [Bootstrapping.](https://en.wikipedia.org/wiki/Bootstrapping_(compilers))
- [Build cross arm-gcc with newlib.](https://gist.github.com/iori-yja/9271860)
- [Building LLVM with CMake.](https://llvm.org/docs/CMake.html)
- [Qt 6 - Build from source (Both Dynamic and Static).](https://www.youtube.com/watch?v=2qsR8Dw8uzA)
- [Building Qt Sources.](https://doc-snapshots.qt.io/qt6-dev/build-sources.html)