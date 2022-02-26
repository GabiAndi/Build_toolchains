# Construir toolchains para C/C++

## ¿Por qué construir mis propias cadenas de herramientas?

Existen varios motivos por los cuales uno se encuentra con la necesidad de construir un conjunto de herramientas personalizado:

* La versión que se distribuye con el sistema operativo no esta actualizada.
* Es necesario un compilador cruzado para una arquitectura en especifico.
* Las versiones precompiladas no están configuradas se necesita.

Yo me vi en la necesidad de crear aplicaciones Qt para la Raspberry Pi 1 B+, y como las versiones modernas de GCC y de Qt no soportan la arquitectura armv6 por defecto, tuve que ponerme en acción y configurarlo por mi mismo.

Realice un sin fin de intentos que fallaron, pero cada vez que fallaba, aprendia algo nuevo. Seguí insistiendo, buscado, leyendo y probando, hasta que por fin pude entender el proceso de configuración y construcción.

Es por eso que realizo esta serie de instructivos. Un poco para acordarme lo que hice por si en un futuro tengo que hacer lo mismo (en vez de hacer memoria puedo leer este post), pero también para ayudar a los interezados en el tema a que no se encuentren con los mismos inconvenientes que yo tuve.

Aclaro que no soy ningún experto en el tema, y que seguramente, todo lo que hice se pueda mejorar.

Aclaro que no soy ningún experto en el tema, y que seguramente, todo lo que hice se pueda mejorar. Todas las guías están orientadas a un entorno GNU/Linux, no hay nada de Windows por aquí. Tomé esta decición debido a la complejidad excesiva y a la falta de documentación para generar cadenas de herramientas GNU en Windows. Probe hacer esto con MinGW, MSYS2, Cygwin, etc, pero nada me funciono. Tal vez en un futuro lo intente nuevamente, pero por ahora, no esta en mis planes, yo con mi Linux Mint estoy mas que contento.

## GCC

### Compilación nativa

* [Compilación nativa (Linux x86_64).](docs/native/gcc/x86_64-linux.md) (**Funcionando**)

### Compilación cruzada

* [Compilación cruzada (Linux ARM).](docs/cross/gcc/arm-linux.md) (**Funcionando**)

## Clang/LLVM

### Compilación nativa

* [Compilación nativa (Linux x86_64).](docs/native/clang/x86_64-linux.md) (**Funcionando**)

## Qt

### Compilación nativa

* [Compilación nativa (Linux x86_64).](docs/native/qt/x86_64-linux.md) (**Funcionando**)

### Compilación cruzada

* [Compilación cruzada (Linux ARM).](docs/cross/qt/arm-linux.md) (**Funcionando**)

# ACS (Autoconfiguration and Compilation Scripts)

**No proporciono scripts de autoconfiguración** debido a la cantidad de eventos que ocurren durante el proceso de compilación. Es por esto que recomiento hacer todo paso a paso.

# Binarios precompilados

**No proporciono binarios precompilados** debido a la complejidad de portabilizar las cadenas de herramientas, sus dependencias, y las configuraciones de cada máquina en particular.
