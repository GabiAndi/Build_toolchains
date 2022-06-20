# Construir Toolchain para ARM Bare Metal

**Aclaración:** ***este documento se revisó por ultima vez el 11 de marzo del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Construir Toolchain para ARM Bare Metal](#construir-toolchain-para-arm-bare-metal)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Construyendo el Toolchain](#construyendo-el-toolchain)
    - [Preparamos el Host](#preparamos-el-host)
    - [Preparando la estructura de carpetas](#preparando-la-estructura-de-carpetas)
    - [Obtenemos las fuentes](#obtenemos-las-fuentes)
    - [Requisitos previos de GCC](#requisitos-previos-de-gcc)
    - [Variables *PATH*](#variables-path)
    - [Construcción de Binutils](#construcción-de-binutils)
    - [Construcción de GCC con Newlib](#construcción-de-gcc-con-newlib)
    - [Construción del depurador GDB](#construción-del-depurador-gdb)
    - [Guardar el Toolchain recien construido](#guardar-el-toolchain-recien-construido)
  - [Conclusiones](#conclusiones)

## Construyendo el Toolchain

### Preparamos el Host

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

### Preparando la estructura de carpetas

En las siguientes instrucciones, asumiré que estás realizando todos los pasos en una carpeta separada, y que mantenes abierta la misma sesión de terminal hasta que todo esté hecho. En mi caso:

- La ruta de descarga de las fuentes se hará en *$HOME/build-toolchain/src*.
- La ruta de compilación de los binarios se hará en *$HOME/build-toolchain/build*.

Ruta de instalación se refiere a donde se guardaran los binarios compilados del Toolchain. En este tutorial se busca crear un compilador GCC portable. Una vez terminemos, empaquetaremos todo en un archivo *cross-armv6zk-hardfp-gcc-X.X.X.tar.xz* que será trasladable a cualquier sistema (siempre y cuando se cumplan las dependencias propias del Toolchain).

Antes de eso, para poder hacer mas sencilla la escritura de comandos, y así evitar errores de tipeo, voy a exportar las rutas y configuraciones como variables de bash:

~~~bash
export N_CPUS="$(nproc)"

export BINUTILS_VERSION="2.38"
export GCC_VERSION="11.2.0"
export NEWLIB_VERSION="4.1.0"
export GDB_VERSION="11.2"

export TARGET="arm-none-eabi"
export TARGET_OPTIONS="--with-arch=armv6zk --with-fpu=vfp --with-float=hard --with-mode=arm"
export EXTRA_OPTIONS="--disable-multilib --enable-multiarch --enable-lto --disable-nls --with-gnu-as --with-gnu-ld --disable-shared --disable-threads"
export GCC_LANGUAGES="c,c++"

export INSTALL_DIR_PREFIX="cross-armv6zk-hardfp"

export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export INSTALL_DIR="$BUILD_DIR/$INSTALL_DIR_PREFIX-gcc-$GCC_VERSION"

export SAVE_TOOLCHAIN_DIR="$HOME"
~~~

Creamos las carpetas base en donde construiremos todo:

~~~bash
mkdir -p $SRC_DIR $BUILD_DIR $INSTALL_DIR
~~~

### Obtenemos las fuentes

Descarguemos lo necesario para construir el compilador cruzado. Binutils, Newlib, GCC y GDB.

Para la biblioteca estandar en mi caso utilizaré Newlib, ya que se adapta mejor a sistemas embebidos o bare metal:

~~~bash
cd $SRC_DIR

wget https://ftpmirror.gnu.org/binutils/binutils-$BINUTILS_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gcc/gcc-$GCC_VERSION/gcc-$GCC_VERSION.tar.xz
wget ftp://sourceware.org/pub/newlib/newlib-$NEWLIB_VERSION.tar.gz
wget https://ftpmirror.gnu.org/gnu/gdb/gdb-$GDB_VERSION.tar.xz

tar -xf binutils-$BINUTILS_VERSION.tar.xz
tar -xf gcc-$GCC_VERSION.tar.xz
tar -xf newlib-$NEWLIB_VERSION.tar.gz
tar -xf gdb-$GDB_VERSION.tar.xz

mkdir -p $BUILD_DIR/binutils $BUILD_DIR/gcc $BUILD_DIR/newlib $BUILD_DIR/gdb
~~~

### Requisitos previos de GCC

GCC necesita algunos paquetes extras que debemos descargar dentro de la carpeta de origen:

~~~bash
cd $SRC_DIR/gcc-$GCC_VERSION
contrib/download_prerequisites
~~~

Ademas, para contruir Newlib junto a GCC tenemos que crear una serie de enlaces simbólicos:

~~~bash
ln -s ../newlib-$NEWLIB_VERSION/newlib .
ln -s ../newlib-$NEWLIB_VERSION/libgloss .
~~~

### Variables *PATH*

Durante todo el proceso de compilación, asegúrese de que el subdirectorio */bin* de la instalación esté en su *PATH*. Puede eliminar este directorio del *PATH* luego de la instalación, pero la mayoría de los pasos de compilación esperan encontrar arm-none-eabi-gcc y otras herramientas de host a través del *PATH*:

~~~bash
export PATH=$INSTALL_DIR/bin:$PATH
~~~

### Construcción de Binutils

Para poder utilizar nuestro compilador cruzado debemos incorporar Binutils a nuestra carpeta resultante por ello es que descargamos las fuentes y procedemos a compilarlo.

A continuación, construimos Binutils para nuestro compilador GCC:

~~~bash
cd $BUILD_DIR/binutils

$SRC_DIR/binutils-$BINUTILS_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS

make -j$N_CPUS
make install-strip DESTDIR=$INSTALL_DIR
~~~

### Construcción de GCC con Newlib

Para Newlib es posible compilar GCC de manera directa, para hacer eso creamos los enlaces simbolicos desde la carpeta de fuentes de Newlib hasta la de fuentes de GCC. Ahora solo hace falta construir:

~~~bash
cd $BUILD_DIR/gcc

$SRC_DIR/gcc-$GCC_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--with-newlib \
--enable-languages=$GCC_LANGUAGES

make -j$N_CPUS
make install DESTDIR=$INSTALL_DIR
~~~

### Construción del depurador GDB

Lo siguiente que debemos hacer antes de probar el compilador es construir el depurador que estará incluido en la lista de binarios:

~~~bash
cd $BUILD_DIR/gdb

$SRC_DIR/gdb-$GDB_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--enable-languages=$GCC_LANGUAGES \
--with-python=/usr/bin/python3

make -j$N_CPUS
make install DESTDIR=$INSTALL_DIR
~~~

Y listo, ahora tenemos el compilador cruzado completo.

### Guardar el Toolchain recien construido

Si deseamos podemos comprimir y guardar en caso que querramos distribuirlos o hacer una copia de seguridad, para ello vaya a la carpeta donde instalo los compiladores:

~~~bash
cd $BUILD_DIR

tar -czf $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION

mv $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $SAVE_TOOLCHAIN_DIR
~~~

## Conclusiones

Como se ve, en realidad esto no es una tarea demasiado complicada, hay que tener una guia sobre que hacer (mas o menos), y despues modificar los argumentos pasados a los scripts de compilación para obtener tus herramientas personalizadas. En mi caso en particular tuve que aprender a hacerlo ya que los compiladores oficiales de ARM GCC 6 o superior, vienen construidos para armv7 por defecto. Esto hacia imposible que pueda usarlos en objetivos armv6zk como la Raspberry Pi 1.
