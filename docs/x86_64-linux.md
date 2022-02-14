# Construir toolchain nativo

**Aclaración:** ***este documento se revisó por última vez el 29 de enero del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Construir toolchain nativo](#construir-toolchain-nativo)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Introducción](#introducción)
  - [Referencias](#referencias)
  - [Construcción del toolchain](#construcción-del-toolchain)
    - [Preparando el sistema](#preparando-el-sistema)
    - [Preparando el entorno de compilación](#preparando-el-entorno-de-compilación)
    - [Descargamos las fuentes](#descargamos-las-fuentes)
    - [Requisitos previos de GCC](#requisitos-previos-de-gcc)
    - [Variables de entorno](#variables-de-entorno)
    - [Construcción de GCC](#construcción-de-gcc)
    - [Probamos el compilador](#probamos-el-compilador)
    - [Construcción de Binutils](#construcción-de-binutils)
    - [Construcción de GDB](#construcción-de-gdb)
    - [Eliminamos los archivos temporales](#eliminamos-los-archivos-temporales)

## Introducción

Este tutorial detalla el proceso que seguí para poder construir un toolchain GNU/Linux para compilación nativa. Me basé en la documentación oficial y en varios artículos para poder armar, mas o menos, una guía base que ayude a los interezados en el tema.

Debe tener en cuenta la versión de GCC que ejecuta su host. Esto es necesario para la comprobación y reducción de errores del compilador nuevo que se construirá. Si desea conocer mas sobre el proceso de compilación de versiones nuevas de GCC a partir de versiones mas antiguas, es posible deba leer [sobre el bootstrapping.](https://en.wikipedia.org/wiki/Bootstrapping_(compilers))

Le recomiendo que **lea a conciencia**. El copiar y pegar comandos como un loco lo llevará a que nada le funcione y se termine por frustrar. Así que tomece el tiempo y la calma necesaria para leer este post, le aseguro que aprendera bastante.

## Referencias

La documentación oficial de GNU no me fue suficiente para poder realizar el proceso de construcción de manera exitosa. Por suerte hay personas que lograron hacerlo y comparten sus experiencias con nosotros (como yo ahora estoy haciendo con ustedes). A continuación pongo a disposición todas las páginas que leí y me ayudaron:

* [Installing GCC.](https://gcc.gnu.org/install/)
* [Building GCC OsDev.](https://wiki.osdev.org/Building_GCC)
* [Building GCC 10 on Ubuntu Linux.](https://solarianprogrammer.com/2016/10/07/building-gcc-ubuntu-linux/)

## Construcción del toolchain

### Preparando el sistema

Primero, asegúrese de que su sistema esté actualizado e instale las dependencias necesarias. Dependiendo de su distribución deberá instalar los paquetes necesarios, en mi caso, para Linux Mint:

~~~TEXT
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential python3 python3-dev python2 python2-dev doxygen git openssl unzip wget libncurses6 libncursesw6 libncurses-dev rsync texinfo texlive autoconf automake gettext gperf autogen guile-3.0 flex patch diffutils libgmp-dev libisl-dev libexpat-dev
~~~

La mayoría de las dependencias ya vienen instaladas en un sistema Linux, sin embargo, puede que mas adelante en la construcción aparezcan errores debido a paquetes que no se encuentren, usted debe corregir esto para poder realizar la compilación con éxito.

Esto no debería tardar mas que un momento.

### Preparando el entorno de compilación

En las siguientes instrucciones, asumiré que está realizando todos los pasos en una carpeta separada y que mantiene abierta la misma sesión de terminal hasta que todo esté hecho. En mi caso:

* La ruta de descarga de las fuentes se hará en *$HOME/build-toolchain*
* La ruta de instalación del toolchain se hará en */opt*

Antes de eso, para poder hacer mas sencilla la escritura de comandos, y así evitar errores de tipeo, voy a exportar las rutas como variables de bash:

~~~TEXT
export N_CPUS="$(nproc)"

export BINUTILS_VERSION="2.37"
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

~~~TEXT
mkdir -p $WORK_DIR $SRC_DIR $BUILD_DIR
sudo mkdir -p $GCC_INSTALL_DIR $GDB_INSTALL_DIR
sudo chown 1000:1000 -R $GCC_INSTALL_DIR $GDB_INSTALL_DIR
~~~

La ruta de instalación puede ser cualquier carpeta, aunque recomiendo no colocar direcciones como */usr/bin* por ejemplo, que son las ubicaciones del sistema.

### Descargamos las fuentes

Descarguemos las fuentes que usaremos para construir el compilador y Binutils:

~~~TEXT
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

~~~TEXT
cd $SRC_DIR/gcc-$GCC_VERSION
contrib/download_prerequisites
~~~

### Variables de entorno

Es conveniente añadir el directorio de instalación al *PATH*, ya que primero construiremos el GCC nuevo con el que se encuentra en nuestro sistema, y luego utilizaremos este GCC nuevo para generar los binutils:

~~~TEXT
export PATH=$GCC_INSTALL_DIR/bin:$PATH
~~~

### Construcción de GCC

Configuraremos la instalación con los parámetros correspondientes, luego procedemos a construir GCC:

~~~TEXT
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

~~~TEXT
mkdir -p $BUILD_DIR/test
cd $BUILD_DIR/test

nano main.cpp
~~~

Y añadimos lo siguiente:

~~~TEXT
#include <iostream>

int main(void)
{
    std::cout << "Hola mundo!!" << std::endl;

    return 0;
}
~~~

Debido a que tenemos la carpeta de nuestro compilador en el *PATH*, ejecutamos:

~~~TEXT
g++ -Wall main.cpp -o test
~~~

No debería salir ninguna ninguna advertencia ni tampoco ningún error. Ahora corremos el programa:

~~~TEXT
./test
~~~

Obteniendo el siguiente resultado:

~~~TEXT
Hola mundo!!
~~~

Si todo nos dio bien, podemos continuar con la creación de binutils y del depurador.

### Construcción de Binutils

Ahora tenemos la posibilidad de generar Binutils con nuestro nuevo GCC:

~~~TEXT
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

~~~TEXT
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
