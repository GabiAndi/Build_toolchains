# Construir toolchain cruzado para ARM con sistema Linux

**Aclaración:** ***este documento se revisó por ultima vez el 9 de febrero del 2022. Por lo que si esta viendo este artículo pasado un buen tiempo desde la publicación, seguro tenga que complementar la información presente aquí.***

## Tabla de contenidos
- [Construir toolchain cruzado para ARM con sistema Linux](#construir-toolchain-cruzado-para-arm-con-sistema-linux)
  - [Tabla de contenidos](#tabla-de-contenidos)
  - [Introducción](#introducción)
    - [Referencias](#referencias)
    - [Información relevante](#información-relevante)
    - [Experiencia personal](#experiencia-personal)
  - [Pasos previos](#pasos-previos)
    - [Verificamos la versión de paquetes de destino](#verificamos-la-versión-de-paquetes-de-destino)
  - [Construyendo Toolchain inicial](#construyendo-toolchain-inicial)
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
  - [Contruir una versión mas reciente de Toolchain](#contruir-una-versión-mas-reciente-de-toolchain)
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

## Introducción

Este tutorial detalla el proceso que seguí para poder construir GCC para compilación cruzada. Basé mi investigación en la documentación oficial de GNU y en varios artículos de terceros, para poder armar una guía base que ayude a los interezados en el tema.

**¿Qué es un compilador cruzado? ¿Para qué necesito crear el mio?**

Un compilador cruzado es un compilador que genera binarios para sistemas distintos del cual se esta ejecutando. En este caso desde ***x86_64 Linux*** generamos archivos ejecutables para ***armv7 Linux***. Esto permite entre otras cosas, no limitarse a la potencia de computo de la Raspberry Pi, y poder generar los ejecutables directamente en nuestra PC de escritorio.

Toda la guía tiene como objetivo Raspberry Pi 2 en adelante, pero se sigue el mismo proceso para una arquitectura diferente, solo se cambian los parámetros de configuración correspondientes.

Debe tener en cuenta la versión de GCC que ejecuta su máquina. La misma debe ser la mas parecida (o en su defecto mas reciente) a la versión de GCC que se busca compilar. Esto es necesario para la comprobación y reducción de errores del compilador nuevo.

Si desea conocer mas sobre el proceso de compilación de versiones nuevas de GCC a partir de versiones mas antiguas, le recomiendo [que lea sobre el bootstrapping.](https://en.wikipedia.org/wiki/Bootstrapping_(compilers))

**Algunas palabras que utilizaremos**

* **Build:** es el sistema donde se está ejecutando el proceso de construcción.

* **Host:** sistema que ejecutará el GCC una vez que esté construido.

* **Target:** sistema en el que se ejecutarán los binarios producidos por el host.

### Referencias

La documentación oficial no me fue suficiente para poder realizar el proceso de construcción de manera exitosa. Por suerte hay personas que lograron hacerlo y comparten sus experiencias con nosotros (como yo ahora estoy haciendo con ustedes). A continuación pongo a disposición todas las páginas que leí y me ayudaron:

**Documentación oficial:**

* [Sobre el kernel de la Raspberry Pi (variación de Linux).](https://www.raspberrypi.org/documentation/linux/kernel/building.md)
* [Artículo de compilación de GCC osdev.org.](https://wiki.osdev.org/Building_GCC)
* [Artículo de compilación cruzada de GCC osdev.org.](https://wiki.osdev.org/GCC_Cross-Compiler)
* [Documentación oficial de GCC.](https://gcc.gnu.org/install/configure.html)

**Artículos de terceros:**

* [Building GCC as a cross compiler for Raspberry Pi.](https://solarianprogrammer.com/2018/05/06/building-gcc-cross-compiler-raspberry-pi/)
* [How to Build a GCC Cross-Compiler.](https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/)
* [Very Simple Guide for Building Cross Compilers Tips.](http://www.ifp.illinois.edu/~nakazato/tips/xgcc.html)
* [Building GDB and GDBserver for cross debugging.](https://sourceware.org/gdb/wiki/BuildingCrossGDBandGDBserver)
* [Raspberry Pi Toolchains.](https://sourceforge.net/projects/raspberry-pi-cross-compilers/files/)
* [Raspberry Pi 3 have CPU armv7l instead Armv8.](https://www.raspberrypi.org/forums/viewtopic.php?t=140572)

Le recomiendo que **lea a conciencia**. El copiar y pegar comandos como un loco lo llevará a que nada le funcione y se termine por frustrar. Así que tomece el tiempo y la calma necesaria para leer este post, le aseguro que aprendera bastante.

### Información relevante

**Sobre GCC**

GCC es parte del proyecto GNU, y tiene como objetivo mejorar el compilador usado en todos los sistemas GNU, incluyendo la variante GNU/Linux. El desarrollo de GCC usa un entorno de desarrollo abierto y soporta muchas plataformas con el fin de fomentar el uso de un compilador-optimizador de clase global, que pueda atraer muchos equipos de desarrollo, y asegure que GCC y los sistemas GNU funcionen en diferentes arquitecturas y diferentes entornos, y más aún, para extender y mejorar las características de GCC.

**Sobre GDB**

GDB o GNU Debugger es el depurador estándar para el compilador GNU.

Es un depurador portable que se puede utilizar en varias plataformas Unix y funciona para varios lenguajes de programación como C, C++ y Fortran. GDB fue escrito por Richard Stallman en 1986. GDB es software libre distribuido bajo la licencia GPL.

GDB ofrece la posibilidad de trazar y modificar la ejecución de un programa. El usuario puede controlar y alterar los valores de las variables internas del programa.

GDB no contiene su propia interfaz gráfica de usuario y por defecto se controla mediante una interfaz de línea de comandos. Existen diversos front-ends que han sido diseñados para GDB.

**Sobre Binutils**

Las GNU Binary Utilities, o binutils, es una colección de herramientas de programación para la manipulación de código de objeto en varios formatos de archivos objeto. Estas herramientas se usan típicamente en conjunto con el GCC, make y GDB.

Originalmente el paquete consistió solamente en las utilidades menores, pero después el GNU Assembler (GAS) y el GNU Linker (GLD) fueron incluidos en los lanzamientos, puesto que su funcionalidad estaba relacionada estrechamente.

**Sobre Glibc**

La biblioteca de C de GNU, comúnmente conocida como Glibc es la biblioteca de tiempo de ejecución estándar del lenguaje C de GNU. Se distribuye bajo los términos de la licencia GNU LGPL.

En los sistemas en los que se usa, esta biblioteca de C que proporciona y define las llamadas al sistema y otras funciones básicas, es utilizada por casi todos los programas. Es muy usada en los sistemas GNU y sistemas basados en el núcleo Linux. Es muy portable y soporta gran cantidad de plataformas de hardware.

**Sobre Newlib**

Newlib es una implementación de la biblioteca estándar de C destinada a su uso en sistemas embebidos. Es un conglomerado de varias partes de bibliotecas, todas bajo Licencia Open Source que la hacen fácilmente utilizable en productos empotrados.

### Experiencia personal

Crear un conjunto de herramientas de compilación cruzada es jugar un poco a la loteria. Existen multitud de configuraciones y versiones de paquetes disponibles, así que tendrá que encontrar la que le funcione.

Es importante saber que no todas las versiones de Glibc son compatibles con todas las versiones de GCC. Para asegurar un buen grado de compatibilidad, recomiendo que se generé un GCC de aproximadamente la misma fecha de lanzamiento que Glibc, lo mismo con Binutils.

Hablo mucho de Glibc porque es la librería estandar que yo utilice, pero en teoría puede utilizar la que mas le guste.

Otra cosa importante es que conviene tener siempre el último lanzamiento de GCC. Ya que el código generado siempre será de mejor calidad. Ademas se corrijen errores y un monton de otras cosas. Si, es cierto que dije que el GCC debería ser de la misma fecha de lanzamiento que las demas herramientas, pero en este tutorial te voy a mostrar como podemos usar un Glibc *viejo* (ya construido) con un GCC mas nuevo.

## Pasos previos

### Verificamos la versión de paquetes de destino

Al día de hoy, Raspberry Pi OS viene con GCC 10.2.1, GDB 10.1, Binutils 2.35.2 y Glibc 2.31. Es importante que construyamos nuestro compilador cruzado usando la misma versión de Glibc que la Raspberry Pi. Esto nos permitirá integrarnos con el sistema operativo. Sin embargo no necesario que los Binutils, GCC o GDB necesiten ser de la misma versión.
	
Si en un futuro las versiones incorporadas cambian, puede verificarlas con estos comandos (ejecutados desde la Raspberry Pi obviamente):

~~~TEXT
gcc --version

gdb --version

ld -v

ldd --version
~~~

Para mi caso en particular la salida obtenida fue la siguiente:

* Versión de GCC: 10.2.1.
* Versión de GDB: 10.1.
* Versión de Binutils: 2.35.2.
* Versión de Glibc: 2.31.

Como no siempre es posible tener la versión de Glibc compatible con el compilador de GCC que queremos generar, en este tutorial hacemos lo siguiente:

1. Construimos una versión compatible de GCC para crear un Toolchain inicial.
2. Construimos Glibc con la versión de GCC compatible.
3. Construimos el GCC mas reciente incorporado dicho Glibc compatible (precompilado anteriormente).

Si la versión de GCC, Binutils y Glibc son compatibles y se puede crear la cadena de herramientas completa sin problemas, entonces con realizar la sección **Toolchain inicial** usted ya tendrá todo lo que necesita.

## Construyendo Toolchain inicial

### Preparamos el Host

Yo utilizo **Linux Mint** que es un sistema operativo basado en debian, por lo que todo lo haré con el gestor de paquetes apt:

~~~TEXT
sudo apt update && sudo apt upgrade -y

sudo apt install -y build-essential python3 python3-dev python2 python2-dev doxygen git openssl unzip wget libncurses6 libncursesw6 libncurses-dev rsync texinfo texlive autoconf automake gettext gperf autogen guile-3.0 flex patch diffutils libgmp-dev libisl-dev libexpat-dev
~~~

### Preparando la estructura de carpetas

En las siguientes instrucciones, asumiré que estás realizando todos los pasos en una carpeta separada, y que mantenes abierta la misma sesión de terminal hasta que todo esté hecho. En mi caso:

* La ruta de descarga de las fuentes se hará en *$HOME/build-toolchain/src*.
* La ruta de compilación de los binarios se hará en *$HOME/build-toolchain/build*.

Ruta de compilación se refiere a donde se guardaran los binarios compilados del Toolchain. En este tutorial se busca crear un compilador GCC portable. Una vez terminemos, empaquetaremos todo en un archivo *cross-pi-X-gcc-X.X.X.tar.xz* que será trasladable a cualquier sistema (siempre y cuando se cumplan las dependencias propias del Toolchain).

Antes de eso, para poder hacer mas sencilla la escritura de comandos, y así evitar errores de tipeo, voy a exportar las rutas y configuraciones como variables de bash:

~~~TEXT
export N_CPUS="$(nproc)"

export BINUTILS_VERSION="2.35.2"
export GCC_VERSION="8.5.0"
export GLIBC_VERSION="2.31"
export GDB_VERSION="10.1"

export TARGET="arm-linux-gnueabihf"
export TARGET_OPTIONS="--with-arch=armv7-a --with-fpu=neon-vfpv4 --with-float=hard"
export EXTRA_OPTIONS="--disable-multilib --enable-multiarch --enable-lto --disable-nls --with-gnu-as --with-gnu-ld"
export GCC_LANGUAGES="c,c++"

export INSTALL_DIR_PREFIX="cross-pi-2"

export WORK_DIR="$HOME/build-toolchain"
export SRC_DIR="$WORK_DIR/src"
export BUILD_DIR="$WORK_DIR/build"

export INSTALL_DIR="$BUILD_DIR/$INSTALL_DIR_PREFIX-gcc-$GCC_VERSION"
export SYSROOT="$INSTALL_DIR/$TARGET/libc"

export SAVE_TOOLCHAIN_DIR="$HOME"
~~~

Creamos las carpetas base en donde construiremos todo:

~~~TEXT
mkdir -p $SRC_DIR $BUILD_DIR $INSTALL_DIR
~~~

### Obtenemos las fuentes

Descarguemos lo necesario para construir el compilador cruzado. Binutils, Glibc, GCC, GDB y la última versión del kernel de Raspberry.

Para la biblioteca estándar, también puede escoger *Newlib* en lugar de *Glibc*. En mi caso utilizare Glibc ya que la Raspberry corre un sistema GNU/Linux. Newlib se adapta mejor a sistemas embebidos o bare metal:

~~~TEXT
cd $SRC_DIR

wget https://ftpmirror.gnu.org/binutils/binutils-$BINUTILS_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gcc/gcc-$GCC_VERSION/gcc-$GCC_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gnu/glibc/glibc-$GLIBC_VERSION.tar.xz
wget https://ftpmirror.gnu.org/gnu/gdb/gdb-$GDB_VERSION.tar.xz
git clone --depth=1 https://github.com/raspberrypi/linux

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

~~~TEXT
cd $SRC_DIR/gcc-$GCC_VERSION
contrib/download_prerequisites
~~~

### Variables *PATH*

Durante todo el proceso de compilación, asegúrese de que el subdirectorio */bin* de la instalación esté en su *PATH*. Puede eliminar este directorio del *PATH* luego de la instalación, pero la mayoría de los pasos de compilación esperan encontrar arm-linux-gnueabihf-gcc y otras herramientas de host a través del *PATH*.

~~~TEXT
export PATH=$INSTALL_DIR/bin:$PATH
~~~

Preste especial atención a las cosas que se instalan debajo *$INSTALL_DIR/*. Este directorio se considera la raíz del sistema de un sistema de destino imaginario. Un compilador de Linux autohospedado podría, en teoría, usar todos los encabezados y bibliotecas colocados aquí. Obviamente, ninguno de los programas creados para el sistema host, como el propio compilador cruzado, se instalará en este directorio.

### Copiar los encabezados del kernel

Este paso instala los archivos de encabezado del kernel de Linux, lo que finalmente permitirá que los programas creados con nuestra nueva cadena de herramientas realicen llamadas del sistema al kernel en el entorno de destino.

Copie los encabezados del kernel en las carpetas anteriores, consulte la [documentación de Raspbian](https://www.raspberrypi.org/documentation/linux/kernel/building.md) para obtener más información:

~~~TEXT
cd $SRC_DIR/linux

KERNEL=kernel7

make \
ARCH=arm \
INSTALL_HDR_PATH=$SYSROOT/usr \
headers_install

mkdir -p $SYSROOT/usr/lib
~~~

### Construcción de Binutils

Para poder utilizar nuestro compilador cruzado debemos incorporar Binutils a nuestra carpeta resultante por ello es que descargamos las fuentes y procedemos a compilarlo:

~~~TEXT
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

~~~TEXT
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

~~~TEXT
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

~~~TEXT
cd $BUILD_DIR/gcc

make -j$N_CPUS all-target-libgcc
make install-target-libgcc DESTDIR=$INSTALL_DIR
~~~

### Biblioteca C estándar

En este paso, finalizamos el paquete Glibc, que construye la biblioteca C estándar e instala sus archivos en */lib*:

~~~TEXT
cd $BUILD_DIR/glibc

make -j$N_CPUS
make install DESTDIR=$SYSROOT
~~~

### Construimos el GCC completo

Finalmente, terminamos de compilar el paquete GCC de manera completa. Esto nos dará soporte completo a las bibliotecas estándar:

~~~TEXT
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

~~~TEXT
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

~~~TEXT
mkdir -p $BUILD_DIR/test
cd $BUILD_DIR/test

nano main.cpp
~~~

Y añadimos lo siguiente:

~~~TEXT
#include <iostream>

using namespace std;

int main (void)
{
    cout << "Hola mundo!!" << endl;

    return 0;
}
~~~

Debido a que tenemos (si no cerro la terminal desde toda la compilación) la carpeta de nuestro compilador en el PATH, probamos compilar el programa:

~~~TEXT
arm-linux-gnueabihf-g++ -Wall main.cpp -o test
~~~

No deberia salir ninguna ninguna advertencia ni tampoco ningun error. Por último pasamos el programa a la Raspberry via scp:

~~~TEXT
scp test RPI_USER@RPI_IP:~
~~~

En la Raspberry Pi ejecutamos:

~~~TEXT
./test
~~~

Obteniendo el siguiente resultado:

~~~TEXT
Hola mundo!!
~~~

Si llegaste hasta aca felicitaciones, fuiste capaz de crear tu propio GCC cruzado para Raspberry Pi.

### Guardar el Toolchain recien construido

Si deseamos podemos comprimir y guardar en caso que querramos distribuirlos o hacer una copia de seguridad, para ello vaya a la carpeta donde instalo los compiladores:

~~~TEXT
cd $BUILD_DIR

tar -czf $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION

mv $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $SAVE_TOOLCHAIN_DIR
~~~

## Contruir una versión mas reciente de Toolchain

Lo que vamos a hacer a continuación es crear un GCC mas moderno con librerías estándar mas antiguas. Por lo que utilizaremos el compilador creado anteriormente (que es compatible con ese Glibc) para construir la Glibc antigua y luego construir nuestro nuevo GCC con la biblioteca estándar.

Tratar de construir Glibc antigua con un GCC moderno causará errores raros que no podrá solucionar, por eso preferí utilizar este enfoque que nos evitará problemas.

En este caso voy a compilar la última versión de GCC a partir de la construcción de la Glibc con la versión compatible.

### Preparando la estructura de carpetas

Tenemos que ir a la carpeta de fuentes y limpiar un poco la instalación anterior:

~~~TEXT
cd $SRC_DIR

rm -rf binutils-$BINUTILS_VERSION gcc-$GCC_VERSION glibc-$GLIBC_VERSION gdb-$GDB_VERSION linux

cd $BUILD_DIR

rm -rf binutils gcc glibc gdb
~~~

Vamos a actualizar las variables de bash para la nueva carpeta:

~~~TEXT
export GCC_VERSION="10.2.0"

export INSTALL_DIR="$BUILD_DIR/$INSTALL_DIR_PREFIX-gcc-$GCC_VERSION"
export SYSROOT="$INSTALL_DIR/$TARGET/libc"
~~~

Por ultimo generamos la nueva carpeta de instalación:

~~~TEXT
mkdir -p $INSTALL_DIR
~~~

### Obtenemos las fuentes

Descarguemos lo necesario para construir el compilador cruzado. GCC y la última versión del kernel de Raspberry Pi OS:

~~~TEXT
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

~~~TEXT
cd $SRC_DIR/gcc-$GCC_VERSION
contrib/download_prerequisites
~~~

### Copiar los encabezados del kernel

Este paso instala los archivos de encabezado del kernel de Linux, lo que finalmente permitirá que los programas creados con nuestra nueva cadena de herramientas realicen llamadas del sistema al kernel en el entorno de destino.

Copie los encabezados del kernel en las carpetas anteriores, consulte la [documentación de Raspbian](https://www.raspberrypi.org/documentation/linux/kernel/building.md) para obtener más información:

~~~TEXT
cd $SRC_DIR/linux

KERNEL=kernel7

make \
ARCH=arm \
INSTALL_HDR_PATH=$SYSROOT/usr \
headers_install

mkdir -p $SYSROOT/usr/lib
~~~

### Construcción de Binutils

Para poder utilizar nuestro compilador cruzado debemos incorporar Binutils a nuestra carpeta resultante por ello es que descargamos las fuentes y procedemos a compilarlo.

A continuación, construimos Binutils para nuestro compilador GCC:

~~~TEXT
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

~~~TEXT
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

~~~TEXT
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

~~~TEXT
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

~~~TEXT
export PATH=$INSTALL_DIR/bin:$PATH
~~~

Ya que ahora exportamos el nuevo compilador al *PATH*, podemos invocarlo para compilar la aplicación de prueba:

~~~TEXT
cd $BUILD_DIR/test

arm-linux-gnueabihf-g++ -Wall main.cpp -o test
~~~

No deberia salir ninguna ninguna advertencia ni tampoco ningun error. Por último pasamos el programa a la Raspberry via scp:

~~~TEXT
scp test RPI_USER@RPI_IP:~
~~~

Y en la Raspberry Pi ejecutamos:

~~~TEXT
./test
~~~

Obteniendo el siguiente resultado:

~~~TEXT
Hola mundo!!
~~~

Si llegaste hasta aca felicitaciones, fuiste capaz de crear tu propio GCC cruzado para Raspberry Pi.

### Guardar el Toolchain recien construido

Si queremos guardar el Toolchain mas reciente ejecutamos:

~~~TEXT
cd $BUILD_DIR

tar -czf $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION

mv $INSTALL_DIR_PREFIX-gcc-$GCC_VERSION.tar.xz $SAVE_TOOLCHAIN_DIR
~~~

Ahora guarde los comprimidos en una carpeta separada. Luego ya puede eliminar todo el contenido de la ruta de compilación temporal.

**Nota:** se mostro el proceso para generar un compilador nuevo a partir de las Glibc anteriores, pero en realidad es conveniente utilizar la misma versión de GCC que la que incorpora la Raspberry.

## Conclusiones

Si ambos compiladores funcionaron, entonces perfecto, pudiste compilar tu propio Toolchain cruzado. Como se ve, en realidad esto no es una tarea demasiado complicada, hay que tener una guía sobre que hacer (mas o menos), y después modificar los argumentos pasados a los scripts de compilación para obtener tus herramientas personalizadas.
