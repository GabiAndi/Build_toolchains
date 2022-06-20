# Construir toolchain cruzado para ARM con sistema Linux

**Aclaración:** ***este documento se revisó por ultima vez el 16 de junio del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Construir toolchain cruzado para ARM con sistema Linux](#construir-toolchain-cruzado-para-arm-con-sistema-linux)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Pasos previos](#pasos-previos)
    - [Verificamos la versión de paquetes de destino](#verificamos-la-versión-de-paquetes-de-destino)
  - [Construyendo el Toolchain inicial](#construyendo-el-toolchain-inicial)
    - [Preparamos el Host](#preparamos-el-host)
    - [Preparando la estructura de carpetas](#preparando-la-estructura-de-carpetas)
    - [Obtenemos las fuentes](#obtenemos-las-fuentes)
    - [Requisitos previos de GCC](#requisitos-previos-de-gcc)
    - [Variables *PATH*](#variables-path)
    - [Copiar los encabezados del kernel](#copiar-los-encabezados-del-kernel)
    - [Construcción de Binutils](#construcción-de-binutils)
    - [Construcción de GCC desnudo](#construcción-de-gcc-desnudo)
    - [Construcción parcial de Glibc](#construcción-parcial-de-glibc)
    - [Construcción de las bibliotecas de soporte](#construcción-de-las-bibliotecas-de-soporte)
    - [Biblioteca C estándar](#biblioteca-c-estándar)
    - [Construimos el GCC completo](#construimos-el-gcc-completo)
    - [Construcción del depurador GDB](#construcción-del-depurador-gdb)
    - [Probamos el Toolchain](#probamos-el-toolchain)
    - [Guardar el Toolchain recien construido](#guardar-el-toolchain-recien-construido)
  - [Contruir una versión mas reciente del Toolchain](#contruir-una-versión-mas-reciente-del-toolchain)
    - [Preparando la estructura de carpetas](#preparando-la-estructura-de-carpetas-1)
    - [Obtenemos las fuentes](#obtenemos-las-fuentes-1)
    - [Requisitos previos de GCC](#requisitos-previos-de-gcc-1)
    - [Copiar los encabezados del kernel](#copiar-los-encabezados-del-kernel-1)
    - [Construcción de Binutils](#construcción-de-binutils-1)
    - [Construcción de Glibc con el compilador GCC compatible](#construcción-de-glibc-con-el-compilador-gcc-compatible)
    - [Construimos el GCC](#construimos-el-gcc)
    - [Construcción del depurador GDB](#construcción-del-depurador-gdb-1)
    - [Probamos el Toolchain](#probamos-el-toolchain-1)
    - [Guardar el Toolchain recien construido](#guardar-el-toolchain-recien-construido-1)
  - [Conclusiones](#conclusiones)

## Pasos previos

### Verificamos la versión de paquetes de destino

Al día de hoy, Raspberry Pi OS viene con GCC 10.2.1, GDB 10.1, Binutils 2.35.2 y Glibc 2.31. Es importante que construyamos nuestro compilador cruzado usando la misma versión de Glibc que la Raspberry Pi. Esto nos permitirá integrarnos con el sistema operativo. Sin embargo no necesario que los Binutils, GCC o GDB necesiten ser de la misma versión, pero sigue siendo recomendable que si lo seán.
	
Si en un futuro las versiones incorporadas cambian, puede verificarlas con estos comandos (ejecutados desde la Raspberry Pi obviamente):

~~~bash
gcc --version
gdb --version
ld -v
ldd --version
~~~

Para mi caso en particular la salida obtenida fue la siguiente:

- Versión de GCC: 10.2.1.
- Versión de GDB: 10.1.
- Versión de Binutils: 2.35.2.
- Versión de Glibc: 2.31.

Como no siempre es posible tener la versión de Glibc compatible con el compilador de GCC que queremos generar, en este tutorial construimos una versión compatible del toolchain, y luego construimos la versión final del toolchain (compilado con la versión compatible).

Si la versión de GCC, Binutils y Glibc son compatibles y se puede crear la cadena de herramientas completa sin problemas, entonces con realizar la sección **Toolchain inicial** usted ya tendrá todo lo que necesita.

## Construyendo el Toolchain inicial

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

Ruta de compilación se refiere a donde se guardaran los binarios compilados del Toolchain. En este tutorial se busca crear un compilador portable. Una vez terminemos, empaquetaremos todo en un archivo *cross-pi-X-gcc-X.X.X.tar.xz* que será trasladable a cualquier sistema (siempre y cuando se cumplan las dependencias propias del Toolchain).

Antes de eso, para poder hacer mas sencilla la escritura de comandos, y así evitar errores de tipeo, voy a exportar las rutas y configuraciones como variables de bash:

~~~bash
export N_CPUS="$(nproc)"

export BINUTILS_VERSION="2.38"
export GCC_VERSION="8.5.0"
export GLIBC_VERSION="2.31"
export GDB_VERSION="11.2"

export TARGET="arm-linux-gnueabihf"
export TARGET_OPTIONS="--with-arch=armv6zk --with-fpu=vfp --with-float=hard --with-mode=arm"
export EXTRA_OPTIONS="--disable-multilib --enable-multiarch --enable-lto --disable-nls --with-gnu-as --with-gnu-ld"
export GCC_LANGUAGES="c,c++"

export INSTALL_DIR_PREFIX="cross-pi-1"

export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export INSTALL_DIR="$BUILD_DIR/$INSTALL_DIR_PREFIX-gcc-$GCC_VERSION"
export SYSROOT="$INSTALL_DIR/$TARGET/libc"

export SAVE_TOOLCHAIN_DIR="$HOME"
~~~

Creamos las carpetas base en donde construiremos todo:

~~~bash
mkdir -p $SRC_DIR $BUILD_DIR $INSTALL_DIR
~~~

### Obtenemos las fuentes

Descarguemos lo necesario para construir el compilador cruzado. Binutils, Glibc, GCC, GDB y la última versión del kernel de Raspberry:

~~~bash
cd $SRC_DIR

wget https://ftpmirror.gnu.org/binutils/binutils-$BINUTILS_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gcc/gcc-$GCC_VERSION/gcc-$GCC_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gnu/glibc/glibc-$GLIBC_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gnu/gdb/gdb-$GDB_VERSION.tar.xz
git clone --depth=1 https://github.com/raspberrypi/linux

rm -rf linux/.git*

tar -czf linux.tar.xz linux

tar -xf binutils-$BINUTILS_VERSION.tar.xz
tar -xf gcc-$GCC_VERSION.tar.xz
tar -xf glibc-$GLIBC_VERSION.tar.xz
tar -xf gdb-$GDB_VERSION.tar.xz

mkdir -p $BUILD_DIR/binutils $BUILD_DIR/gcc $BUILD_DIR/glibc $BUILD_DIR/gdb
~~~

Tanto GCC como Binutils, son las utilidades para construcción de binarios. Eso significa que conviene tener (siempre y cuando sean compatibles) las versiones mas actualizadas de ambos. El tema esta en la versión de Glibc. Este conjunto de librerías estándar **sí debe ser igual al de la Raspberry**. Ya que los binarios compilados se vinculan (por defecto) de manera dinámica.

### Requisitos previos de GCC

GCC necesita algunos paquetes extras que debemos descargar dentro de la carpeta de origen:

~~~bash
cd $SRC_DIR/gcc-$GCC_VERSION
contrib/download_prerequisites
~~~

### Variables *PATH*

Durante todo el proceso de compilación, asegúrese de que el subdirectorio */bin* de la instalación esté en su *PATH*. Puede eliminar este directorio del *PATH* luego de la instalación, pero la mayoría de los pasos de compilación esperan encontrar arm-linux-gnueabihf-gcc y otras herramientas de host a través del *PATH*:

~~~bash
export PATH=$INSTALL_DIR/bin:$PATH
~~~

### Copiar los encabezados del kernel

Este paso instala los archivos de encabezado del kernel de Linux, lo que finalmente permitirá que los programas creados con nuestra nueva cadena de herramientas realicen llamadas del sistema al kernel en el entorno de destino.

Copie los encabezados del kernel en las carpetas anteriores, consulte la [documentación de Raspbian](https://www.raspberrypi.org/documentation/linux/kernel/building.md) para obtener más información:

~~~bash
cd $SRC_DIR/linux

KERNEL=kernel

make \
ARCH=arm \
INSTALL_HDR_PATH=$SYSROOT/usr \
headers_install

mkdir -p $SYSROOT/usr/lib
~~~

### Construcción de Binutils

Para poder utilizar nuestro compilador cruzado debemos incorporar Binutils a nuestra carpeta resultante por ello es que descargamos las fuentes y procedemos a compilarlo:

~~~bash
cd $BUILD_DIR/binutils

$SRC_DIR/binutils-$BINUTILS_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT

make -j$N_CPUS
make install-strip DESTDIR=$INSTALL_DIR
~~~

### Construcción de GCC desnudo

Todos los pasos restantes implican la construcción de GCC y Glibc. El truco es que hay partes de GCC que dependen de partes de Glibc que ya se están construyendo y viceversa. No podemos construir ninguno de los paquetes en un solo paso. Necesitamos primero construir una versión desnuda de GCC (sin ninguna librería estándar incorporada), para luego construir Glibc y por último compilar la versión completa de GCC con las librerías estándar ya incorporadas.

Este paso creará compiladores cruzados de C y C++ de GCC únicamente y los instalará en */bin*. No invocará a esos compiladores para crear bibliotecas todavía:

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
--without-headers \
--enable-languages=$GCC_LANGUAGES \
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT

make -j$N_CPUS all-gcc
make install-strip-gcc DESTDIR=$INSTALL_DIR
~~~

### Construcción parcial de Glibc

En este paso, instalamos los encabezados de la biblioteca C estándar de Glibc en */include*. También usamos el compilador de C del paso anterior para compilar los archivos de inicio de la biblioteca e instalarlos en */lib*. Finalmente, creamos un par de archivos ficticios que se esperan en el paso siguiente, pero que serán reemplazados en el paso posterior al siguiente:

~~~bash
cd $BUILD_DIR/glibc

$SRC_DIR/glibc-$GLIBC_VERSION/configure \
--prefix=/usr \
--build=$MACHTYPE \
--host=$TARGET \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--with-headers=$SYSROOT/usr/include \
--with-lib=$SYSROOT/usr/lib \
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT \
libc_cv_forced_unwind=yes

make install-bootstrap-headers=yes install-headers DESTDIR=$SYSROOT
make -j$N_CPUS csu/subdir_lib

install \
csu/crt1.o \
csu/crti.o \
csu/crtn.o \
$SYSROOT/usr/lib

$TARGET-gcc \
-nostdlib \
-nostartfiles \
-shared \
-x c /dev/null \
-o $SYSROOT/usr/lib/libc.so

touch $SYSROOT/usr/include/gnu/stubs.h $SYSROOT/usr/include/bits/stdio_lim.h
~~~

### Construcción de las bibliotecas de soporte

Este paso utiliza los compiladores cruzados creados dos pasos antes para crear la biblioteca de soporte del compilador. La biblioteca de soporte del compilador contiene algunas excepciones de C++ que manejan código repetitivo, entre otras cosas. Esta biblioteca depende de los archivos de inicio instalados en el paso anterior. La biblioteca en sí es necesaria en el paso siguiente:

~~~bash
cd $BUILD_DIR/gcc

make -j$N_CPUS all-target-libgcc
make install-target-libgcc DESTDIR=$INSTALL_DIR
~~~

### Biblioteca C estándar

En este paso, finalizamos el paquete Glibc, que construye la biblioteca C estándar e instala sus archivos en */lib*:

~~~bash
cd $BUILD_DIR/glibc

make -j$N_CPUS
make install DESTDIR=$SYSROOT
~~~

### Construimos el GCC completo

Finalmente, terminamos de compilar el paquete GCC de manera completa. Esto nos dará soporte completo a las bibliotecas estándar:

~~~bash
cd $BUILD_DIR/gcc

rm -rf *

$SRC_DIR/gcc-$GCC_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--enable-languages=$GCC_LANGUAGES \
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT

make -j$N_CPUS
make install-strip DESTDIR=$INSTALL_DIR
~~~

### Construcción del depurador GDB

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
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT \
--with-python=/usr/bin/python3

make -j$N_CPUS
make install DESTDIR=$INSTALL_DIR
~~~

Y listo, ahora tenemos el compilador cruzado completo.

### Probamos el Toolchain

Ahora que tenemos el compilador construido, tenemos que testearlo. Para eso genere un hola mundo para pasarlo a la Raspberry Pi y probarlo:

~~~bash
mkdir -p $BUILD_DIR/test
cd $BUILD_DIR/test

nano main.cpp
~~~

Y añadimos lo siguiente:

~~~c++
#include <iostream>

using namespace std;

int main (void)
{
    cout << "Hola mundo!!" << endl;

    return 0;
}
~~~

Debido a que tenemos (si no cerro la terminal desde toda la compilación) la carpeta de nuestro compilador en el PATH, probamos compilar el programa:

~~~bash
arm-linux-gnueabihf-g++ -Wall main.cpp -o test
~~~

No deberia salir ninguna ninguna advertencia ni tampoco ningun error. Por último pasamos el programa a la Raspberry via scp:

~~~bash
scp test RPI_USER@RPI_IP:~
~~~

En la Raspberry Pi ejecutamos:

~~~bash
./test
~~~

Obteniendo el siguiente resultado:

~~~text
Hola mundo!!
~~~

Si llegaste hasta aca felicitaciones, fuiste capaz de crear tu propio GCC cruzado para Raspberry Pi.

### Guardar el Toolchain recien construido

Si deseamos podemos comprimir y guardar en caso que querramos distribuirlos o hacer una copia de seguridad, para ello vaya a la carpeta donde instalo los compiladores:

~~~bash
cd $BUILD_DIR

tar -czf $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION

mv $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $SAVE_TOOLCHAIN_DIR
~~~

## Contruir una versión mas reciente del Toolchain

Lo que vamos a hacer a continuación es crear un GCC mas moderno con librerías estándar mas antiguas. Por lo que utilizaremos el compilador creado anteriormente (que es compatible con ese Glibc) para construir la Glibc antigua y luego construir nuestro nuevo GCC con la biblioteca estándar.

Tratar de construir Glibc antigua con un GCC moderno causará errores raros que no podrá solucionar, por eso preferí utilizar este enfoque que nos evitará problemas.

En este caso voy a compilar la última versión de GCC a partir de la construcción de la Glibc con la versión compatible.

### Preparando la estructura de carpetas

Tenemos que ir a la carpeta de fuentes y limpiar un poco la instalación anterior:

~~~bash
cd $SRC_DIR

rm -rf binutils-$BINUTILS_VERSION gcc-$GCC_VERSION glibc-$GLIBC_VERSION gdb-$GDB_VERSION linux

cd $BUILD_DIR

rm -rf binutils gcc glibc gdb
~~~

Vamos a actualizar las variables de bash para la nueva carpeta:

~~~bash
export GCC_VERSION="10.2.0"

export INSTALL_DIR="$BUILD_DIR/$INSTALL_DIR_PREFIX-gcc-$GCC_VERSION"
export SYSROOT="$INSTALL_DIR/$TARGET/libc"
~~~

Por ultimo generamos la nueva carpeta de instalación:

~~~bash
mkdir -p $INSTALL_DIR
~~~

### Obtenemos las fuentes

Descarguemos lo necesario para construir el compilador cruzado. GCC y la última versión del kernel de Raspberry Pi OS:

~~~bash
cd $SRC_DIR

wget https://ftpmirror.gnu.org/gcc/gcc-$GCC_VERSION/gcc-$GCC_VERSION.tar.xz

tar -xf binutils-$BINUTILS_VERSION.tar.xz
tar -xf gcc-$GCC_VERSION.tar.xz
tar -xf glibc-$GLIBC_VERSION.tar.xz
tar -xf gdb-$GDB_VERSION.tar.xz
tar -xf linux.tar.xz

mkdir -p $BUILD_DIR/binutils $BUILD_DIR/gcc $BUILD_DIR/glibc $BUILD_DIR/gdb
~~~

### Requisitos previos de GCC

GCC necesita algunos paquetes extras que debemos descargar dentro de la carpeta de origen:

~~~bash
cd $SRC_DIR/gcc-$GCC_VERSION
contrib/download_prerequisites
~~~

### Copiar los encabezados del kernel

Este paso instala los archivos de encabezado del kernel de Linux, lo que finalmente permitirá que los programas creados con nuestra nueva cadena de herramientas realicen llamadas del sistema al kernel en el entorno de destino.

Copie los encabezados del kernel en las carpetas anteriores, consulte la [documentación de Raspbian](https://www.raspberrypi.org/documentation/linux/kernel/building.md) para obtener más información:

~~~bash
cd $SRC_DIR/linux

KERNEL=kernel

make \
ARCH=arm \
INSTALL_HDR_PATH=$SYSROOT/usr \
headers_install

mkdir -p $SYSROOT/usr/lib
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
$EXTRA_OPTIONS \
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT

make -j$N_CPUS
make install-strip DESTDIR=$INSTALL_DIR
~~~

### Construcción de Glibc con el compilador GCC compatible

Ya que no hemos cerrado la terminal, el compilador anterior que creamos sigue en el *PATH*, por lo que utilizaremos este para construir nuevamente las librerías estándar:

~~~bash
cd $BUILD_DIR/glibc

$SRC_DIR/glibc-$GLIBC_VERSION/configure \
--prefix=/usr \
--build=$MACHTYPE \
--host=$TARGET \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--with-headers=$SYSROOT/usr/include \
--with-lib=$SYSROOT/usr/lib \
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT

make -j$N_CPUS
make install DESTDIR=$SYSROOT
~~~

### Construimos el GCC

Ahora que tenemos Glibc construido con el compilador compatible, podemos generar el nuevo GCC de manera directa:

~~~bash
cd $BUILD_DIR/gcc

$SRC_DIR/gcc-$GCC_VERSION/configure \
--prefix= \
--build=$MACHTYPE \
--host=$MACHTYPE \
--target=$TARGET \
$TARGET_OPTIONS \
$EXTRA_OPTIONS \
--enable-languages=$GCC_LANGUAGES \
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT

make -j$N_CPUS
make install-strip DESTDIR=$INSTALL_DIR
~~~

### Construcción del depurador GDB

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
--with-sysroot=/$TARGET/libc \
--with-build-sysroot=$SYSROOT \
--with-python=/usr/bin/python3

make -j$N_CPUS
make install DESTDIR=$INSTALL_DIR
~~~

Y listo, ahora tenemos el compilador cruzado completo.

### Probamos el Toolchain

Ahora que tenemos el compilador construido, tenemos que testearlo. Para eso genere un hola mundo para pasarlo a la Raspberry Pi y probarlo. Pero antes tenemos que exportar la nueva ubicación del compilador al *PATH*:

~~~bash
export PATH=$INSTALL_DIR/bin:$PATH
~~~

Ya que ahora exportamos el nuevo compilador al *PATH*, podemos invocarlo para compilar la aplicación de prueba:

~~~bash
cd $BUILD_DIR/test

arm-linux-gnueabihf-g++ -Wall main.cpp -o test
~~~

No deberia salir ninguna ninguna advertencia ni tampoco ningun error. Por último pasamos el programa a la Raspberry via scp:

~~~bash
scp test RPI_USER@RPI_IP:~
~~~

Y en la Raspberry Pi ejecutamos:

~~~bash
./test
~~~

Obteniendo el siguiente resultado:

~~~text
Hola mundo!!
~~~

Si llegaste hasta aca felicitaciones, fuiste capaz de crear tu propio GCC cruzado para Raspberry Pi.

### Guardar el Toolchain recien construido

Si queremos guardar el Toolchain mas reciente ejecutamos:

~~~bash
cd $BUILD_DIR

tar -czf $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION

mv $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $SAVE_TOOLCHAIN_DIR
~~~

Ahora guarde los comprimidos en una carpeta separada. Luego ya puede eliminar todo el contenido de la ruta de compilación temporal.

## Conclusiones

Si ambos compiladores funcionaron, entonces perfecto, pudiste compilar tu propio Toolchain cruzado. Como se ve, en realidad esto no es una tarea demasiado complicada, hay que tener una guía sobre que hacer (mas o menos), y después modificar los argumentos pasados a los scripts de compilación para obtener tus herramientas personalizadas.
