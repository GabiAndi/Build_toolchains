# Construir Toolchain nativo

**Aclaración:** ***este documento se revisó por última vez el 11 de marzo del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Construir Toolchain nativo](#construir-toolchain-nativo)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Construcción del Toolchain](#construcción-del-toolchain)
    - [Preparamos el sistema](#preparamos-el-sistema)
    - [Preparando el entorno de compilación](#preparando-el-entorno-de-compilación)
    - [Descargamos las fuentes](#descargamos-las-fuentes)
    - [Construimos el compilador](#construimos-el-compilador)

## Construcción del Toolchain

### Preparamos el sistema

Dependiendo de su sistema operativo es como deberá instalar los paquetes necesarios.

En caso de **Arch Linux**:

~~~bash
sudo pacman -Syu

sudo pacman -S --needed base-devel python doxygen git openssl unzip wget ncurses rsync texlive-most gperf autogen guile diffutils gmp isl expat clang llvm cmake ninja meson graphviz gtk2
~~~

En caso de **Linux Mint**:

~~~bash
sudo apt update && sudo apt upgrade -y

sudo apt install build-essential python3 python-is-python3 python3-dev doxygen git openssl unzip wget libncurses6 libncursesw6 libncurses-dev rsync gperf texlive-full autogen guile-3.0 diffutils libgmp10 libgmp-dev libisl22 libisl-dev libmpfr6 libmpfr-dev expat clang llvm cmake ninja-build meson graphviz
~~~

### Preparando el entorno de compilación

En las siguientes instrucciones, asumiré que está realizando todos los pasos en una carpeta separada y que mantiene abierta la misma sesión de terminal hasta que todo esté hecho. En mi caso:

- La ruta de descarga de las fuentes se hará en *$HOME/build-toolchain*.
- La ruta de instalación del toolchain se hará en */opt*.

Antes de eso, para poder hacer mas sencilla la escritura de comandos, y así evitar errores de tipeo, voy a exportar las rutas como variables de bash:

~~~bash
export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export INSTALL_DIR="/opt/clang"
~~~

Creamos las carpetas base en donde construiremos todo:

~~~bash
mkdir -p $WORK_DIR $SRC_DIR $BUILD_DIR
sudo mkdir -p $INSTALL_DIR
sudo chown 1000:1000 -R $INSTALL_DIR
~~~

La ruta de instalación puede ser cualquier carpeta, aunque recomiendo no colocar direcciones como */usr/bin* por ejemplo, que son las ubicaciones del sistema.

### Descargamos las fuentes

Descarguemos las fuentes que usaremos para construir el compilador:

~~~bash
cd $SRC_DIR

git clone https://github.com/llvm/llvm-project.git
~~~

### Construimos el compilador

Ahora que tenemos todas las fuentes nos vamos a la carpeta de construcción y contruimos:

~~~bash
cd $BUILD_DIR

cmake \
-DCMAKE_C_COMPILER=clang \
-DCMAKE_CXX_COMPILER=clang++ \
-DLLVM_ENABLE_PROJECTS="clang;lld;lldb;clang-tools-extra;pstl;openmp;polly;mlir" \
-DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libc;libclc;libunwind;compiler-rt" \
-DLLVM_PARALLEL_LINK_JOBS=1 \
-DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-gnu" \
-DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
-DCMAKE_BUILD_TYPE="MinSizeRel" \
-S $SRC_DIR/llvm-project/llvm \
-B . \
-G "Ninja"
~~~

Si todo se construyo de manera correcta ahora puede compilar e instalar:

~~~bash
cmake --build . --parallel 2
cmake --install .
~~~

Cuando el proceso termine tendrá una instalación de clang lista para usar.