# Construir toolchain cruzado para ARM con sistema Linux

**Aclaración:** ***este documento se revisó por última vez el 20 de febrero del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Construir toolchain cruzado para ARM con sistema Linux](#construir-toolchain-cruzado-para-arm-con-sistema-linux)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Introducción](#introducción)
  - [Referencias](#referencias)
  - [Construcción del toolchain](#construcción-del-toolchain)
    - [Preparando el sistema](#preparando-el-sistema)
    - [Preparando el entorno de compilación](#preparando-el-entorno-de-compilación)
    - [Descargamos las fuentes](#descargamos-las-fuentes)
    - [Construimos el compilador](#construimos-el-compilador)
    - [Construimos las librerias para el objetivo](#construimos-las-librerias-para-el-objetivo)

## Introducción

Este tutorial detalla el proceso que seguí para poder construir un toolchain Clang/LLVM para compilación cruzada. Me basé en la documentación oficial y en varios artículos para poder armar, mas o menos, una guía base que ayude a los interezados en el tema.

Debe tener en cuenta la versión de Clang que ejecuta su host. Esto es necesario para la comprobación y reducción de errores del compilador nuevo que se construirá. Si desea conocer mas sobre el proceso de compilación de versiones nuevas de Clang a partir de versiones mas antiguas, es posible deba leer [sobre el bootstrapping.](https://en.wikipedia.org/wiki/Bootstrapping_(compilers))

**Algunas palabras que utilizaremos**

* **Build:** es el sistema donde se está ejecutando el proceso de construcción.

* **Host:** sistema que ejecutará el GCC una vez que esté construido.

* **Target:** sistema en el que se ejecutarán los binarios producidos por el host.

## Referencias

La documentación oficial de LLVM esta bastante bien detallada, lo que me facilitó enormemente el proceso de configuración de las fuentes. Lo links que aportan buena información son:

* [Building LLVM with CMake.](https://llvm.org/docs/CMake.html)
* [Cross-compilation using Clang.](https://clang.llvm.org/docs/CrossCompilation.html)
* [How To Cross-Compile Clang/LLVM using Clang/LLVM.](https://llvm.org/docs/HowToCrossCompileLLVM.html)
* [Cross compiling made easy, using Clang and LLVM.](https://mcilloni.ovh/2021/02/09/cxx-cross-clang/)

Le recomiendo que **lea a conciencia**. El copiar y pegar comandos como un loco lo llevará a que nada le funcione y se termine por frustrar. Así que tomece el tiempo y la calma necesaria para leer este post, le aseguro que aprendera bastante.

## Construcción del toolchain

### Preparando el sistema

Primero, asegúrese de que su sistema esté actualizado e instale las dependencias necesarias. Dependiendo de su distribución deberá instalar los paquetes necesarios, en mi caso, para Linux Mint:

~~~TEXT
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential python3 python3-dev python2 python2-dev doxygen git openssl unzip wget libncurses6 libncursesw6 libncurses-dev rsync texinfo texlive clang clang-tools llvm llvm-dev libclang-dev lldb lld gettext gperf autogen guile-3.0 flex patch diffutils libgmp-dev libisl-dev libexpat-dev libmpfr-dev cmake ninja-build gcc-multilib
~~~

La mayoría de las dependencias ya vienen instaladas en un sistema Linux, sin embargo, puede que mas adelante en la construcción aparezcan errores debido a paquetes que no se encuentren, usted debe corregir esto para poder realizar la compilación con éxito.

Esto no debería tardar mas que un momento.

### Preparando el entorno de compilación

En las siguientes instrucciones, asumiré que está realizando todos los pasos en una carpeta separada y que mantiene abierta la misma sesión de terminal hasta que todo esté hecho. En mi caso:

* La ruta de descarga de las fuentes se hará en *$HOME/build-toolchain*
* La ruta de instalación del toolchain se hará en */opt*

Antes de eso, para poder hacer mas sencilla la escritura de comandos, y así evitar errores de tipeo, voy a exportar las rutas como variables de bash:

~~~TEXT
export TARGET="arm-linux-gnuabihf"

export COMPILE_FLAGS="--target=$TARGET -march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard"

export CLANG_VERSION="15.0.0"

export INSTALL_DIR_PREFIX="cross-pi-2"

export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export INSTALL_DIR="$BUILD_DIR/$INSTALL_DIR_PREFIX-clang-$CLANG_VERSION"

export SAVE_TOOLCHAIN_DIR="$HOME"
~~~

Creamos las carpetas base en donde construiremos todo:

~~~TEXT
mkdir -p $WORK_DIR $SRC_DIR $BUILD_DIR $BUILD_DIR/llvm $INSTALL_DIR
~~~

La ruta de instalación puede ser cualquier carpeta, aunque recomiendo no colocar direcciones como */usr/bin* por ejemplo, que son las ubicaciones del sistema.

### Descargamos las fuentes

Descarguemos las fuentes que usaremos para construir el compilador:

~~~TEXT
cd $SRC_DIR

git clone https://github.com/llvm/llvm-project.git
~~~

### Construimos el compilador

Ahora que tenemos todas las fuentes nos vamos a la carpeta de construcción y contruimos el compilador:

~~~TEXT
cd $BUILD_DIR/llvm

cmake \
-DCMAKE_C_COMPILER=clang \
-DCMAKE_CXX_COMPILER=clang++ \
-DLLVM_ENABLE_PROJECTS="clang;lld;lldb;clang-tools-extra" \
-DLLVM_ENABLE_RUNTIMES="" \
-DLLVM_PARALLEL_LINK_JOBS=1 \
-DLLVM_DEFAULT_TARGET_TRIPLE=$TARGET \
-DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
-DCMAKE_BUILD_TYPE="MinSizeRel" \
-S $SRC_DIR/llvm-project/llvm \
-B . \
-G "Ninja"
~~~

Si todo se construyo de manera correcta ahora puede compilar e instalar:

~~~TEXT
cmake --build . --parallel 2
cmake --install .
~~~

### Construimos las librerias para el objetivo

Una vez tengamos el compilador, ahora debemos crear las librerías estandar para el objetivo. Para lograr esto, apuntamos a nuestro clang generado y le pasamos los flags correspondientes a nuestro sistema:

~~~TEXT
rm -rf *

cmake \
-DCMAKE_C_COMPILER=clang \
-DCMAKE_C_FLAGS="$COMPILE_FLAGS" \
-DCMAKE_CXX_COMPILER=clang++ \
-DCMAKE_CXX_FLAGS="$COMPILE_FLAGS" \
-DLLVM_ENABLE_PROJECTS="" \
-DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libc;libclc;libunwind;compiler-rt" \
-DLLVM_PARALLEL_LINK_JOBS=1 \
-DLLVM_DEFAULT_TARGET_TRIPLE=$TARGET \
-DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
-DCMAKE_BUILD_TYPE="MinSizeRel" \
-S $SRC_DIR/llvm-project/llvm \
-B . \
-G "Ninja"
~~~

Si todo se construyo de manera correcta ahora puede compilar e instalar:

~~~TEXT
cmake --build . --parallel 2
cmake --install .
~~~

Cuando el proceso termine tendrá una instalación de clang lista para usar.