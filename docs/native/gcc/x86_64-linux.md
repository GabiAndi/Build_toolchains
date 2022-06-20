# Construir Toolchain nativo

**Aclaración:** ***este documento se revisó por última vez el 11 de marzo del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Construir Toolchain nativo](#construir-toolchain-nativo)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Construcción del Toolchain](#construcción-del-toolchain)
    - [Preparamos el sistema](#preparamos-el-sistema)
    - [Preparando el entorno de compilación](#preparando-el-entorno-de-compilación)
    - [Descargamos las fuentes](#descargamos-las-fuentes)
    - [Requisitos previos de GCC](#requisitos-previos-de-gcc)
    - [Variables de entorno](#variables-de-entorno)
    - [Construcción de GCC](#construcción-de-gcc)
    - [Probamos el compilador](#probamos-el-compilador)
    - [Construcción de Binutils](#construcción-de-binutils)
    - [Construcción de GDB](#construcción-de-gdb)
    - [Eliminamos los archivos temporales](#eliminamos-los-archivos-temporales)

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
export N_CPUS="$(nproc)"

export BINUTILS_VERSION="2.38"
export GCC_VERSION="11.2.0"
export GDB_VERSION="11.2"

export TARGET_OPTIONS="--with-tune=generic"
export EXTRA_OPTIONS="--disable-multilib --enable-multiarch --enable-lto --disable-nls --with-gnu-as --with-gnu-ld"
export GCC_LANGUAGES="c,c++"

export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export GCC_INSTALL_DIR="/opt/gcc-$GCC_VERSION"
export GDB_INSTALL_DIR="/opt/gdb-$GDB_VERSION"
~~~

Creamos las carpetas base en donde construiremos todo:

~~~bash
mkdir -p $WORK_DIR $SRC_DIR $BUILD_DIR
sudo mkdir -p $GCC_INSTALL_DIR $GDB_INSTALL_DIR
sudo chown 1000:1000 -R $GCC_INSTALL_DIR $GDB_INSTALL_DIR
~~~

La ruta de instalación puede ser cualquier carpeta, aunque recomiendo no colocar direcciones como */usr/bin* por ejemplo, que son las ubicaciones del sistema.

### Descargamos las fuentes

Descarguemos las fuentes que usaremos para construir el compilador y Binutils:

~~~bash
cd $SRC_DIR

wget https://ftpmirror.gnu.org/gcc/gcc-$GCC_VERSION/gcc-$GCC_VERSION.tar.xz
wget https://ftpmirror.gnu.org/binutils/binutils-$BINUTILS_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gnu/gdb/gdb-$GDB_VERSION.tar.xz

tar -xf gcc-$GCC_VERSION.tar.xz
tar -xf binutils-$BINUTILS_VERSION.tar.xz
tar -xf gdb-$GDB_VERSION.tar.xz

mkdir -p $BUILD_DIR/gcc $BUILD_DIR/binutils $BUILD_DIR/gdb
~~~

### Requisitos previos de GCC

GCC necesita algunos paquetes extras que podemos descargar dentro de la carpeta de origen:

~~~bash
cd $SRC_DIR/gcc-$GCC_VERSION
contrib/download_prerequisites
~~~

### Variables de entorno

Es conveniente añadir el directorio de instalación al *PATH*, ya que primero construiremos el GCC nuevo con el que se encuentra en nuestro sistema, y luego utilizaremos este GCC nuevo para generar los binutils:

~~~bash
export PATH=$GCC_INSTALL_DIR/bin:$PATH
~~~

### Construcción de GCC

Configuraremos la instalación con los parámetros correspondientes, luego procedemos a construir GCC:

~~~bash
cd $BUILD_DIR/gcc

$SRC_DIR/gcc-$GCC_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$MACHTYPE \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--enable-languages=$GCC_LANGUAGES

make -j$N_CPUS
make install-strip DESTDIR=$GCC_INSTALL_DIR
~~~

### Probamos el compilador

Ahora que tenemos el compilador construido, tenemos que testearlo. Para eso generamos un hola mundo:

~~~bash
mkdir -p $BUILD_DIR/test
cd $BUILD_DIR/test

nano main.cpp
~~~

Y añadimos lo siguiente:

~~~c++
#include <iostream>

int main(void)
{
    std::cout << "Hola mundo!!" << std::endl;

    return 0;
}
~~~

Debido a que tenemos la carpeta de nuestro compilador en el *PATH*, ejecutamos:

~~~bash
g++ -Wall main.cpp -o test
~~~

No debería salir ninguna ninguna advertencia ni tampoco ningún error. Ahora corremos el programa:

~~~bash
./test
~~~

Obteniendo el siguiente resultado:

~~~text
Hola mundo!!
~~~

Si todo nos dio bien, podemos continuar con la creación de binutils y del depurador.

### Construcción de Binutils

Ahora tenemos la posibilidad de generar Binutils con nuestro nuevo GCC:

~~~bash
cd $BUILD_DIR/binutils

$SRC_DIR/binutils-$BINUTILS_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$MACHTYPE \
$TARGET_OPTIONS \
$EXTRA_OPTIONS

make -j$N_CPUS
make install-strip DESTDIR=$GCC_INSTALL_DIR
~~~

### Construcción de GDB

Este paso creará el depurador y lo instalará en la carpeta:

~~~bash
cd $BUILD_DIR/gdb

$SRC_DIR/gdb-$GDB_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$MACHTYPE \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--enable-languages=$GCC_LANGUAGES \
--with-python=/usr/bin/python3

make -j$N_CPUS
make install DESTDIR=$GDB_INSTALL_DIR
~~~

Una vez terminado esto, tendremos el conjunto de herramientas completo.

### Eliminamos los archivos temporales

Los archivos creados en *$WORK_DIR* ya pueden ser borrados sin peligro.
